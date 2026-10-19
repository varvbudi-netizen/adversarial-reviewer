# Attack Catalog

The adversarial question bank. For each axis, the goal is to construct **one concrete
counterexample**: a specific input or state that produces a wrong result. Read the axis,
try to build that counterexample against the real code, and either confirm a finding or
move on. Skip axes that genuinely do not apply.

Ordering is by yield: the top axes catch the most real defects. You do not need all of them
on every review — pick the ones the change actually touches.

---

## 1. Fail-open vs fail-closed (highest yield for security/safety code)

The question: **when something goes wrong, does the code default to "allow/proceed" or to
"block/abort"?** Secure and safe code fails closed.

- What happens on an error return, a thrown exception, a timeout, a `nil`/`null`, an empty collection, a missing config key? Trace each to its end state.
- Is there a `catch`/`recover`/`except` that swallows the error and continues as if it succeeded?
- Is there a "default" branch that permits the action when the input didn't match any known-good case? (Default-allow is the classic fail-open bug.)
- If a validation, auth check, or kill-switch install *fails*, does the surrounding flow stop, or does it log a warning and keep going?
- Empty-input degenerate case: does "no rules" mean "block everything" (closed) or "allow everything" (open)?

Counterexample shape: *"Config is empty / the probe returns no IP / the guard install exits 1 → the code proceeds and traffic/data flows unprotected."*

## 2. Boundary & degenerate input

Enumerate the edges and feed each one in:

- Empty: `""`, `[]`, `{}`, empty file, zero rows.
- Zero / one / two / N / N+1 / max / max+1 / negative / off-by-one at the boundary.
- Duplicate keys, duplicate list entries, repeated calls.
- Missing / null / absent field vs present-but-empty field (they are different — check both).
- Wrong-but-plausible type: hostname where an IP is expected, IPv6 where IPv4 is assumed, string where number is assumed, relative where absolute path is assumed.
- Unicode, whitespace-only, leading `-` (looks like a flag), very long input.
- Out-of-order / unsorted input where order is assumed.

Counterexample shape: *"With exactly one exit node / a hostname instead of an IP / a duplicate name, the function returns X but should return Y (or should reject)."*

## 3. Silent failure & inappropriate fallback

The code "works" but hides a problem:

- An error is caught and turned into a default/empty value that a caller can't distinguish from a real result.
- A fallback path (retry, secondary source, cached value) is taken quietly when the primary failed — the caller thinks it got fresh/authoritative data.
- A partial result is returned as if complete (loop broke early, a batch item failed, a page was truncated).
- A "best effort" write that can fail without anyone noticing.
- Logging that says "success" before verifying success.

Counterexample shape: *"Upstream returns a 500 → we return the stale cached value with no error → caller acts on stale data believing it is current."*

## 4. State, persistence & idempotency

For anything that reads/writes state, a file, a DB, or shared memory:

- **Aliasing / shared mutable state**: does a returned slice/map/object share backing storage with an internal one, so a caller can mutate your invariants? Does a "copy" actually deep-copy?
- **Persisting the derived**: is something that should be *computed on read* being *written to disk*, so it goes stale or drifts? (Synthesized/defaulted values usually should not be persisted.)
- **Idempotency**: run the operation twice — does the second run duplicate, corrupt, degrade, or re-trigger side effects? Onboarding the same node, applying the same migration, retrying the same request.
- **Atomicity**: if the process dies mid-write, is the file/DB left half-written? Is there a temp-file+rename or a transaction?
- **Round-trip**: does `save` then `load` reproduce the exact state? Does a field silently drop, reorder, or change type?
- **Leak on serialize**: does dumping "the whole object" to a client/log include a secret, private key, password, or internal address that should have been redacted?

Counterexample shape: *"bootstrap → SaveState → LoadState → bootstrap again persists the synthesized egress policy the second time, so a later edit to the source no longer takes effect."*

## 5. Concurrency & ordering

Whenever two things can run at once or in an unexpected order:

- Two callers hit the same function — shared counter, cache, connection, file. Interleave them.
- Check-then-act: `if !exists { create }`, `if free { take }` — can two racers both pass the check?
- A value read at the top of a function that another goroutine/thread can change before it is used.
- Ordering assumptions: does the code assume callbacks/events/messages arrive in order? What if they don't?
- Cancellation / timeout mid-operation — what state is left behind?

Counterexample shape: *"Two concurrent onboards of different nodes both read the same map, both write, one write is lost."*

## 6. Injection & untrusted data

Any value that crosses a trust boundary (network, file, user, another service) and lands in an interpreter:

- Interpolated into a shell command, SQL, HTML, a regex, a file path, a format string, a URL?
- Is the "it's safe because I quoted it" claim actually true for *all* inputs, or only the happy ones? (A value that's a safe bareword today may not be after a refactor.)
- Path traversal: `../`, absolute paths, symlinks.
- A field the attacker controls used as a flag/option (`-` prefix), a key, or a redirect target.
- Deserialization of attacker-controlled data.

Counterexample shape: *"The node alias flows into `ssh <alias>` unquoted; an alias of `-oProxyCommand=...` executes an arbitrary command."*

## 7. Resource, arithmetic & external-call correctness

- Unbounded growth: a list/map/log/retry-loop with no cap; a `while` that may not terminate.
- Leaked handles: files, sockets, locks, goroutines not released on the error path.
- Integer overflow / float rounding / division by zero / precision loss on money.
- Timeouts missing on network calls; no backoff; a retry storm.
- Assuming an external call succeeds, returns non-empty, returns the shape you expect.

Counterexample shape: *"A node whose probe never returns hangs the whole bootstrap because there is no per-probe timeout."*

## 8. Test quality (review the tests, not just the code)

The tests are the safety net — attack the net:

- Does each test assert the **actual value**, or merely that no error was thrown / that something is non-empty?
- Could the test pass with the logic deleted or inverted? (Mentally break the code — does a test go red?)
- Are the fixtures realistic, or do they dodge the hard case (only happy path, only one item, only valid input)?
- Is there a test for the fail-closed path, the boundary, the error, the idempotency?
- Does a "0 failures" actually run the code, or did a filter/skip/early-return make it a no-op? (Beware harness bugs where the pass is vacuous.)
- Golden/snapshot tests: is the snapshot pinned, and does anyone verify it is *correct* rather than just *stable*?

Counterexample shape: *"The test checks that the client profile is generated, but never asserts the private key is absent — so the leak passes."*

## 9. Regression against preserved behavior

When a change claims "existing behavior is unchanged":

- Find the existing behavior's own tests (golden tests) and confirm they still pass and still assert the old values.
- Construct the original real-world case and run it through the new code — same result?
- Look for hardcoded assumptions the refactor removed: a region bonus, a special-cased name, a magic constant that other code silently relied on.

Counterexample shape: *"The generic refactor dropped the `region == 'ru'` gate; the original ru1→lv route now scores differently and a different exit is selected."*

---

## Using the catalog

1. Pick the axes the change actually touches; there is no required count. A pure-arithmetic function does not need the concurrency axis; a kill-switch needs fail-open first.
2. For each, spend the effort to build a *concrete* counterexample, not a vague worry.
3. Verify (run it or trace it). Confirmed or plausible — label it.
4. Move findings to `report-format.md`'s template, most-severe first.
5. Note the axes you checked and cleared — that is signal too.
