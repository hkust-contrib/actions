# Authenticate GitHub Action

This GitHub Action exchanges the runner's GitHub OIDC ID token for a workload identity federation token via [RFC 8693 token exchange](https://www.rfc-editor.org/rfc/rfc8693) against [Wayport](https://login.ust.dev), the multi-tenant identity provider at `https://login.ust.dev`.

Wayport namespaces every endpoint under an organization (`POST /orgs/{orgId}/token`), so each organization supplies its own ID.

Before using the action, a Wayport admin must configure, for your organization:

1. A provider with issuer `https://token.actions.githubusercontent.com` and workload `audiences` containing the requested GitHub token audience (the `id_token_audience` input, or the runner's default audience when omitted)
2. A workload identity in the organization, with its allowed audiences, allowed scopes, and token TTL
3. A federated credential on that identity whose conditions match your repository's GitHub `sub` claim (e.g. `repo:hkust-contrib/example:ref:refs/heads/main`)

```yml
permissions:
  id-token: write

steps:
- name: Authenticate runner with Wayport
  id: authenticate
  uses: hkust-contrib/actions/authenticate@latest
  with:
    org: 00000000-1111-2222-3333-444444444444
    audience: https://example.internal
    scope: terraform
```

### Action Inputs

- `org` **(required)** ID of the organization on Wayport that owns the workload identity. Each organization supplies its own
- `endpoint` base URL of the identity provider, defaults to `https://login.ust.dev`
- `id_token_audience` audience of the GitHub OIDC ID token, must match one of the provider's configured workload audiences. Defaults to the runner's default audience
- `scope` space-separated scopes to request, must be a subset of the workload identity's allowed scopes. Defaults to all allowed scopes
- `audience` audience of the issued access token. Defaults to the workload identity's single allowed audience; required when the identity allows multiple audiences

### Action Outputs

- `access_token` the issued access token (`at+jwt`, RS256, signed with the organization's signing key), masked in logs
- `expires_in` lifetime of the token in seconds, as configured on the workload identity
- `scope` scopes granted on the token

Authenticate only performs the exchange; it does not read secrets or configure credentials. Downstream steps consume `${{ steps.authenticate.outputs.access_token }}` as a bearer token against services that validate it via the organization's JWKS at `https://login.ust.dev/orgs/{orgId}/jwks.json`.
