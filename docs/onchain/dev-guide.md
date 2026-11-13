---
title: Token template — developer integration
---

# Token template — developer integration

How the template is consumed by generators, how to deploy a generated coin correctly, and how to
integrate with the resulting coin. See the [API reference](api-reference.md) for the full surface.

## Normative requirements

1. **`finalize_registration` MUST run after publish.** Until it does, `Currency<T>` is pending
   (owned by `0xc`) and wallets/indexers cannot resolve the coin. Anyone may call it:
   `0x2::coin_registry::finalize_registration<T>(0xc registry, pending Currency)`.
2. **Supply and metadata policy MUST be decided at generation time** — `INITIAL_SUPPLY`,
   `FIXED_SUPPLY` and `FROZEN_METADATA` are compiled constants, applied by `init` in the publish
   transaction. A fixed supply leaves no `TreasuryCap` and frozen metadata no `MetadataCap`; every
   cap and coin that does exist lands with the publisher, whose custody MUST be decided explicitly.
   The registry records both guarantees (`is_supply_fixed`, `is_metadata_cap_deleted`).
3. **Immutability MUST be verified by the UpgradeCap being consumed**, not by the package object's
   owner (a package is always `Immutable`-owned).
4. **Generators MUST patch only the designated constants.** The distinct placeholder constants
   (`TEMPLATE_SYMBOL`, `TEMPLATE_NAME`, `TEMPLATE_DESCRIPTION`, `TEMPLATE_ICON_URL`, `DECIMALS = 9`,
   `INITIAL_SUPPLY = 1_000_000_007`, `FIXED_SUPPLY = true`, `FROZEN_METADATA = false` — the two
   booleans differ so each keeps its own pool entry)
   and the identifiers `token_issuance_template` / `TOKEN_ISSUANCE_TEMPLATE` are the contract with
   `@meddleware/token-issuance-ui` (bytecode patching via `@mysten/move-bytecode-template`) and
   `scripts/03_create_token.sh` (source generation). Renaming or merging them breaks both.

## Coupling with `@meddleware/token-issuance-client`

The compiled template, its patcher and the text files of the downloadable source package live in `@meddleware/token-issuance-client`
(`token-issuance-ui` only draws the form and calls it). The client regenerates them from this package:

| Artefact in token-issuance-client | Derived from | Kept in sync by |
| --- | --- | --- |
| `src/template/artifact.ts`: `TEMPLATE_MODULE_B64`, `TEMPLATE_BUILD_INFO` | the compiled template bytecode and `build-info.json` of this package | `npm run gen:template` (reads the pinned `@meddleware/token-issuance-template-sui` devDependency) |
| `src/template/files.ts`: `TEMPLATE_FILES` (the downloadable source package) | `Move.toml`, the source, `scripts/publish.sh` and the other text files | `npm run gen:template`; `npm run check:template` fails when the committed files differ |
| `TEMPLATE_DEFAULTS` / `TEMPLATE_IDENTIFIERS` | the constants and identifiers the patcher replaces | `gen:template` checks them against both the source and the bytecode's constant pool |

The generator checks itself before it writes: the bytecode and source hashes match `build-info.json`, the module decodes and re-encodes byte for byte,
and every identifier and default the patcher replaces is present. After publishing a new template version: bump the devDependency in token-issuance-client,
run `npm run gen:template` and then `npm run check:template`, release the client, and bump it in the apps.

## Deploying a generated package

```bash
# generated package, active env must equal the target network
MYTOKEN_ICON_URL=https://… bash scripts/publish.sh testnet            # prompts before burning
MYTOKEN_ICON_URL=https://… bash scripts/publish.sh testnet --confirm-immutable
```

Each step saves IDs to `.env.<network>`; re-running resumes (no second publish).

## Integrating with a generated coin

- Balances and transfers use the standard `Coin<T>` / address-balance APIs — nothing custom.
- Metadata: `client.getCoinMetadata({ coinType })` (gRPC) returns the registry `Currency`.
- Minting (`0x2::coin::mint`) and burning (`0x2::coin::burn`) require the `TreasuryCap`. A coin with
  a fixed supply has none: `Currency<T>` reports `is_supply_fixed` and `total_supply` instead.

## Command-line deployment paths

Besides the Token Issuance app, a coin package can be built and published from this repository:

| Path | Registration | Supply / metadata policy | UpgradeCap |
| --- | --- | --- | --- |
| `deploy_token.sh` (CLI) | finalized | same constants (`TOKEN_INITIAL_SUPPLY`, `TOKEN_SUPPLY_POLICY`, `TOKEN_METADATA_POLICY`); the TreasuryCap (if any) and the initial supply go to `TREASURY_ADDRESS`; the MetadataCap (if any) stays with the deployer | always burned |
| generated `scripts/publish.sh` | finalized; icon set per network unless the metadata is frozen | what remains stays with the deployer | burned only with `--confirm-immutable` / `CONFIRM` |
