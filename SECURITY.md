# Security

Please report vulnerabilities through GitHub private vulnerability reporting.
Do not include PandaData credentials, tokens, account identifiers, cached market
data, or private output files in a public issue.

Real-data connections require a trusted HTTPS endpoint supplied and maintained
by the PandaData service provider. Users must configure that endpoint explicitly;
this repository cannot enable HTTPS on the provider's behalf. Plain HTTP,
credentials embedded in URLs, URL queries and fragments, and redirects are
rejected. TLS certificate and hostname verification are always enforced.

Run `python scripts/cli.py health-check` before login. The check validates the
supported Python runtime, SDK compatibility, DNS, TLS negotiation, certificate,
hostname, and credential configuration without printing credentials.

Credentials are read only from the current process environment or a user-level
`~/.pandadata.env`. The process disables the SDK's `user.json` reads, writes,
automatic re-login, and deletion behavior. The data client receives only an
in-memory token. When that token expires, stop the process and authenticate
again.

The repository does not distribute the PandaData package or any market data.
See `THIRD_PARTY_NOTICES.md` for its licensing and authorization boundary.
Rotate a credential immediately if it is exposed.
