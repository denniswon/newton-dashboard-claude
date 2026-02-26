---
paths:
  - "app/core/blockchain/**"
  - "scripts/sync_abis.py"
---

# Blockchain Integration Guidelines

## Architecture

### Module Structure

```
app/core/blockchain/
    __init__.py            # Package docstring and public exports
    config.py              # ChainId enum, chain names, constants
    provider.py            # AsyncWeb3 per-chain singleton with retry
    registry.py            # ABI loader and deployment address registry
    base_contract.py       # BaseContract async wrapper
    abi/                   # Extracted ABI JSON files (synced from Foundry)
        NewtonPolicy.json
        NewtonPolicyClient.json
        ...
    deployments/           # Per-chain deployment addresses
        1-prod.json
        11155111-stagef.json
        ...
    contracts/             # Type-safe contract wrappers
        policy.py
        policy_client.py
        policy_factory.py
        registry.py
```

### Key Contracts

| Contract | Purpose | Dashboard Usage |
|----------|---------|-----------------|
| NewtonPolicyClient | Per-user smart contract wallet | KYC ownership verification |
| NewtonPolicy | Policy configuration and CIDs | Policy metadata reads |
| NewtonPolicyFactory | Deterministic policy deployment | Policy lookup by owner |
| PolicyClientRegistry | Client registration tracking | Registration status checks |
| NewtonProverTaskManager | Task creation and attestation | Read-only status queries |

## Async-First Web3

### Always Use AsyncWeb3

All on-chain calls must be async. Synchronous `Web3` blocks the FastAPI event loop.

```python
# Good: Async contract call via typed wrapper
from app.core.blockchain.contracts import NewtonPolicyClientContract

async def check_owner(address: str, chain_id: int) -> str:
    contract = NewtonPolicyClientContract(address, chain_id)
    return await contract.get_owner()

# Bad: Synchronous Web3 blocks the event loop
from web3 import Web3
web3 = Web3(Web3.HTTPProvider(url))
owner = web3.eth.contract(...).functions.getOwner().call()
```

### Provider Singletons

Use `get_async_web3()` from `app.core.blockchain.provider` — it returns a cached singleton per chain. Never create `AsyncWeb3` instances directly.

```python
# Good: Singleton provider
from app.core.blockchain.provider import get_async_web3
w3 = await get_async_web3(chain_id)

# Bad: Creates a new provider per call
w3 = AsyncWeb3(AsyncHTTPProvider(url))
```

## Contract Wrappers

### Adding a New Contract Wrapper

1. Extract the ABI: add the contract name to `CONTRACTS` in `scripts/sync_abis.py` and run it
2. Create a wrapper class inheriting from `BaseContract`:

```python
from app.core.blockchain.base_contract import BaseContract

class MyContract(BaseContract):
    CONTRACT_ABI_NAME = "MyContractName"

    async def my_function(self) -> str:
        return await self._call("myFunction")
```

3. Export from `app/core/blockchain/contracts/__init__.py`
4. Use in routes/services via direct instantiation:

```python
contract = MyContract(address, chain_id)
result = await contract.my_function()
```

### BaseContract._call() Pattern

Use `_call()` for all read-only contract interactions. It handles:
- Lazy contract initialization
- `ContractLogicError` → `AuthorizationError` conversion
- Logging of failures

Do not call `contract.functions.X().call()` directly in wrapper methods.

### Address Handling

Always use `AsyncWeb3.to_checksum_address()` for addresses returned from on-chain calls. Accept both checksummed and lowercase addresses as input — normalize in the wrapper.

```python
async def get_owner(self) -> str:
    owner = await self._call("getOwner")
    return AsyncWeb3.to_checksum_address(owner)
```

## ABI and Deployment Management

### Syncing from Foundry

ABIs and deployment addresses are synced from `newton-prover-avs/contracts/`:

```bash
python scripts/sync_abis.py
```

The script:
- Extracts the `"abi"` array from Foundry artifacts (`contracts/out/{Name}.sol/{Name}.json`)
- Merges per-category deployment files into per-chain JSONs
- Writes to `app/core/blockchain/abi/` and `app/core/blockchain/deployments/`

Run this after contract changes in `newton-prover-avs`. Commit the extracted files.

### ABI Files

- Store only the ABI array, not the full Foundry artifact (bytecode, metadata)
- One file per contract: `{ContractName}.json`
- Loaded via `registry.get_abi("ContractName")` (LRU cached)

### Deployment Files

- Format: `{chainId}-{environment}.json` (e.g., `1-prod.json`, `11155111-stagef.json`)
- Environment mapping: `local` and `stagef` → `stagef` deployments, everything else → `prod`
- Loaded via `registry.get_deployment(chain_id)` or `registry.get_contract_address(name, chain_id)`

### Adding a New Chain

1. Add the chain to `ChainId` enum in `config.py`
2. Add an optional RPC URL field to `RPC` model in `secrets.py`
3. Add the chain to `CHAIN_FIELD_MAPPING` in `secrets.py`
4. Add the chain name to `CHAIN_NAMES` in `config.py`
5. Add deployment files via `scripts/sync_abis.py`
6. Add `RPC__{field_name}` to `.env`

## Configuration

### RPC URLs

RPC URLs are configured via environment variables in `secrets.py`:

| Chain | Env Variable | Required |
|-------|-------------|----------|
| Ethereum Mainnet (1) | `RPC__ETHEREUM_MAINNET_URL` | Yes |
| Sepolia (11155111) | `RPC__SEPOLIA_URL` | Yes |
| Base (8453) | `RPC__BASE_URL` | No |
| Base Sepolia (84532) | `RPC__BASE_SEPOLIA_URL` | No |
| Anvil (31337) | `RPC__ANVIL_URL` | No |

`RPC.supported_chain_ids` returns only chains with a configured URL.

### Retry Configuration

The provider uses `ExceptionRetryConfiguration`:
- 3 retries with 0.25s backoff factor
- Retries on `ConnectionError`, `TimeoutError`, `OSError`
- 30-second request timeout

## Error Handling

### On-Chain Error Hierarchy

`BaseContract._call()` raises `AuthorizationError` (→ HTTP 403) for:
- Contract reverts (`ContractLogicError`)
- RPC connection failures within contract calls

Higher-level utilities (e.g., `onchain_verification.py`) may raise `BlockchainError` (→ HTTP 502) for infrastructure-level failures like RPC connectivity, and `AuthorizationError` for ownership mismatches.

### Do Not Catch AuthorizationError or BlockchainError in Wrappers

Let errors propagate to the route layer where FastAPI middleware converts them to the appropriate HTTP response.

```python
# Good: Let errors propagate
owner = await policy_client.get_owner()

# Bad: Catching and re-wrapping
try:
    owner = await policy_client.get_owner()
except AuthorizationError:
    raise HTTPException(403, "verification failed")  # Redundant
```

## Testing

### Mock the Provider

For unit tests, mock `get_async_web3` or the contract wrapper methods directly:

```python
from unittest.mock import AsyncMock, patch

@patch("app.core.blockchain.contracts.policy_client.NewtonPolicyClientContract.get_owner")
async def test_verify_owner(mock_get_owner):
    mock_get_owner.return_value = "0xExpectedAddress..."
    result = await verify_contract_owner(contract_addr, expected_owner, chain_id=1)
    assert result == "0xExpectedAddress..."
```

### Clear Singletons Between Tests

Call `clear_providers()` in test fixtures to reset cached Web3 instances:

```python
from app.core.blockchain.provider import clear_providers

@pytest.fixture(autouse=True)
def reset_web3():
    yield
    clear_providers()
```

## Anti-patterns

| Anti-pattern | Problem | Solution |
|--------------|---------|----------|
| Synchronous `Web3` | Blocks event loop | Use `AsyncWeb3` via provider singleton |
| Inline ABIs | Hard to maintain, no sync | Use `registry.get_abi()` with synced files |
| Hardcoded addresses | Breaks across environments | Use `registry.get_contract_address()` |
| Direct `AsyncWeb3()` | No retry, no caching | Use `get_async_web3()` |
| Raw `contract.functions.X().call()` | No error handling | Use `BaseContract._call()` |
| Catching `AuthorizationError` in wrappers | Redundant wrapping | Let errors propagate |
