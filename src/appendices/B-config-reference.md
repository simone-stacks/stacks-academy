# Appendix B: Node Configuration Reference

The Stacks node is configured via a TOML file passed at startup. The configuration is parsed by `ConfigFile::from_path()` into a `ConfigFile` struct, then converted to the runtime `Config` struct via `Config::from_config_file()`. Sample configurations are in `sample/conf/`.

**Source:** `stackslib/src/config/mod.rs` (all config structs and defaults)

---

## Minimal Examples

### Mainnet Follower

```toml
[node]
rpc_bind = "0.0.0.0:20443"
p2p_bind = "0.0.0.0:20444"

[burnchain]
mode = "mainnet"
peer_host = "127.0.0.1"
```

### Mainnet Miner

```toml
[node]
rpc_bind = "127.0.0.1:20443"
p2p_bind = "127.0.0.1:20444"
seed = "YOUR_HEX_PRIVATE_KEY"
miner = true

[burnchain]
mode = "mainnet"
peer_host = "127.0.0.1"
username = "bitcoin_rpc_user"
password = "bitcoin_rpc_pass"
burn_fee_cap = 20000
satoshis_per_byte = 25
```

---

## `[node]` Section

Controls the node's identity, networking, and operational mode.

**Struct:** `NodeConfig` at `stackslib/src/config/mod.rs:1932`

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `String` | `"helium-node"` | Human-readable node name (used in logging and temp directories). |
| `seed` | `String` (hex) | Random 32 bytes | Private key seed for the node's keychain. Required for miners. |
| `working_dir` | `String` | `/tmp/stacks-node-{timestamp}` | Root directory for all persistent data. Override with `STACKS_WORKING_DIR` env var. |
| `rpc_bind` | `String` | `"0.0.0.0:20443"` | Address and port for the HTTP RPC server. |
| `p2p_bind` | `String` | `"0.0.0.0:20444"` | Address and port for the P2P listener. |
| `data_url` | `String` | Derived from `rpc_bind` | Publicly advertised RPC URL for peer discovery. |
| `p2p_address` | `String` | Derived from `rpc_bind` | Publicly advertised P2P address. Configure explicitly if behind NAT. |
| `local_peer_seed` | `String` (hex) | Random 32 bytes | Separate private key for P2P identity and message signing. |
| `bootstrap_node` | `String` | `""` | Comma-separated list of `"PUBKEY@HOST:PORT"` bootstrap peers. |
| `deny_nodes` | `String` | `""` | Comma-separated list of `"HOST:PORT"` peers to deny. |
| `miner` | `bool` | `false` | Enable mining. Requires `seed` or `[miner].mining_key`. |
| `stacker` | `bool` | `false` | Enable StackerDB replication for signer support. Required for signer-connected nodes. |
| `mock_mining` | `bool` | `false` | Enable simulated mining for local testing. |
| `mine_microblocks` | `bool` | `true` | Enable microblock mining. *Deprecated: ignored in Epoch 2.5+.* |
| `microblock_frequency` | `u64` (ms) | `30000` | Microblock production interval. *Deprecated.* |
| `max_microblocks` | `u64` | `65535` | Maximum microblocks per anchor block. *Deprecated.* |
| `wait_time_for_microblocks` | `u64` (ms) | `30000` | Cooldown after microblock production. *Deprecated.* |
| `wait_time_for_blocks` | `u64` (ms) | `30000` | Max time to wait for block sync after new burnchain block. |
| `next_initiative_delay` | `u64` (ms) | `10000` | Nakamoto relayer idle polling interval. |
| `prometheus_bind` | `String` | `None` | Address for Prometheus metrics server (e.g., `"0.0.0.0:9153"`). |
| `marf_cache_strategy` | `String` | `None` (="noop") | MARF trie caching: `"noop"`, `"everything"`, or `"node256"`. |
| `marf_defer_hashing` | `bool` | `true` | Defer MARF hash computation to flush time. |
| `pox_sync_sample_secs` | `u64` (s) | `30` | PoX watchdog polling interval. *Deprecated in Epoch 3.0+.* |
| `use_test_genesis_chainstate` | `bool` | `None` | Use alternative test genesis. Disallowed on mainnet. |
| `chain_liveness_poll_time_secs` | `u64` (s) | `300` | Interval for chain liveness monitoring thread. |
| `stacker_dbs` | `[String]` | `[]` | Additional StackerDB contracts to replicate. Auto-populated for miners/stackers. |
| `txindex` | `bool` | `false` | Enable transaction index for tx-by-ID lookups. Uses significant disk space. |

---

## `[burnchain]` Section

Configures the Bitcoin connection and consensus parameters.

**Struct:** `BurnchainConfig` at `stackslib/src/config/mod.rs:1254`

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `chain` | `String` | `"bitcoin"` | The burnchain type. Only `"bitcoin"` is supported. |
| `mode` | `String` | `"mocknet"` | Network mode. Values: `"mainnet"`, `"xenon"`, `"mocknet"`, `"helium"`, `"neon"`, `"argon"`, `"krypton"`, `"nakamoto-neon"`. |
| `chain_id` | `u32` | Auto | Network chain ID. Mainnet: `0x00000001`, testnet: `0x80000000`. Do not modify. |
| `burn_fee_cap` | `u64` (sats) | `20000` | Maximum satoshis to spend on a block commit. Hot-reloadable. |
| `commit_anchor_block_within` | `u64` (ms) | `5000` | Delay after burnchain tip before mining. Testing only. |
| `peer_host` | `String` | `"0.0.0.0"` | Bitcoin node hostname/IP. |
| `peer_port` | `u16` | `8333` | Bitcoin P2P port. |
| `rpc_port` | `u16` | `8332` | Bitcoin RPC port. |
| `rpc_ssl` | `bool` | `false` | Use HTTPS for Bitcoin RPC. |
| `username` | `String` | `None` | Bitcoin RPC username. Required for miners. |
| `password` | `String` | `None` | Bitcoin RPC password. Required for miners. |
| `timeout` | `u64` (s) | `300` | Bitcoin RPC request timeout. |
| `socket_timeout` | `u64` (s) | `30` | Bitcoin socket operation timeout. |
| `magic_bytes` | `String` | `"X2"` | 2-char ASCII network identifier. Mainnet: `"X2"`, testnet: `"T2"`. |
| `local_mining_public_key` | `String` (hex) | `None` | Bitcoin regtest mining key. Mandatory for `"helium"` mode. |
| `process_exit_at_block_height` | `u64` | `None` | Exit node at this Bitcoin height. Testing only. |
| `poll_time_secs` | `u64` (s) | `10` | Bitcoin polling interval. Set to ~300 for mainnet. |
| `satoshis_per_byte` | `u64` | `50` | Default fee rate in sats/vByte for Bitcoin transactions. |
| `max_rbf` | `u64` (%) | `150` | Maximum RBF fee rate as percentage of original. |
| `leader_key_tx_estimated_size` | `u64` (vB) | `290` | Estimated size of leader key registration tx. |
| `block_commit_tx_estimated_size` | `u64` (vB) | `380` | Estimated size of block commit tx. |
| `rbf_fee_increment` | `u64` (sats/vB) | `5` | Fee increment per RBF attempt. |
| `wallet_name` | `String` | `""` | Bitcoin wallet name for multi-wallet nodes. |
| `max_unspent_utxos` | `u64` | `1024` | Max UTXOs to request from Bitcoin. Must be <= 1024. |

### Testing-Only Burnchain Fields

These fields are rejected on mainnet:

| Field | Type | Description |
|-------|------|-------------|
| `first_burn_block_height` | `u64` | Override starting Bitcoin height. |
| `first_burn_block_timestamp` | `u32` | Override starting block timestamp. |
| `first_burn_block_hash` | `String` | Override starting block hash. |
| `pox_prepare_length` | `u32` | Override PoX prepare phase length. |
| `pox_reward_length` | `u32` | Override PoX reward cycle length. |
| `pox_2_activation` | `u32` | Override PoX-2 activation height. |
| `sunset_start` | `u32` | PoX sunset start height. *Deprecated.* |
| `sunset_end` | `u32` | PoX sunset end height. *Deprecated.* |

### `[[burnchain.epochs]]` (Testing Only)

Custom epoch definitions for non-mainnet modes. Each entry requires `epoch_name` and `start_height`:

```toml
[[burnchain.epochs]]
epoch_name = "1.0"
start_height = 0

[[burnchain.epochs]]
epoch_name = "2.0"
start_height = 0

[[burnchain.epochs]]
epoch_name = "2.05"
start_height = 1

[[burnchain.epochs]]
epoch_name = "2.1"
start_height = 2

[[burnchain.epochs]]
epoch_name = "3.0"
start_height = 3
```

Valid epoch names: `"1.0"`, `"2.0"`, `"2.05"`, `"2.1"`, `"2.2"`, `"2.3"`, `"2.4"`, `"2.5"`, `"3.0"`, `"3.1"`, `"3.2"`, `"3.3"`.

**Rules** (enforced at `stackslib/src/config/mod.rs:683-818`):
- Must be in strict chronological order
- Epoch `"1.0"` must start at height 0
- Cannot skip epochs
- Start heights must be non-decreasing

---

## `[miner]` Section

Configures mining behavior. Only relevant when `node.miner = true`.

**Struct:** `MinerConfig` at `stackslib/src/config/mod.rs:2596`

### Core Mining Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mining_key` | `String` (hex) | From `node.seed` | Private key for block signing. Auto-derived from node seed when `[miner]` section is present. |
| `nakamoto_attempt_time_ms` | `u64` (ms) | `5000` | Max time for mempool walk when building a Nakamoto block. |
| `min_time_between_blocks_ms` | `u64` (ms) | `1000` | Minimum gap between blocks. Must be >= 1000 (signer requirement). |
| `empty_mempool_sleep_time` | `u64` (ms) | `2500` | Sleep duration between mining attempts when mempool is empty. |
| `segwit` | `bool` | `false` | Use p2wpkh address for Bitcoin transactions. |
| `max_execution_time_secs` | `u64` (s) | `30` | Timeout for individual transaction execution during block building. |
| `max_tenure_bytes` | `u64` (bytes) | `10485760` (10 MB) | Maximum bytes in a tenure. Mining stops if reached. |

### Mempool Walk Strategy

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mempool_walk_strategy` | `String` | `"NextNonceWithHighestFeeRate"` | `"GlobalFeeRate"` or `"NextNonceWithHighestFeeRate"`. |
| `probability_pick_no_estimate_tx` | `u8` (%) | `25` | Chance of picking txs without fee estimates (0-100). Only for `GlobalFeeRate`. |
| `txs_to_consider` | `String` | All types | Comma-separated: `"TokenTransfer"`, `"SmartContract"`, `"ContractCall"`. |
| `filter_origins` | `String` | `""` | Comma-separated Stacks addresses to whitelist. |
| `nonce_cache_size` | `usize` (bytes) | `1048576` | In-memory nonce cache size. |
| `candidate_retry_cache_size` | `usize` (items) | `1048576` | Retry cache for `GlobalFeeRate` strategy. |
| `tenure_cost_limit_per_block_percentage` | `u8` (%) | `25` | Max % of remaining tenure budget per block. |
| `contract_cost_limit_percentage` | `u8` (%) | `95` | % of block budget after which non-boot contract calls stop. |
| `log_skipped_transactions` | `bool` | `false` | Log skipped transactions. Testing use. |

### Signer Interaction

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `first_rejection_pause_ms` | `u64` (ms) | `5000` | Pause after first signer threshold rejection. |
| `subsequent_rejection_pause_ms` | `u64` (ms) | `10000` | Pause after subsequent rejections. |
| `block_commit_delay` | `u64` (ms) | `40000` | Wait time after burnchain block before submitting block commit. |
| `stackerdb_timeout` | `u64` (s) | `120` | Socket timeout for StackerDB communication. |

### Tenure Extension

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `tenure_extend_poll_timeout` | `u64` (s) | `1` | Polling interval for tenure extension checks. |
| `tenure_extend_wait_timeout` | `u64` (ms) | `120000` | Grace period before extending when next miner is inactive. |
| `tenure_timeout` | `u64` (s) | `180` | Time threshold for time-based tenure extends. |
| `tenure_extend_cost_threshold` | `u64` (%) | `50` | Minimum tenure budget usage before time-based extend. |
| `read_count_extend_cost_threshold` | `u64` (%) | `25` | Minimum read-count budget usage before extend. |

### Rejection Timeout Steps

Adaptive timeouts based on accumulated signer rejection weight:

```toml
[miner.block_rejection_timeout_steps]
"0" = 180     # No rejections: wait up to 180s
"10" = 90     # 10%+ rejection weight: wait 90s
"20" = 45     # 20%+ rejection weight: wait 45s
"30" = 0      # 30%+ rejection weight: give up immediately
```

### Legacy Fields (Pre-Nakamoto)

These are ignored in Epoch 3.0+ but remain for backward compatibility:

| Field | Default | Description |
|-------|---------|-------------|
| `first_attempt_time_ms` | `10` | First block mining attempt time. |
| `subsequent_attempt_time_ms` | `120000` | Subsequent mining attempt time. |
| `microblock_attempt_time_ms` | `30000` | Microblock mining attempt time. |
| `block_reward_recipient` | `None` | Custom coinbase reward address (Epoch 2.1+). |
| `min_tx_count` | `0` | Minimum txs for RBF replacement. |
| `only_increase_tx_count` | `false` | Require increasing tx count in retry. |
| `max_reorg_depth` | `3` | Reorg analysis depth. |
| `target_win_probability` | `0.0` | Underperformance detection target. |
| `underperform_stop_threshold` | `None` | Blocks before pausing on underperformance. |

---

## `[[events_observer]]` Section

Configure one or more webhook endpoints to receive node events. Each entry is an `EventObserverConfig`.

**Struct:** `EventObserverConfig` at `stackslib/src/config/mod.rs`

```toml
[[events_observer]]
endpoint = "localhost:3700"
events_keys = ["*"]
timeout_ms = 1000
disable_retries = false
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `endpoint` | `String` | Required | `HOST:PORT` of the observer HTTP server. |
| `events_keys` | `[String]` | Required | Event types to subscribe to (see below). |
| `timeout_ms` | `u64` | `1000` | HTTP request timeout for this observer. |
| `disable_retries` | `bool` | `false` | Skip retries on delivery failure. **Warning:** events may be lost. |

### Event Keys

| Key | Description |
|-----|-------------|
| `"*"` | All events. |
| `"stx"` | STX transfer, mint, burn, and lock events. |
| `"memtx"` | New mempool transactions and drops. |
| `"burn_blocks"` | New burnchain (Bitcoin) blocks. |
| `"microblocks"` | New microblocks (pre-Nakamoto). |
| `"stackerdb"` | StackerDB chunk updates. |
| `"block_proposal"` | Block proposal validation responses. |
| `"CONTRACT_ID::EVENT_NAME"` | Smart contract events (e.g., `"SP000...002Q6VF78.pox-4::handle-unlock"`). |
| `"ADDR.CONTRACT.ASSET"` | Asset-specific events (NFT/FT transfers). |

**Environment variable:** `STACKS_EVENT_OBSERVER=HOST:PORT` registers a catch-all observer (equivalent to `events_keys = ["*"]`).

See Appendix C for the full event dispatcher architecture.

---

## `[connection_options]` Section

P2P and HTTP networking parameters.

**Struct:** `ConnectionOptions` at `stackslib/src/net/connection.rs`

Key fields with their defaults from `HELIUM_DEFAULT_CONNECTION_OPTIONS` (`stackslib/src/config/mod.rs:143`):

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `inbox_maxlen` | `u64` | `100` | Max pending inbound messages per connection. |
| `outbox_maxlen` | `u64` | `100` | Max pending outbound messages per connection. |
| `timeout` | `u64` (s) | `15` | General connection timeout. |
| `idle_timeout` | `u64` (s) | `15` | Idle HTTP connection timeout. |
| `heartbeat` | `u32` (s) | `3600` | P2P heartbeat interval. |
| `num_neighbors` | `u64` | `32` | Number of neighbors to track inventories for. |
| `num_clients` | `u64` | `750` | Max inbound P2P connections. |
| `soft_num_neighbors` | `u64` | `16` | Soft limit on tracked neighbors. |
| `max_neighbors_per_host` | `u64` | `1` | Max neighbors per IP host. |
| `max_clients_per_host` | `u64` | `4` | Max inbound connections per IP host. |
| `max_http_clients` | `u64` | `1000` | Max HTTP API connections. |
| `walk_interval` | `u64` (s) | `60` | Neighbor walk frequency. |
| `inv_sync_interval` | `u64` (s) | `45` | Block inventory refresh interval. |
| `inv_reward_cycles` | `u64` | `3` | Reward cycles to look back for inventory sync (mainnet). |
| `download_interval` | `u64` (s) | `10` | Block download scan interval. |
| `dns_timeout` | `u64` (ms) | `15000` | DNS resolution timeout. |
| `max_inflight_blocks` | `u64` | `6` | Max concurrent block downloads. |
| `auth_token` | `String` | `None` | Shared secret for authenticated RPC endpoints (used by signers). |

---

## `[fee_estimation]` Section

Controls the transaction fee estimation subsystem.

**Struct:** `FeeEstimationConfig` at `stackslib/src/config/mod.rs:2227`

```toml
[fee_estimation]
cost_estimator = "naive_pessimistic"
fee_estimator = "scalar_fee_rate"
cost_metric = "proportion_dot_product"
log_error = false
fee_rate_fuzzer_fraction = 0.1
fee_rate_window_size = 5
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `disabled` | `bool` | `false` | Disable fee estimation entirely. |
| `cost_estimator` | `String` | `"naive_pessimistic"` | Cost estimation strategy. Only `"naive_pessimistic"` supported. |
| `fee_estimator` | `String` | `"scalar_fee_rate"` | Fee estimation strategy: `"scalar_fee_rate"` or `"fuzzed_weighted_median_fee_rate"`. |
| `cost_metric` | `String` | `"proportion_dot_product"` | Cost metric: only `"proportion_dot_product"` supported. |
| `log_error` | `bool` | `false` | Log cost estimation errors. |
| `fee_rate_fuzzer_fraction` | `f64` | `0.1` | Random noise fraction for fuzzed estimator (0.0-1.0). |
| `fee_rate_window_size` | `u64` | `5` | Window size for weighted median estimator. |

---

## `[atlas]` Section

Configures the Atlas attachment system for off-chain data (e.g., BNS name zonefile data).

**Struct:** `AtlasConfig` at `stackslib/src/net/atlas/mod.rs`

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `attachments_max_size` | `u32` | `1048576` | Maximum attachment size in bytes. |
| `max_uninstantiated_attachments` | `u32` | `10000` | Max unresolved attachment instances. |
| `uninstantiated_attachments_expire_after` | `u32` | `3600` | Expiry for unresolved attachments in seconds. |
| `unresolved_attachment_instances_expire_after` | `u32` | `172800` | Expiry for unresolved instances (48h default). |

---

## `[[ustx_balance]]` Section (Testing Only)

Pre-allocate STX balances at genesis for testing. Rejected on mainnet.

```toml
[[ustx_balance]]
address = "ST2QKZ4FKHAH1NQKYKYAYZPY440FEPK7GZ1R5HBP2"
amount = 10000000000000000
```

| Field | Type | Description |
|-------|------|-------------|
| `address` | `String` | Stacks address (testnet format: `ST...`). |
| `amount` | `u64` | Amount in microSTX (1 STX = 1,000,000 uSTX). |

---

## Runtime Config Reloading

Several configuration sections support hot-reloading from the config file without restarting the node:

- **`burnchain`** settings (via `Config::get_burnchain_config()` at line 420)
- **`miner`** settings (via `Config::get_miner_config()` at line 435)
- **`node`** settings (via `Config::get_node_config()` at line 448)

These methods re-read the config file from disk on each call, allowing operators to adjust parameters like `burn_fee_cap` or `satoshis_per_byte` while the node is running.
