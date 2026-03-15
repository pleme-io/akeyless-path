# akeyless-path Design

## Motivation

sops works standalone: `sops -d secrets.yaml` decrypts and prints.
No Nix module, no activation scripts, no generations.

Akeyless has `akeyless get-secret-value`, but it's a single-secret CLI
with no batch mode, no permission setting, and no config file.

akeyless-path bridges this: a batch secret fetcher with declarative config,
proper file permissions, and integration with akeyless-auth for biometric auth.

## Design Principles

1. **Minimal** — fetch and write, nothing else
2. **Declarative** — TOML config declares secret→path mappings
3. **Composable** — works standalone, with akeyless-auth, or inside akeyless-nix
4. **Trait-based** — every I/O boundary behind a trait for testing

## Flow

```
Config (TOML) → Authenticator → SecretFetcher → SecretWriter
                    │                  │               │
                    ▼                  ▼               ▼
              akeyless-auth      akeyless-api      std::fs
              (Touch ID)         (GET secret)      (write + chmod)
```

## Error Handling

- Auth failure → hard fail (no fallback, unlike akeyless-nix cache)
- Secret not found → report and continue (configurable: fail-fast or best-effort)
- Write failure → hard fail (permission denied, parent missing)
- Partial success → report which secrets succeeded/failed

## Future Considerations

- **Watch mode**: poll for changes and re-fetch (like `sops exec-file`)
- **Template support**: could delegate to igata for template rendering
- **Diff mode**: only write if value changed (avoid unnecessary file writes)
