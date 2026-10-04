# audit-toolbox

A containerized bundle of the Linux/network audit tools that inspect
something *external* to themselves, so they containerize cleanly:
ssh-audit, Trivy, and testssl.sh. Output is normalized to JSON-lines for
easy ingestion by a log shipper (Filebeat, Graylog Sidecar, etc.).

## Why these three (and not others)

| Tool | Containerizes? | Why |
|---|---|---|
| ssh-audit | Yes | Pure network client — audits a remote SSH service |
| Trivy | Yes | Scans images/filesystems, not the host it runs on |
| testssl.sh | Yes | Pure network client — grades a remote TLS endpoint |
| Lynis | No | Audits its *own* host — in a container it would score the container, not your host. Run the host binary. |
| auditd | No | Kernel host daemon — cannot run meaningfully in a container |
| Docker Bench | N/A | Already ships as its own container (`docker/docker-bench-security`) |

## Usage

```bash
cp .env.example .env        # fill in your real targets
docker compose build

# Run a single tool ad hoc
docker compose run --rm audit-toolbox ssh-audit -l warn host.example.com
docker compose run --rm audit-toolbox trivy image nginx:1.27
docker compose run --rm audit-toolbox testssl.sh https://example.com

# Run everything in .env and write one normalized JSON-lines file
docker compose run --rm audit-toolbox audit-collect
```

## Shipping to a log pipeline

Point your shipper at `./out/*.jsonl`. Ship raw lines (don't pre-parse
JSON in the shipper config) so your backend's own JSON-parsing pipeline
rule fires and can apply a consistent field prefix — this keeps audit
events consistent with however your other log sources are already tagged.

## License

MIT — see LICENSE.
