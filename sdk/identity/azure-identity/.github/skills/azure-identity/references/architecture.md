# Architecture Reference for azure-identity

## Credential Taxonomy

Credentials fall into five categories:

| Category | Examples | Base Class / Mixin |
|----------|---------|-------------------|
| **Service principal** | `ClientSecretCredential`, `CertificateCredential`, `ClientAssertionCredential` | `ClientCredentialBase` → MSAL `ConfidentialClientApplication` |
| **User / interactive** | `InteractiveBrowserCredential`, `DeviceCodeCredential`, `UsernamePasswordCredential` | `InteractiveCredential` → MSAL `PublicClientApplication` |
| **Developer tool** | `AzureCliCredential`, `AzurePowerShellCredential`, `AzureDeveloperCliCredential` | Direct subprocess calls (no MSAL) |
| **Managed identity** | `ManagedIdentityCredential` | Environment-sniffing dispatcher → backend-specific clients |
| **Chain** | `DefaultAzureCredential`, `ChainedTokenCredential` | Iterates child credentials |

## MSAL Integration

`_internal/msal_credentials.py` wraps MSAL with Azure Identity conventions:

- **Per-tenant app caching**: Separate `ConfidentialClientApplication` / `PublicClientApplication` per tenant ID
- **CAE separation**: Separate app and cache instances for Continuous Access Evaluation (CAE) vs non-CAE flows
- **Persistent cache**: When `TokenCachePersistenceOptions` is provided, uses `msal_extensions` for platform-specific encrypted storage (DPAPI on Windows, Keychain on macOS, libsecret on Linux)
- **Pickling support**: Apps and caches are non-picklable; `__getstate__`/`__setstate__` strip and recreate them

## Managed Identity Backends

`ManagedIdentityCredential` sniffs the environment and picks ONE backend:

| Environment | Detection | Backend Module |
|------------|-----------|----------------|
| App Service | `IDENTITY_ENDPOINT` + `IDENTITY_HEADER` | `app_service.py` |
| Service Fabric | `IDENTITY_ENDPOINT` + `IDENTITY_SERVER_THUMBPRINT` | `service_fabric.py` |
| Azure Arc | `IDENTITY_ENDPOINT` + `IMDS_ENDPOINT` | `azure_arc.py` |
| Azure ML | `MSI_ENDPOINT` + `MSI_SECRET` | `azure_ml.py` |
| Cloud Shell | `MSI_ENDPOINT` (no secret) | `cloud_shell.py` |
| Workload Identity | `AZURE_FEDERATED_TOKEN_FILE` + `AZURE_TENANT_ID` + `AZURE_CLIENT_ID` | `workload_identity.py` |
| IMDS (fallback) | None of the above | `imds.py` |

Each backend has sync and async implementations. Changes to managed identity must be tested across backends.

## DefaultAzureCredential Chain

The DAC constructs a `ChainedTokenCredential` with credentials in this order:

1. `EnvironmentCredential`
2. `WorkloadIdentityCredential`
3. `ManagedIdentityCredential`
4. `SharedTokenCacheCredential`
5. `VisualStudioCodeCredential`
6. `AzureCliCredential`
7. `AzurePowerShellCredential`
8. `AzureDeveloperCliCredential`
9. `InteractiveBrowserCredential` (if not excluded)
10. `BrokerCredential` (if `azure-identity-broker` installed, Windows/WSL only)

Each can be excluded via `exclude_<name>_credential=True`. If a credential fails to **initialize**, it's replaced with a `FailedDACCredential` stub that raises `CredentialUnavailableError` — this allows the chain to continue and report the initialization error only if all credentials fail.

The `AZURE_TOKEN_CREDENTIALS` env var or `require_envvar=True` can constrain DAC to only use service credentials (Environment, WorkloadIdentity, ManagedIdentity).

## Async Parity Rules

- Every sync credential in `_credentials/` has an async counterpart in `aio/_credentials/` with the same class name
- Async credentials use `async def get_token(...)`, `async def close()`, and `async with` context manager
- Shared logic should live in sync `_internal/` and be imported by async — avoid duplicating helpers
- The async `DefaultAzureCredential` chain uses `AsyncFailedDACCredential` (not `FailedDACCredential`)
- `aio/__init__.py` may export fewer symbols than sync (e.g., `DeviceCodeCredential` and `UsernamePasswordCredential` are sync-only in `__all__`)
