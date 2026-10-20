# AutoDNS

[![Docker](https://github.com/baker-scripts/autodns/actions/workflows/docker-image.yml/badge.svg)](https://github.com/baker-scripts/autodns/actions/workflows/docker-image.yml)
[![Release](https://img.shields.io/github/v/release/baker-scripts/autodns)](https://github.com/baker-scripts/autodns/releases)

Small Flask service that updates a Cloudflare A record to the caller's IP. A client hits `/update-dns?guid=<GUID>`, AutoDNS looks up the record mapped to that GUID, and sets its content to the client IP. Optional Apprise notifications.

## Quick Start

Requires Docker and a Cloudflare API token with DNS edit permission on the zone. The A record must already exist; AutoDNS updates records but does not create them.

```shell
docker run -d \
  --name autodns \
  --restart unless-stopped \
  -p 4295:4295 \
  -v /your/path/to/config:/config \
  -e CF_ZONE_ID="your-zone-id" \
  -e CF_API_TOKEN="your-api-token" \
  -e AUTODNS_PORT=4295 \
  ghcr.io/baker-scripts/autodns:develop
```

[`docker-compose.example.yml`](docker-compose.example.yml) has the same setup for Compose.

Create a GUID for a record (the full record name, for example `home.example.com`):

```shell
docker exec autodns python autodns.py generate home.example.com
```

The GUID is the SHA-256 hash of the name, so anyone who knows the name can compute it. Treat the GUID as an identifier, not a secret, and keep the endpoint behind access controls.

Mappings are stored in `/config/guid_mapping.json`, so mount `/config` to keep them.

## Configuration

All settings are environment variables.

| Variable | Default | Description |
| --- | --- | --- |
| `CF_ZONE_ID` | none | Cloudflare zone ID that holds the records. |
| `CF_API_TOKEN` | none | Cloudflare API token. |
| `AUTODNS_PORT` | `5000` | Port the server listens on. The Docker image exposes `4295`, so set `AUTODNS_PORT=4295` to match. |
| `AUTODNS_HOST` | `0.0.0.0` | Address the server binds to. |
| `ENABLE_NOTIFICATIONS` | `false` | Set to `true`, `1`, or `t` to send notifications. |
| `APPRISE_URLS` | empty | Comma-separated [Apprise](https://github.com/caronc/apprise) URLs. Used only when notifications are enabled. |
| `TRUSTED_PROXIES` | empty | Comma-separated proxy IPs. `X-Forwarded-For` is trusted only when the direct connection comes from one of these. When empty, the connection's remote address is always used. |

## Usage

```shell
curl "http://localhost:4295/update-dns?guid=<GUID>"
```

The record is set to the client IP. Run it from cron or a scheduler on the host whose IP you want to publish.

| Status | Meaning |
| --- | --- |
| 200 | Record updated. |
| 400 | `guid` parameter missing. |
| 404 | Record does not exist in Cloudflare, or the Cloudflare API returned an error. |
| 429 | Unknown GUID, or the record was updated in the last 10 minutes. |
| 500 | Cloudflare rejected the update. |

CLI commands:

| Command | Description |
| --- | --- |
| `python autodns.py` | Start the server. |
| `python autodns.py generate <name>` | Create a GUID for a record name and save it to the mapping file. |

For local development, run `pip install -r requirements.txt` and then `python autodns.py`. The mapping file path is fixed at `/config/guid_mapping.json`.

## Contributing

Issues and pull requests welcome. See the [contributing guidelines](https://github.com/baker-scripts/.github/blob/main/CONTRIBUTING.md). The default branch is `develop`.

## Disclaimer

This software is provided as-is with no warranty. Test in a non-production zone first. The authors are not responsible for DNS issues or service disruptions.

## License

[GPL-3.0](LICENSE)
