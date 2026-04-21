# Create GitHub App Token (with Reuse)

A wrapper around the official [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token) that adds **token reuse**. When you pass a previously generated token back via the `current` input, this action checks whether it is still valid (more than 5 minutes remaining) and skips regeneration if so. Otherwise it delegates to `actions/create-github-app-token@v3` to mint a fresh installation token.

This is useful in workflows that invoke the action multiple times across jobs or steps — rather than generating a new token each time (and counting against rate limits), you generate once and reuse until it expires.

## Usage

### Basic — generate a new token

If you don't pass `current`, the action always generates a new token, behaving identically to `actions/create-github-app-token`.

```yaml
steps:
  - name: Generate token
    id: app-token
    uses: mwerickso/create-github-app-token@v1
    with:
      client-id: ${{ vars.APP_CLIENT_ID }}
      private-key: ${{ secrets.APP_PRIVATE_KEY }}

  - name: Use token
    env:
      GH_TOKEN: ${{ steps.app-token.outputs.token }}
    run: gh api /repos/${{ github.repository }}
```

### Reuse an existing token across steps

Generate the token once, then pass the `result` output back on subsequent calls. If the token still has more than 5 minutes of validity, the action returns it as-is without calling the GitHub API.

```yaml
steps:
  - name: Generate token
    id: first
    uses: mwerickso/create-github-app-token@v1
    with:
      client-id: ${{ vars.APP_CLIENT_ID }}
      private-key: ${{ secrets.APP_PRIVATE_KEY }}

  # ... other steps that use the token ...

  - name: Reuse token
    id: second
    uses: mwerickso/create-github-app-token@v1
    with:
      client-id: ${{ vars.APP_CLIENT_ID }}
      private-key: ${{ secrets.APP_PRIVATE_KEY }}
      current: ${{ steps.first.outputs.result }}

  - name: Check if token was reused
    run: echo "Generated new token? ${{ steps.second.outputs.generated }}"
```

### Reuse a token across jobs

Pass the `result` output between jobs to avoid regenerating tokens.

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      token-result: ${{ steps.app-token.outputs.result }}
    steps:
      - name: Generate token
        id: app-token
        uses: mwerickso/create-github-app-token@v1
        with:
          client-id: ${{ vars.APP_CLIENT_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

  deploy:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - name: Reuse token
        id: app-token
        uses: mwerickso/create-github-app-token@v1
        with:
          client-id: ${{ vars.APP_CLIENT_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          current: ${{ needs.setup.outputs.token-result }}

      - name: Use token
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: gh api /repos/${{ github.repository }}
```

### Use with `actions/checkout`

```yaml
steps:
  - name: Generate token
    id: app-token
    uses: mwerickso/create-github-app-token@v1
    with:
      client-id: ${{ vars.APP_CLIENT_ID }}
      private-key: ${{ secrets.APP_PRIVATE_KEY }}

  - name: Checkout
    uses: actions/checkout@v4
    with:
      token: ${{ steps.app-token.outputs.token }}
```

### Configure git CLI for commits as the app bot

```yaml
steps:
  - name: Generate token
    id: app-token
    uses: mwerickso/create-github-app-token@v1
    with:
      client-id: ${{ vars.APP_CLIENT_ID }}
      private-key: ${{ secrets.APP_PRIVATE_KEY }}

  - name: Checkout
    uses: actions/checkout@v4
    with:
      token: ${{ steps.app-token.outputs.token }}

  - name: Configure git
    env:
      APP_SLUG: ${{ steps.app-token.outputs.app-slug }}
      USER_ID: ${{ steps.app-token.outputs.user-id }}
    run: |
      git config user.name "${APP_SLUG}[bot]"
      git config user.email "${USER_ID}+${APP_SLUG}[bot]@users.noreply.github.com"
```

## Inputs

| Name | Description | Required | Default |
| --- | --- | --- | --- |
| `client-id` | GitHub App client ID. | Yes | — |
| `private-key` | GitHub App private key (PEM format). | Yes | — |
| `owner` | The owner of the GitHub App installation. | No | Current repository owner |
| `repositories` | Comma-separated list of repositories to grant access to. | No | Current repository only |
| `current` | Base64-encoded JSON from a previous invocation's `result` output. When provided and the token has more than 5 minutes of validity remaining, the action reuses it instead of generating a new one. | No | `""` |

> [!NOTE]
> The `client-id`, `private-key`, `owner`, and `repositories` inputs are passed directly to [`actions/create-github-app-token@v3`](https://github.com/actions/create-github-app-token) when a new token is needed. Refer to that action's documentation for additional context on GitHub App setup and credential management.

## Outputs

| Name | Description |
| --- | --- |
| `token` | The installation token (plain text). |
| `app-slug` | The GitHub App slug. |
| `user-id` | The GitHub App bot user ID. |
| `result` | Base64-encoded JSON containing `token`, `expires-at`, `app-slug`, and `user-id`. Pass this value back as the `current` input on subsequent calls to enable token reuse. |
| `generated` | `"true"` if a new token was generated, `"false"` if an existing token was reused. |

For most use cases, reference the direct outputs:

```yaml
${{ steps.app-token.outputs.token }}
${{ steps.app-token.outputs.app-slug }}
${{ steps.app-token.outputs.user-id }}
```

The `result` output is only needed when passing tokens between steps or jobs via the `current` input.

## How it works

```
┌─────────────────────────┐
│ current input provided? │
└────────┬────────────────┘
         │
    No   │   Yes
    │    │    │
    │    │    ▼
    │    │  ┌──────────────────────────┐
    │    │  │ Parse token & expires-at │
    │    │  └────────┬─────────────────┘
    │    │           │
    │    │           ▼
    │    │  ┌──────────────────────────┐
    │    │  │ Remaining > 5 minutes?   │
    │    │  └────┬─────────────┬───────┘
    │    │       │ Yes         │ No
    │    │       ▼             │
    │    │  Return existing    │
    │    │  token (reuse)      │
    │    │  generated=false    │
    │    │                     │
    ▼    ▼                     ▼
┌──────────────────────────────────┐
│ actions/create-github-app-token  │
│ @v3                              │
│                                  │
│ Generates new installation token │
│ via GitHub REST API              │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│ Fetch bot user ID via            │
│ GET /users/{app-slug}[bot]       │
└────────────────┬─────────────────┘
                 │
                 ▼
        Return JSON result
        generated=true
```

1. **Check existing token** — If the `current` input is provided and contains a `token` and `expires-at` field, the action calculates the remaining validity. If more than **300 seconds (5 minutes)** remain, the token is returned as-is.

2. **Generate new token** — Otherwise, the action delegates to [`actions/create-github-app-token@v3`](https://github.com/actions/create-github-app-token), which creates an installation access token using the [Create an installation access token for an app](https://docs.github.com/en/rest/apps/apps#create-an-installation-access-token-for-an-app) REST API endpoint.

3. **Resolve bot identity** — After generating a new token, the action queries the GitHub API for the app's bot user ID (`GET /users/{app-slug}[bot]`), which is useful for configuring git commits as the app.

4. **Return result** — The token, expiration timestamp, app slug, and user ID are bundled into a JSON object and returned via the `result` output.

> [!IMPORTANT]
> Installation tokens generated by `actions/create-github-app-token` expire after **1 hour**. The 5-minute threshold ensures you never receive a token that is about to expire before your step can use it.

## License

[MIT](LICENSE)
