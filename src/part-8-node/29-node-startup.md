# Chapter 29: Node Startup

## Purpose

The `stacks-node` binary is the main entry point for running a Stacks blockchain node. This chapter traces the startup sequence from `main()` through configuration parsing, run loop selection, chainstate initialization, and the transition from Epoch 2.x (Neon) to Epoch 3.x (Nakamoto) operation. Understanding this flow is essential for anyone debugging startup issues, adding new CLI commands, or modifying the boot sequence.

## Key Concepts

### Run Loop Hierarchy

The node has three run loop implementations, reflecting the evolution of the Stacks protocol:

1. **Helium** (`run_loop/helium.rs`) -- For local development with `mocknet` or `helium` modes. Simulates a burnchain locally.
2. **Neon** (`run_loop/neon.rs`) -- For Epoch 2.x operation on real Bitcoin networks. Handles PoX, mining, and block processing.
3. **Nakamoto** (`run_loop/nakamoto.rs`) -- For Epoch 3.x operation with signer-based consensus.

A fourth struct, `BootRunLoop` (`run_loop/boot_nakamoto.rs`), orchestrates the transition: it starts with Neon and switches to Nakamoto when the Epoch 3.0 boundary is reached.

### Configuration

Configuration flows through two structs:
- `ConfigFile` -- The raw TOML deserialization target. Can also be constructed from preset profiles (`mocknet()`, `helium()`, `xenon()`, `mainnet()`).
- `Config` -- The validated, processed configuration with computed paths, parsed epochs, and burnchain parameters.

## Architecture Overview

```
main()
  |
  +-- Parse CLI args (pico_args)
  |
  +-- Select config source:
  |     "mocknet"  -> ConfigFile::mocknet()
  |     "helium"   -> ConfigFile::helium()
  |     "testnet"  -> ConfigFile::xenon()
  |     "mainnet"  -> ConfigFile::mainnet()
  |     "start"    -> ConfigFile::from_path(config_path)
  |
  +-- Config::from_config_file(config_file)
  |
  +-- Select run loop based on burnchain.mode:
        "helium" | "mocknet"  -> helium::RunLoop
        "neon" | "nakamoto-neon" | "xenon" | "krypton" | "mainnet"
                              -> boot_nakamoto::BootRunLoop
```

## Code Walkthrough: main()

The entry point is `stacks-node/src/main.rs:252`. The function does several things before entering the run loop:

### Panic Handler (line 253)

```rust
panic::set_hook(Box::new(|panic_info| {
    error!("Process abort due to thread panic: {panic_info}");
    let bt = Backtrace::new();
    error!("Panic backtrace: {bt:?}");
    // On unix: send SIGQUIT to trigger core dump
    unsafe { kill(pid, SIGQUIT) };
    process::exit(1);
}));
```

This custom panic handler ensures that any thread panic produces a full backtrace in the logs and attempts to generate a core dump (if `ulimit -c unlimited` is set) before exiting. This is critical for post-mortem debugging of production crashes.

### Memory Allocator

On supported platforms (not macOS, Windows, or ARM), the node uses jemalloc instead of the system allocator:

```rust
#[cfg(not(any(target_os = "macos", target_os = "windows", target_arch = "arm")))]
#[global_allocator]
static GLOBAL: Jemalloc = Jemalloc;
```

### CLI Subcommands (line 287)

The node supports several subcommands beyond just running:

| Subcommand | Description |
|-----------|-------------|
| `mocknet` | Local development with simulated burnchain |
| `helium` | Local development with bitcoind regtest |
| `testnet` | Join the public testnet |
| `mainnet` | Join mainnet |
| `start --config <path>` | Start with a custom config file |
| `check-config --config <path>` | Validate config without starting |
| `version` | Print version string |
| `key-for-seed` | Derive WIF/hex secret key from a seed |
| `pick-best-tip` | Analyze chain tips and pick the best one |
| `get-spend-amount` | Calculate mining spend for a given height |

The `pick-best-tip` and `get-spend-amount` commands are offline analysis tools that open the databases directly without starting the node.

### Run Loop Selection (line 416)

After configuration is parsed, the burnchain mode determines which run loop to use:

```rust
if conf.burnchain.mode == "helium" || conf.burnchain.mode == "mocknet" {
    let mut run_loop = helium::RunLoop::new(conf);
    run_loop.start(num_round);
} else if conf.burnchain.mode == "neon"
    || conf.burnchain.mode == "nakamoto-neon"
    || conf.burnchain.mode == "xenon"
    || conf.burnchain.mode == "krypton"
    || conf.burnchain.mode == "mainnet"
{
    let mut run_loop = boot_nakamoto::BootRunLoop::new(conf).unwrap();
    run_loop.start(None, 0);
}
```

For all production modes, `BootRunLoop` is used.

## Code Walkthrough: BootRunLoop

The `BootRunLoop` (`stacks-node/src/run_loop/boot_nakamoto.rs:66`) manages the Neon-to-Nakamoto transition:

```rust
pub struct BootRunLoop {
    config: Config,
    active_loop: InnerLoops,
    coordinator_channels: Arc<Mutex<CoordinatorChannels>>,
}

enum InnerLoops {
    Epoch2(NeonRunLoop),
    Epoch3(NakaRunLoop),
}
```

### Construction (line 78)

```rust
pub fn new(config: Config) -> Result<Self, String> {
    let (coordinator_channels, active_loop) = if !Self::reached_epoch_30_transition(&config)? {
        let neon = NeonRunLoop::new(config.clone());
        (neon.get_coordinator_channel().unwrap(), InnerLoops::Epoch2(neon))
    } else {
        let naka = NakaRunLoop::new(config.clone(), None, None, None);
        (naka.get_coordinator_channel().unwrap(), InnerLoops::Epoch3(naka))
    };
    // ...
}
```

The constructor checks the current burn height against the Epoch 3.0 start height. If the node has already passed the transition point (e.g., syncing from scratch on a chain that's already in Nakamoto), it starts directly in Nakamoto mode. Otherwise, it starts in Neon mode.

### Epoch Transition Detection

`reached_epoch_30_transition()` (line 233) opens the SortitionDB (if it exists) and checks:

```rust
fn reached_epoch_30_transition(config: &Config) -> Result<bool, String> {
    let burn_height = Self::get_burn_height(config);
    let epochs = config.burnchain.get_epoch_list();
    let epoch_3 = epochs.get(StacksEpochId::Epoch30)?;
    Ok(u64::from(burn_height) >= epoch_3.start_height - 1)
}
```

If the SortitionDB doesn't exist yet, `get_burn_height` returns 0, so the node starts in Neon mode by default.

### The Transition (line 155)

When starting from Neon, `start_from_neon()`:

1. Spawns an `epoch-2/3-boot` monitor thread that polls `reached_epoch_30_transition()` every second
2. Starts the Neon run loop (which will run until stopped)
3. When the monitor thread detects epoch 3.0, it sets the termination flag to `false` (telling Neon to stop)
4. The Neon loop exits and returns `Neon2NakaData` containing the leader key state and peer network
5. A new `NakaRunLoop` is created with the inherited state
6. The coordinator channel reference is updated (important: the `Arc<Mutex<CoordinatorChannels>>` lets external code seamlessly follow the transition)
7. The Nakamoto run loop starts

The `Neon2NakaData` struct (`boot_nakamoto.rs:38`) carries state across the transition:

```rust
pub struct Neon2NakaData {
    pub leader_key_registration_state: LeaderKeyRegistrationState,
    pub peer_network: Option<PeerNetwork>,
}
```

This ensures the node doesn't lose its VRF key registration or have to rebuild the entire peer network when transitioning.

## Code Walkthrough: Nakamoto RunLoop

The `NakaRunLoop` (`stacks-node/src/run_loop/nakamoto.rs:60`) is the primary run loop for Epoch 3.x:

```rust
pub struct RunLoop {
    config: Config,
    globals: Option<Globals>,
    counters: Counters,
    coordinator_channels: Option<(CoordinatorReceivers, CoordinatorChannels)>,
    should_keep_running: Arc<AtomicBool>,
    event_dispatcher: EventDispatcher,
    pox_watchdog: Option<PoxSyncWatchdog>,
    is_miner: Option<bool>,
    burnchain: Option<Burnchain>,
    pox_watchdog_comms: PoxSyncWatchdogComms,
    miner_status: Arc<Mutex<MinerStatus>>,
    monitoring_thread: Option<JoinHandle<Result<(), MonitoringError>>>,
}
```

### Miner Detection (line 167)

The `check_is_miner()` method determines whether this node should mine:

1. If `config.node.miner` is false, the node runs as a follower
2. If `mock_mining` is true, skip UTXO checks
3. Otherwise, derive the miner's Bitcoin addresses (both legacy and segwit)
4. Check for UTXOs at those addresses via the Bitcoin RPC, retrying up to 6 times with 10-second intervals
5. If no UTXOs found after all retries, panic -- a miner without funds cannot operate

### Chainstate Boot (line 226)

`boot_chainstate()` initializes the Stacks chain state:

1. Loads genesis balances from config
2. Creates a `ChainStateBootData` with initial balances, lockups, namespaces, and names
3. Opens the chainstate database with `StacksChainState::open_and_exec()`
4. Announces boot receipts to the event dispatcher

### Coordinator Thread Spawn (line 277)

`spawn_chains_coordinator()` starts the chains coordinator in its own thread. The coordinator processes new Bitcoin blocks and sorts them, triggers block processing, and manages the Atlas attachment database.

### The start() Method

The `start()` method on `NakaRunLoop` performs the full boot sequence:

1. Initialize the burnchain connection
2. Check if we're a miner
3. Boot the chainstate
4. Start the chains coordinator thread
5. Start the PoX sync watchdog
6. Create `Globals` (shared inter-thread state)
7. Spawn the `StacksNode` (which creates the relayer and p2p threads)
8. Enter the main event loop

## Initial Block Download (IBD)

During IBD, the node is catching up to the chain tip. The `PoxSyncWatchdog` communicates IBD status through `PoxSyncWatchdogComms`. While in IBD:

- The miner is blocked from mining (would produce invalid blocks)
- The p2p thread focuses on block download
- The relayer processes blocks but doesn't initiate new tenures

IBD ends when the node's burnchain view is close enough to the Bitcoin tip.

## How It Connects

- **Chapter 30 (Thread Architecture)**: The `Globals`, relayer, and p2p threads spawned during startup are detailed there.
- **Chapter 31 (Bitcoin Controller)**: The `BitcoinRegtestController` created during startup manages the Bitcoin connection.
- **Chapter 10 (Bitcoin Integration)**: The burnchain configuration parsed here determines how the node talks to Bitcoin.
- **Chapter 20 (Coordinator)**: The chains coordinator thread spawned during startup is the main block processing engine.

## Edge Cases and Gotchas

1. **Epoch transition is one-way**: Once the node transitions from Neon to Nakamoto, it cannot go back. The `BootRunLoop` replaces its `active_loop` permanently.

2. **Coordinator channel swap**: External code (like test frameworks) may hold an `Arc<Mutex<CoordinatorChannels>>` from `BootRunLoop::coordinator_channels()`. When the transition happens, the mutex-guarded channel is swapped in-place so existing holders automatically use the new coordinator.

3. **SortitionDB creation on open**: The `get_burn_height()` method explicitly checks if the SortitionDB file exists before trying to open it, because `SortitionDB::open()` creates the file even if it can't initialize the tables, which would break subsequent `connect()` calls.

4. **Jemalloc platform exclusion**: macOS, Windows, and ARM targets use the system allocator. This is because jemalloc may not compile or perform well on those platforms. Production Linux nodes benefit from jemalloc's reduced fragmentation.

5. **Core dump on panic**: The custom panic handler sends SIGQUIT to the process to trigger a core dump. This only works if the operator has set `ulimit -c unlimited`. The `process::exit(1)` is a fallback in case SIGQUIT doesn't terminate the process.

6. **Multiple burnchain modes map to the same run loop**: `neon`, `nakamoto-neon`, `xenon`, `krypton`, and `mainnet` all use `BootRunLoop`. The mode string primarily affects default configuration values, not the run loop architecture.
