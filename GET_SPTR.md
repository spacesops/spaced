# GET_SPTR - Get Pointer Information

## Overview

The `getptr` command retrieves information about a Space Pointer (SPTR) from the spaced RPC server. A Space Pointer is a UTXO that points to a specific script pubkey and can contain optional data.

## CLI Usage

### Basic Syntax

```bash
spaces getptr <sptr>
```

### Example

```bash
spaces getptr sptr1z5fu9gsrvhgj74alq829n2z7pq6u9y5ae856gxkdxjzdk9myh50qgx5zaj
```

### With JSON Output Formatting

```bash
spaces getptr sptr1z5fu9gsrvhgj74alq829n2z7pq6u9y5ae856gxkdxjzdk9myh50qgx5zaj | jq .
```

## JSON-RPC API Usage

### Method

`getptr`

### Parameters

- `ptr` (string): The Space Pointer identifier (SPTR) in bech32 format

### Request Format

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $AUTH_TOKEN" \
  -d '{"jsonrpc":"2.0","method":"getptr","params":["<sptr>"],"id":1}' \
  http://127.0.0.1:7224 | jq .
```

### Authentication Setup

Before making the request, set up the authentication token:

```bash
export AUTH_TOKEN=$(echo -n "$SPACED_RPC_USER:$SPACED_RPC_PASSWORD" | base64)
```

Ensure the following environment variables are set:
- `SPACED_RPC_USER`: RPC username (e.g., `testuser` for testnet4, `admin` for mainnet)
- `SPACED_RPC_PASSWORD`: RPC password (e.g., `SomeRisk84` for testnet4, `admin` for mainnet)

### RPC Endpoints

- **Testnet4**: `http://127.0.0.1:7224`
- **Mainnet**: `http://127.0.0.1:7225`

### Example Request

```bash
export AUTH_TOKEN=$(echo -n "$SPACED_RPC_USER:$SPACED_RPC_PASSWORD" | base64)

curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $AUTH_TOKEN" \
  -d '{"jsonrpc":"2.0","method":"getptr","params":["sptr1z5fu9gsrvhgj74alq829n2z7pq6u9y5ae856gxkdxjzdk9myh50qgx5zaj"],"id":1}' \
  http://127.0.0.1:7224 | jq .
```

## Response Format

The response is a JSON object containing pointer information, or `null` if the pointer is not found.

### Success Response

```json
{
  "txid": "a92901cbc8b021e80ba9554ebfacdfb89cd0cee2e00db62fdeb820ed543d9867",
  "n": 0,
  "id": "sptr1z5fu9gsrvhgj74alq829n2z7pq6u9y5ae856gxkdxjzdk9myh50qgx5zaj",
  "data": "01000d6b6e6f776e40746162636f6e66011e687474703a2f2f37302e3235312e3230392e3230372f6170692d646f6373",
  "last_update": 117835,
  "value": 1007,
  "script_pubkey": "5120d9d96c98d1a21b45a962c77d4e448adc0de60dc13b7d56f3603d4ae9d3acfba3"
}
```

### Response Fields

- `txid` (string): Transaction ID where the pointer UTXO was created
- `n` (integer): Output index (vout) of the pointer UTXO
- `id` (string): The Space Pointer identifier (SPTR) in bech32 format
- `data` (string|null): Optional hex-encoded data associated with the pointer. Can be `null` if no data is stored.
- `last_update` (integer): Block height of the last update to this pointer
- `value` (integer): Value of the UTXO in satoshis
- `script_pubkey` (string): Hex-encoded script pubkey that this pointer references
- `genesis_spk` (string, optional): The original genesis script pubkey (may be present in some responses)

### Not Found Response

If the pointer does not exist or has not been confirmed yet:

```json
null
```

## Examples

### Example 1: Get pointer with data

```bash
spaces getptr sptr1z5fu9gsrvhgj74alq829n2z7pq6u9y5ae856gxkdxjzdk9myh50qgx5zaj
```

Response:
```json
{
  "txid": "a92901cbc8b021e80ba9554ebfacdfb89cd0cee2e00db62fdeb820ed543d9867",
  "n": 0,
  "id": "sptr1z5fu9gsrvhgj74alq829n2z7pq6u9y5ae856gxkdxjzdk9myh50qgx5zaj",
  "data": "01000d6b6e6f776e40746162636f6e66011e687474703a2f2f37302e3235312e3230392e3230372f6170692d646f6373",
  "last_update": 117835,
  "value": 1007,
  "script_pubkey": "5120d9d96c98d1a21b45a962c77d4e448adc0de60dc13b7d56f3603d4ae9d3acfba3"
}
```

### Example 2: Get pointer without data

```bash
spaces getptr sptr14frcyqqrnwydthgj7u09a8vq78wr4eewnwevffrzpake7ng9crkssgd5nz
```

Response:
```json
{
  "txid": "cdd18f6b142722a5f9ab22f2ca5283fc9ea2b620fd413628dd885048cedb5d66",
  "n": 0,
  "value": 1007,
  "script_pubkey": "5120489cb40915375e750fcf6dc036a4bf555db7c670afd06e6de76f4614f95af012",
  "genesis_spk": "5120489cb40915375e750fcf6dc036a4bf555db7c670afd06e6de76f4614f95af012",
  "data": null
}
```

### Example 3: Using curl with JSON-RPC

```bash
export AUTH_TOKEN=$(echo -n "testuser:SomeRisk84" | base64)

curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $AUTH_TOKEN" \
  -d '{"jsonrpc":"2.0","method":"getptr","params":["sptr16yxrh2x04yhhxrw2qjyc7v60x2ey2q4858wnw7g3ylzyxz4pzm0qarf4lv"],"id":1}' \
  http://127.0.0.1:7224 | jq .
```

## Notes

- The pointer must exist on-chain and be confirmed before it can be retrieved
- If a pointer was just created, you may need to wait for transaction confirmation before `getptr` returns results
- The `data` field contains hex-encoded data that was stored when the pointer was created (via `createptr`)
- The `last_update` field indicates the block height when the pointer was last modified
- Pointers are identified by their bech32-encoded SPTR format, which is derived from the script pubkey

## Related Commands

- `createptr`: Create a new space pointer
- `getallptrs`: Get all pointers (with optional data filtering)
- `getptrout`: Get pointer information by outpoint (txid:n)
- `transferptr`: Transfer a pointer to a new address
