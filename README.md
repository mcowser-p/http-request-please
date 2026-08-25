# http-request-please 🙏

**HTTP Request, Please** — declarative HTTP sanity checks. Define a suite
of requests in a JSON or YAML file (or inline vars), and run it three
ways: as an Ansible role against **Linux and Windows** hosts, from any
inventory — or as a **reusable GitHub Action** on a runner.

```bash
ansible-galaxy collection install mcowser_p.http_request_please
```

## As a GitHub Action

```yaml
- uses: mcowser-p/http-request-please@v1
  with:
    requests_file: .github/http-checks.yml
```

or a one-liner check:

```yaml
- uses: mcowser-p/http-request-please@v1
  with:
    url: https://api.example.com/health
    status_code: "200"
    contains: '"status":"ok"'
```

## As an Ansible role

```yaml
- hosts: web_servers        # Linux or Windows — same vars
  roles:
    - role: mcowser_p.http_request_please.http_request_please
      vars:
        http_request_please_file: files/checks.yml
```

## The request suite

```yaml
# checks.yml — a list (or {"requests": [...]}) in YAML or JSON
- name: api-health
  url: https://api.example.com/health

- name: by-ip-with-sni            # hit an IP, present a hostname
  host: 10.0.0.5
  sni: api.internal.example.com   # sent as Host:, validated against the cert SAN
  port: 8443
  path: /health
  cert_check: true                # real SAN validation, from the controller

- name: plain-dns-custom-port
  host: db-admin.internal
  scheme: http
  port: 8081
  status_code: [200, 302]
  contains: login
```

Every field has a role-level default (`http_request_please_scheme`,
`_port`, `_method`, `_status_code`, `_validate_certs`, `_timeout`,
`_retries`, `_delay`) overridable per item. `fail_fast: true` stops at
the first failure; otherwise every check runs and the role fails at the
end with a table of everything that broke.

### IP + SNI/SAN targets

Two independent layers, per request:

- **Routing** — with `sni:` set, the request goes to the IP with a
  `Host:` header, and request-level cert validation defaults off (an IP
  URL can never match a hostname cert).
- **Certificate** — `cert_check: true` fetches the certificate *from the
  controller* with real SNI (`community.crypto.get_certificate`) and
  asserts the name appears in the SAN list. Works identically for
  Windows targets, because the TLS probe never runs on the target.

## Windows

Same vars, same files — the role switches to `ansible.windows.win_uri`
on Windows targets, over **WinRM or OpenSSH** connections (both are
exercised in CI).

## Releasing

Conventional commits on `main` drive semantic-release: version stamped
into `galaxy.yml`, tarball attached to the GitHub release, published to
Ansible Galaxy when the `GALAXY_API_KEY` secret is set (skipped with a
warning otherwise).

## Contributing

Commit messages drive releases here — semantic-release reads them to pick
the next version, and a commit **without a scope is ignored entirely**, so
`feat: thing` silently costs you a release where `feat(role): thing` cuts
one. CI checks every commit on a Linux runner, so the verdict is identical
whatever you develop on.

To get the same check locally before the commit lands — on Windows, macOS
or Linux alike:

```bash
npm install
```

That installs the `commit-msg` hook. It runs through Git for Windows' own
bundled `sh`, so there is no separate Windows path to keep working. To
check a message by hand:

```bash
npx commitlint --edit .git/COMMIT_EDITMSG
```

Dependabot PRs are approved and merged automatically once the required
checks pass, and never merged when a check fails. Major version bumps are
the exception: they are assigned to the repository owner for review instead
of being merged, since those are the ones that actually break things.

## License

Apache-2.0
