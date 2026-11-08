# Changelog

All notable changes to `@meddleware/token-issuance-template-sui` are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.0.13] - 2026-11-08

### Fixed

- Documentation and source comments said the coin name is permanent. The Sui coin registry lets the `MetadataCap` holder change the name (`coin_registry::set_name`), description and icon URL until the cap is deleted; only the symbol and decimals are permanent. The shipped bytecode is unchanged (same hash); only its recorded source hash moved.

## [1.0.12] - 2026-11-07

### Changed

- The on-chain overview no longer mentions the command-line pipeline or an "operator fee" (user-facing); the command-line paths moved to the developer guide.
