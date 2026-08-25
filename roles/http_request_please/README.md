# http_request_please

Declarative HTTP sanity checks — define a suite of requests in a JSON or
YAML file (or inline vars) and assert status codes, body content, and
certificate SANs against **Linux and Windows** targets.

This is the only role in the `mcowser_p.http_request_please` collection.
See the [collection README](../../README.md) for the full guide, including
the reusable GitHub Action; this page is the role reference.

## Requirements

- Ansible `>= 2.16`
- Collections: `ansible.windows`, `community.crypto` (installed
  automatically with the collection)

`cert_check` runs from the controller via `community.crypto`, so it
validates the certificate the controller sees.

## Role Variables

### Defining the suite

| Variable | Default | Description |
| --- | --- | --- |
| `http_request_please_file` | `""` | Path to a JSON or YAML file. Its top level must be a list of request items, or a dict with a `requests` list. |
| `http_request_please_requests` | `[]` | Inline list of request items. Concatenated with the file's items, not replaced by them. |
| `http_request_please_fail_fast` | `false` | Stop at the first failing check instead of running everything and reporting all failures at the end. |

### Per-request defaults

Every one of these can be overridden on an individual request item.

| Variable | Default | Description |
| --- | --- | --- |
| `http_request_please_scheme` | `https` | `http` or `https`. |
| `http_request_please_port` | `""` | Empty uses the scheme default (80/443). |
| `http_request_please_method` | `GET` | |
| `http_request_please_status_code` | `[200]` | Acceptable status codes. |
| `http_request_please_validate_certs` | `true` | |
| `http_request_please_timeout` | `10` | Seconds. |
| `http_request_please_retries` | `1` | |
| `http_request_please_delay` | `3` | Seconds between retries. |

### Request item schema

All keys are optional except a target — either `url` **or** `host`.

| Key | Description |
| --- | --- |
| `name` | Label used in reporting. Defaults to the url. |
| `url` | Full URL. Mutually exclusive with `host`/`port`/`scheme`/`path`. |
| `host` | IP address or DNS name. |
| `sni` | Hostname to present when `host` is an IP. Sent as the `Host:` header and, with `cert_check`, validated against the certificate's SAN list. |
| `port` | Integer port. |
| `scheme` | `http` or `https`. |
| `path` | Request path. Defaults to `/`. |
| `method` | `GET`, `POST`, `PUT`, `DELETE`, … |
| `headers` | Dict of extra headers. |
| `body` | Request body. Use `body_format: json` for dicts. |
| `body_format` | `raw`, `json`, or `form-urlencoded`. |
| `status_code` | List of acceptable status codes. |
| `contains` | Regex the response body must match. |
| `cert_check` | `true` fetches the certificate from the controller with SNI and asserts `sni` (or `host`) appears in its SANs. |
| `validate_certs`, `timeout`, `retries`, `delay` | Per-item overrides. |

**On `sni` and certificate validation:** when `sni` is set and
`validate_certs` is not explicitly set, cert validation is disabled for the
request itself — that is routing mode, where you are deliberately hitting an
IP while presenting a hostname. Use `cert_check: true` to validate the
certificate honestly instead.

## Dependencies

None.

## Example Playbook

```yaml
- hosts: web_servers        # Linux or Windows — same vars
  roles:
    - role: mcowser_p.http_request_please.http_request_please
      vars:
        http_request_please_file: files/checks.yml
```

Inline requests, without a file:

```yaml
- hosts: localhost
  roles:
    - role: mcowser_p.http_request_please.http_request_please
      vars:
        http_request_please_fail_fast: true
        http_request_please_requests:
          - name: api-health
            url: https://api.example.com/health
            contains: '"status":"ok"'

          - name: by-ip-with-sni
            host: 10.0.0.5
            sni: api.internal.example.com
            port: 8443
            path: /health
            cert_check: true

          - name: plain-dns-custom-port
            host: db-admin.internal
            scheme: http
            port: 8081
            status_code: [200, 302]
            contains: login
```

## License

Apache-2.0
