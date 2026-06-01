---
name: azure-identity
description: 'Domain knowledge for azure-identity. Covers architecture, credential chain, MSAL integration, managed identity, async parity, testing, and common pitfalls. WHEN: modify azure-identity; fix azure-identity bug; add azure-identity credential; azure-identity feature; change DefaultAzureCredential; update managed identity.'
---

# azure-identity — Package Skill

`azure-identity` is **entirely hand-authored** — there is no TypeSpec, no `_generated/` directory, no `_patch.py` files, and no code generation step. Every file is owned and maintained directly. This fundamentally changes how an agent should approach modifications compared to generated SDK packages.

## Architecture

```
azure/identity/
├── __init__.py                  # Public API re-exports; __all__ is the contract
├── _version.py                  # VERSION string
├── _constants.py                # EnvironmentVariables, AzureAuthorityHosts, KnownAuthorities
├── _enums.py                    # RegionalAuthority, TokenRefreshStatus
├── _auth_record.py              # AuthenticationRecord (serializable identity snapshot)
├── _bearer_token_provider.py    # get_bearer_token_provider() helper
├── _persistent_cache.py         # TokenCachePersistenceOptions, platform-specific cache loading
├── _exceptions.py               # CredentialUnavailableError, AuthenticationRequiredError
├── _credentials/                # Sync credential implementations (one file per credential)
│   ├── default.py               # DefaultAzureCredential — the chain
│   ├── chained.py               # ChainedTokenCredential — base chain logic
│   ├── managed_identity.py      # ManagedIdentityCredential — env-sniffing dispatcher
│   ├── environment.py           # EnvironmentCredential — reads env vars
│   ├── certificate.py           # CertificateCredential
│   ├── client_secret.py         # ClientSecretCredential
│   ├── browser.py               # InteractiveBrowserCredential
│   ├── device_code.py           # DeviceCodeCredential
│   ├── azure_cli.py             # AzureCliCredential
│   ├── azure_powershell.py      # AzurePowerShellCredential
│   ├── azd_cli.py               # AzureDeveloperCliCredential
│   ├── workload_identity.py     # WorkloadIdentityCredential
│   ├── azure_pipelines.py       # AzurePipelinesCredential
│   ├── broker.py                # BrokerCredential (WAM, Windows/WSL only)
│   └── ...                      # on_behalf_of, shared_cache, vscode, etc.
├── _internal/                   # Shared plumbing (NOT public API)
│   ├── get_token_mixin.py       # GetTokenMixin — the standard token acquisition flow
│   ├── msal_credentials.py      # MSAL app wrapper (ConfidentialClient/PublicClient)
│   ├── msal_client.py           # Low-level MSAL HTTP adapter
│   ├── managed_identity_client.py      # HTTP-based managed identity client
│   ├── msal_managed_identity_client.py # MSAL-based managed identity client
│   ├── client_credential_base.py       # Base for confidential client credentials
│   ├── interactive.py           # Base for interactive credentials
│   ├── decorators.py            # Logging decorators (log_get_token, log_get_token_async)
│   ├── utils.py                 # Authority normalization, DAC helpers
│   └── pipeline.py              # Custom HTTP pipeline for identity
└── aio/                         # Async mirror
    ├── __init__.py              # Async public API re-exports
    ├── _credentials/            # Async credential implementations (mirrors sync)
    └── _internal/               # Async plumbing (mirrors sync)
```

## Common Pitfalls

1. **Every sync credential MUST have an async mirror.** Adding a sync credential without its `aio/` counterpart breaks the async `DefaultAzureCredential` chain. The async version is in `aio/_credentials/` with the same filename.

2. **`__all__` in both `__init__.py` files.** New public symbols must be added to BOTH `azure/identity/__init__.py` AND `azure/identity/aio/__init__.py` (if async-applicable). Missing entries silently hide the class from users.

3. **`DefaultAzureCredential` chain order matters.** The chain is defined in `_credentials/default.py` (sync) and `aio/_credentials/default.py` (async). New credentials must be inserted at the correct position in BOTH files. The `exclude_*` kwargs must also be added.

4. **Multi-tenant: `additionally_allowed_tenants`.** Since v1.14.0, credentials reject cross-tenant token requests unless the target tenant is in `additionally_allowed_tenants` or `*` is specified. Forgetting this causes `ClientAuthenticationError`.

5. **MSAL is the core dependency.** Most credentials delegate to MSAL (`msal` and `msal-extensions`). The `_internal/msal_credentials.py` adapter manages separate app/cache instances per tenant, and separate instances for CAE vs non-CAE. Don't bypass MSAL for token acquisition.

6. **Managed identity is an environment-sniffing dispatcher.** `ManagedIdentityCredential` detects the hosting environment (App Service, IMDS, Azure Arc, Service Fabric, Cloud Shell, Azure ML, Workload Identity) and picks the right backend. Changes to managed identity must handle ALL backends, not just one.

7. **Don't hardcode authority URLs.** Use `AzureAuthorityHosts` constants and `get_default_authority()` / `normalize_authority()` from `_internal/utils.py`. Hardcoded URLs break sovereign clouds.

8. **Token refresh has three states.** `TokenRefreshStatus` in `_enums.py` defines `REQUIRED`, `RECOMMENDED`, and `NOT_NEEDED`. The `GetTokenMixin.get_token()` flow in `_internal/get_token_mixin.py` uses all three. Don't treat refresh as a simple boolean.

## Key Internal Flow: `GetTokenMixin`

Most credentials inherit `GetTokenMixin`, which implements the standard `get_token()` flow:

1. Validate at least one scope is provided
2. Try `_acquire_token_silently()` (cache hit / refresh token)
3. Check `get_refresh_status()` → `REQUIRED` | `RECOMMENDED` | `NOT_NEEDED`
4. If `REQUIRED` or `RECOMMENDED`, call `_request_token()` (full auth flow)
5. Log result — log level depends on `within_credential_chain` context

Each credential implements `_acquire_token_silently()` and `_request_token()`.

## Testing

See `references/testing.md` for test commands and patterns.

## References

- Detailed testing guide: `references/testing.md`
- Credential architecture and MSAL details: `references/architecture.md`
