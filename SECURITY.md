# Security

Please report vulnerabilities through GitHub private vulnerability reporting.
Do not include PandaData credentials, tokens, account identifiers, cached market
data, or private output files in a public issue.

Real-data connections require an explicitly configured trusted HTTPS endpoint.
Plain HTTP and redirects are rejected. TLS certificate verification is forced
on even if the third-party SDK environment is configured otherwise.

Credentials are read only from the current process environment or a user-level
`~/.pandadata.env`. The process disables the SDK's `user.json` reads, writes,
automatic re-login, and deletion behavior. The data client receives only an
in-memory token. When that token expires, stop the process and authenticate
again.

The repository does not distribute the PandaData package or any market data.
See `THIRD_PARTY_NOTICES.md` for its licensing and authorization boundary.
Rotate a credential immediately if it is exposed.
