# VeoveoAI Fork

This fork carries Veoveo changes on `veoveo/main` over upstream rmcp
(`upstream` = `https://github.com/modelcontextprotocol/rust-sdk`, branch `main`). Our
forks stay ours: nothing here is offered upstream.

- Current upstream base: `fe45871` (2026-11-01), after `rmcp-v3.8.0`, with the Skills
  extension (SEP-2640, #1392). The crates' version is `3.8.0`, which consumers pin
  beside the revision of `veoveo/main` they use.
- Previous fork line: `main` up to `faa4e9d`, upstream `rmcp-v3.5.0` with our changes,
  carried through merges of upstream releases. `veoveo/main` replays those changes as
  one commit each on upstream `main`, and `main` is no longer updated.
- Each change is the smallest one that closes a gap a Veoveo consumer has. Update this
  file whenever a fork change is added, changed or retired.

## Updating To Latest Upstream

1. Fetch `upstream` and replay the commits under "Carried" onto `upstream/main` in a new
   branch, one commit each, never a merge.
2. For each, first check whether upstream now does the same. If it does, drop the commit
   and record that under "Retired"; if upstream does part of it, keep only the rest.
3. Resolve conflicts by intent: what each item below says it does.
4. Run rmcp's CI checks, as `.github/workflows/ci.yml` runs them:
   - `cargo +nightly fmt --all -- --check`
   - `cargo clippy --all-targets --all-features -- -D warnings`, the same with every
     feature but `local`, and `cargo hack clippy --package rmcp --lib --each-feature
     --exclude-features local -- -D warnings`
   - `cargo test --all-features`, `cargo test -p rmcp --lib --no-default-features` and
     `cargo test -p rmcp` with every feature but `local`
5. Update the upstream base above, push the branch as `veoveo/main`, and move the
   consumers' pins.

## Carried

### Skills manifest boundaries and capability negotiation (upstream #1412)

- `crates/rmcp/src/model/skill.rs`, `crates/rmcp/src/service/client.rs`: a skill's
  manifest may name only files inside the skill's own directory (dot segments, encoded
  dots, another root and a directory leaf are refused), a `SKILL.md` in a query or
  fragment names no skill, and the client sends `skills/list` and `skills/get` only to
  a server that declared the skills extension, answering `-32601` otherwise.
- Why: upstream pull request #1412 by thomas (@tsarlandie-oai), still open. The Veoveo
  Hub forwards server skills through one MCP surface and must not pass on a manifest
  that reaches outside its skill.
- Check: `crates/rmcp/tests/test_skills.rs` and the unit tests in `model/skill.rs`.
- Drop when upstream merges #1412.

### Task status notifications through `subscriptions/listen`

- `crates/rmcp/src/model.rs`: `SubscriptionFilter` carries `task_ids`, with builder
  methods `task_ids` and `task_id`; intersection, subset and reduction to the server's
  capabilities cover them, and task IDs are kept only when the server supports tasks.
- `crates/rmcp/src/handler/server.rs`: a listen that asks for task IDs fails with
  missing-required-client-capability when the client did not declare the tasks
  extension.
- `crates/rmcp/src/service/server.rs`: `SubscriptionSink` sends `notifications/tasks`
  for an accepted task ID (`notify_task_status`, the complete `DetailedTask`) and refuses
  any other; `crates/rmcp/src/service/client.rs`: a `Subscription` delivers them.
- The message schemas under `crates/rmcp/tests/test_message_schema/` and the README's
  subscription section describe it.
- Why: the Veoveo Hub and its servers report Task status to subscribed callers instead
  of polling `tasks/get`. Upstream does not route SEP-2663 task status notifications
  through `subscriptions/listen` yet.
- Check: `test_subscriptions.rs`, `test_subscriptions_model.rs`,
  `test_subscriptions_streamable_http.rs` and `test_message_schema.rs`.
- Drop when upstream's `SubscriptionFilter` carries task IDs.

### Typed HTTP rejections

- `crates/rmcp/src/transport/streamable_http_client.rs`: `StreamableHttpError` gains
  `HttpResponse { status, body }`.
- `crates/rmcp/src/transport/common/reqwest/streamable_http_client.rs`: a non-success
  response that is no JSON-RPC error and that upstream's `HttpStatus` (429 and 5xx,
  #1386) does not cover ends a POST with `HttpResponse` instead of
  `UnexpectedServerResponse("HTTP {status}: {body}")`: 400, 413 or 422, a redirect the
  client does not follow, and a 401 or 403 without a challenge, whose JSON-RPC body
  upstream deliberately does not forward.
- Why: the Veoveo Hub forwards an upstream server's rejection to its caller with the
  upstream's own status, such as 413 for a body over the server's limit, which text
  cannot carry. Since #1386, a consumer reads 429 and 5xx from `HttpStatus`, without a
  body, and every other status from `HttpResponse`.
- Check: `test_streamable_http_4xx_error_body.rs`, which runs only without the `local`
  feature, and the redirect tests in the reqwest client's unit tests. Upstream's tests
  of a redirect not followed and of a 401 or 403 without a challenge expect the typed
  status here.
- Drop when upstream exposes the status of every HTTP rejection.

### A response the closed peer never received logs at DEBUG

- `crates/rmcp/src/service.rs`: a handler's response that finishes during the shutdown
  drain after the peer closed its transport is logged at DEBUG, and the drain ends,
  instead of ERROR "failed to send pending response during drain".
- Why: the Veoveo Hub ends a stateless MCP operation for every HTTP request, and each
  closed client logged an ERROR that alerts on nothing.
- Check: by reading; the race has no deterministic test.
- Drop when upstream classifies this send as shutdown noise.

### A startup failure its caller receives logs at DEBUG

- `crates/rmcp/src/transport/streamable_http_client.rs`: when a streamable HTTP client
  worker's first request fails and its caller received the error, the worker ends as
  `TransportClosed` after one DEBUG line; when no caller received it, the quit stays
  fatal and ERROR.
- Why: the Veoveo Hub reconnects to a registered server that is replaced while it runs,
  and each reconnect logged ERROR "worker quit with fatal" for a failure the Hub
  handles.
- Check: the log-level unit tests at the end of `streamable_http_client.rs`.
- Drop when upstream stops logging a startup failure its caller received as fatal.

## Retired

- The merges of upstream releases on `main` (3.1.4, 3.4.0, 3.5.0): `veoveo/main` replays
  the changes on upstream `main` instead. The conflict resolutions those merges held for
  task subscriptions and typed HTTP rejections are part of the carried commits.
- Typed HTTP rejections for 429 and 5xx: upstream's body-free `HttpStatus` (#1386) covers
  them, so `HttpResponse` covers only the statuses it does not.
