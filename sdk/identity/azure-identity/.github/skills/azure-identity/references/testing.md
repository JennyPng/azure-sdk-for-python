# Testing Guide for azure-identity

## Running Tests

```bash
cd sdk/identity/azure-identity

# Run unit tests (playback mode, no credentials needed)
azpysdk pytest .

# Run a specific test file
azpysdk pytest tests/test_default.py

# Run async tests
azpysdk pytest tests/test_default_async.py

# Run with keyword filter
azpysdk pytest tests/ -k "test_default_credential"
```

## Test Structure

Tests mirror the credential layout with sync/async pairs:

```
tests/
├── test_default.py / test_default_async.py
├── test_managed_identity.py / test_managed_identity_async.py
├── test_certificate_credential.py / test_certificate_credential_async.py
├── test_cli_credential.py / test_cli_credential_async.py
├── test_browser_credential.py / test_browser_credential_async.py
├── test_chained_credential.py / test_chained_credential_async.py
├── managed-identity-live/          # Live MI tests (require Azure environment)
├── helpers.py                      # Shared test utilities
└── conftest.py                     # Fixtures, sanitizers, env-gated marks
```

## Key Testing Patterns

- **`time.sleep` and `asyncio.sleep` are patched to no-op** in non-live mode (`conftest.py`). Don't rely on timing in unit tests.
- **Custom markers**: `@pytest.mark.manual` (requires human interaction), `@pytest.mark.prints` (outputs to terminal).
- **Live tests are env-gated**: Service principal creds, certificate paths, and username/password must be in env vars. Tests skip if vars are missing.
- **Managed identity live tests** are in a separate `managed-identity-live/` directory with their own `conftest.py`. They require Key Vault access and specific environment configuration.
- **Most tests mock MSAL** rather than hitting identity endpoints. Use `unittest.mock.patch` on MSAL methods, not on HTTP transport.
- **Async tests** use `pytest-asyncio`. The event loop policy is overridden on Windows to `WindowsSelectorEventLoopPolicy` in `conftest.py`.

## Validation

```bash
# Run full validation suite
azsdk_package_run_check with checkType="All"

# Or individual checks
azpysdk pylint .
azpysdk mypy .
```
