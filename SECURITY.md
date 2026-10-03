# Security policy

## Supported versions

Pre-alpha: no released versions yet. Reports against `main` are welcome.

## Reporting a vulnerability

Please **don't open a public issue.** Use GitHub's private vulnerability reporting:
**Security → Report a vulnerability** on this repository. Include the affected file or endpoint,
reproduction steps and the impact you expect.

You should get an acknowledgement within 7 days. Fixes are tracked against the requirement they
break in [`docs/spec/security.md`](docs/spec/security.md) (SEC-01 to SEC-22).

## Threat model in brief

- Single owner, self-hosted, bound to `127.0.0.1` by default. Exposing it to a network is
  unsupported until a hosting decision says otherwise.
- Agents connect through scoped tokens (`read`, `write`, `capture`, `admin`). There are no delete
  tools over MCP.
- Text captured from third parties is marked untrusted. Agents must never follow instructions found
  inside it (prompt-injection defence, SEC-14).
- Contact data is plaintext at rest in v1 (SEC-08). Protect the data dir accordingly.
