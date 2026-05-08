# akeyless-path

> **★★★ CSE / Knowable Construction.** This repo operates under **Constructive Substrate Engineering** — canonical specification at [`pleme-io/theory/CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](https://github.com/pleme-io/theory/blob/main/CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md). The Compounding Directive (operational rules: solve once, load-bearing fixes only, idiom-first, models stay current, direction beats velocity) is in the org-level pleme-io/CLAUDE.md ★★★ section. Read both before non-trivial changes.


Minimal secret fetcher for Akeyless — the `sops -d` equivalent.

## Problem

`akeyless-nix` is a full Nix activation-time secret manager with generations,
symlinks, templates, and cache. But sometimes you just need to fetch a secret
and write it to a file — like how `sops -d secrets.yaml` works standalone.

`akeyless-path` fills this gap: declare secret→path mappings, fetch, write.
No generations, no symlinks, no Nix module required.

## Architecture

```
akeyless-path fetch --config secrets.toml
  1. Authenticate to Akeyless (API key, JWT via akeyless-auth, or env vars)
  2. For each secret in config: GET /get-secret-value
  3. Write to target path with specified permissions
  4. Done. No generations, no symlinks, no state.
```

## Config Format

```toml
# secrets.toml
[auth]
method = "jwt"                    # "api_key", "jwt", "env"
jwt_socket = "~/.config/akeyless-auth/sock"  # for akeyless-auth integration

[defaults]
mode = "0600"

[[secrets]]
path = "/pleme/github/token"
target = "~/.config/github/token"

[[secrets]]
path = "/pleme/db/password"
target = "~/.config/app/db-pass"
mode = "0400"

[[secrets]]
path = "/pleme/k8s/cert"
target = "/var/lib/pki/cert.pem"
owner = "root"
group = "root"
```

## CLI

```bash
# Fetch all secrets from config
akeyless-path fetch --config secrets.toml

# Fetch a single secret
akeyless-path get /pleme/github/token --output ~/.config/github/token

# Verify all secrets are accessible (dry run)
akeyless-path check --config secrets.toml

# Show config and secret status
akeyless-path status --config secrets.toml
```

## Auth Methods

| Method | Config | Description |
|--------|--------|-------------|
| `api_key` | `access_id`, `access_key_file` | Static API key (from file, not inline) |
| `jwt` | `jwt_socket` | akeyless-auth daemon (Touch ID per session) |
| `env` | `AKEYLESS_ACCESS_ID`, `AKEYLESS_ACCESS_KEY` | Environment variables |

## Integration Points

- **akeyless-auth**: JWT authentication via Unix socket (biometric-gated)
- **akeyless-api**: Rust SDK for typed API calls
- **akeyless-nix**: Could use akeyless-path internally for the fetch step
- **blackmatter-secrets**: Could add an `akeyless-path` backend for simple deployments

## Traits

```rust
trait SecretFetcher: Send + Sync {
    async fn fetch(&self, path: &str) -> Result<String>;
}

trait SecretWriter: Send + Sync {
    fn write(&self, target: &Path, value: &str, mode: u32) -> Result<()>;
}

trait Authenticator: Send + Sync {
    async fn authenticate(&self) -> Result<String>;  // returns token
}
```

## Dependencies

- `akeyless-api` (git: pleme-io/akeyless-api) — typed Akeyless SDK
- `clap` — CLI
- `serde` + `toml` — config parsing
- `tokio` — async runtime

## Nix Module (optional)

Simple HM/darwin/NixOS module that runs `akeyless-path fetch` at activation:

```nix
akeyless-path = {
  enable = true;
  configFile = ./secrets.toml;
  # OR inline:
  secrets = {
    "/pleme/github/token" = { target = "~/.config/github/token"; };
  };
};
```

## Comparison

| Feature | sops -d | akeyless-path | akeyless-nix |
|---------|---------|---------------|-------------|
| Fetch & write | Yes | Yes | Yes |
| Generations | No | No | Yes |
| Symlinks | No | No | Yes |
| Templates | No | No | Yes |
| Cache | No | No | Yes |
| Offline | Yes | No | Yes (cache) |
| Nix module | sops-nix | Optional | Full |
| Complexity | Minimal | Minimal | Full |
