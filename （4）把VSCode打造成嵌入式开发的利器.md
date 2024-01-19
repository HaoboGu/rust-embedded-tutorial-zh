## 必选插件
- rust analyzer
- cortex debug
## VSCode配置
### `.vscode/tasks.json`
```json
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558 
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            /*
             * This is the default cargo build task,
             * but we need to provide a label for it,
             * so we can invoke it from the debug launcher.
             */
            "label": "Cargo Build (debug)",
            "type": "process",
            "command": "cargo",
            "args": ["build"],
            "problemMatcher": [
                "$rustc"
            ],
            "group": {
                "kind": "build",
            }
        },
        {
            "label": "Cargo Build (release)",
            "type": "process",
            "command": "cargo",
            "args": ["build", "--release"],
            "problemMatcher": [
                "$rustc"
            ],
            "group": "build"
        },
        {
            "label": "Flash",
            "type": "shell",
            "command": "openocd -f ${workspaceFolder}/openocd.cfg -c \"program target/thumbv7em-none-eabihf/debug/rust-embedded-demo preverify verify reset exit\"",
            "dependsOn": [
                "Cargo Build (debug)"
            ],
            "group": "build"
        },
        {
            "label": "Cargo Clean",
            "type": "process",
            "command": "cargo",
            "args": ["clean"],
            "problemMatcher": [],
            "group": "build"
        },
    ]
}

```
### `.vscode/launch.json`
```json
{
    /* 
     * Requires the Rust Language Server (rust-analyzer) and Cortex-Debug extensions
     * https://marketplace.visualstudio.com/items?itemName=rust-lang.rust-analyzer
     * https://marketplace.visualstudio.com/items?itemName=marus25.cortex-debug
     */
    "version": "0.2.0",
    "configurations": [
        {
            /* Configuration for the STM32F303 Discovery board */
            "type": "cortex-debug",
            "request": "launch",
            "name": "Debug (OpenOCD)",
            "servertype": "openocd",
            "cwd": "${workspaceRoot}",
            "preLaunchTask": "Flash",
            "runToEntryPoint": "main",
            "executable": "./target/thumbv7em-none-eabihf/debug/rust-embedded-demo",
            "svdFile": "${workspaceRoot}/STM32H7B0x.svd",
            "configFiles": [
                "${workspaceRoot}/openocd.cfg"
            ],
            "rttConfig": {
                "enabled": true,
                "address": "auto",
                "clearSearch": false,
                "polling_interval": 20,
                "rtt_start_retry": 2000,
                "decoders": [
                    {
                        "label": "RTT channel 0",
                        "port": 0,
                        "type": "console"
                    }
                ]
            },
        }
    ]
}
```
## 工程配置
### `Cargo.toml`
```toml
[package]
authors = ["Haobo Gu <haobogu@outlook.com>"]
edition = "2018"
readme = "README.md"
name = "rust-embedded-demo"
version = "0.1.0"

[dependencies]
cortex-m = "0.7.7"
cortex-m-rt = "0.7.3"
panic-rtt-target = { version = "0.1.2", features = ["cortex-m"] }
rtt-target = "0.4.0"
stm32h7xx-hal = { version = "0.15.1", features = ["rt", "stm32h7b0"] }
log = "0.4.20"

# Uncomment for the panic example.
# panic-itm = "0.4.1"

# Uncomment for the allocator example.
# alloc-cortex-m = "0.4.0"

# Uncomment for the device example.
# Update `memory.x`, set target to `thumbv7em-none-eabihf` in `.cargo/config`,
# and then use `cargo build --examples device` to build it.
# [dependencies.stm32f3]
# features = ["stm32f303", "rt"]
# version = "0.7.1"

# this lets you use `cargo fix`!
[[bin]]
name = "rust-embedded-demo"
test = false
bench = false

[profile.release]
codegen-units = 1 # better optimizations
debug = true      # symbols are nice and they don't increase the size on Flash
lto = true        # better optimizations
```
## 代码
### `main.rs`
```rust
#![no_main]
#![no_std]

mod rtt_logger;

use cortex_m_rt::entry;
use log::{info, warn, error};
use panic_rtt_target as _;
use stm32h7xx_hal::{pac, prelude::*};

#[entry]
fn main() -> ! {
    rtt_logger::init(log::LevelFilter::Debug);
    // 获取cortex核心外设和stm32h7的所有外设
    let cp = cortex_m::Peripherals::take().unwrap();
    let dp = pac::Peripherals::take().unwrap();

    // Power 设置
    let pwr = dp.PWR.constrain();
    let pwrcfg = pwr.freeze();

    // 初始化RCC
    let rcc = dp.RCC.constrain();
    let ccdr = rcc.sys_ck(200.MHz()).freeze(pwrcfg, &dp.SYSCFG);

    // 设置LED对应的GPIO
    let gpioe = dp.GPIOE.split(ccdr.peripheral.GPIOE);
    let mut led = gpioe.pe3.into_push_pull_output();

    // cortex-m已经实现好了delay函数，直接拿到，下面使用
    let mut delay = cp.SYST.delay(ccdr.clocks);

    let mut cnt: u32 = 0;
    loop {
        // 点灯
        if cnt == 1 {
            info!("current cnt: {}", cnt);
        } else if cnt == 2 {
            warn!("current cnt: {}", cnt);
        } else if cnt == 3 {
            error!("current cnt: {}", cnt);
        } else if cnt > 100000 {
            panic!("panicked!")
        } else {
            debug!("current cnt: {}", cnt);
        }
        cnt += 1;
        led.toggle();
        // 延时500ms
        delay.delay_ms(500_u16);
    }
}

```
### `rtt_logger.rs`
```rust
use panic_rtt_target as _;

use log::{Level, LevelFilter, Metadata, Record};
use rtt_target::{rprintln, rtt_init_print};

pub struct Logger {
    level: Level,
}

static LOGGER: Logger = Logger { level: Level::Info };

pub fn init(level: LevelFilter) {
    rtt_init_print!();
    log::set_logger(&LOGGER)
        .map(|()| log::set_max_level(level))
        .unwrap();
}

impl log::Log for Logger {
    fn enabled(&self, metadata: &Metadata) -> bool {
        metadata.level() <= self.level
    }

    fn log(&self, record: &Record) {
        let prefix = match record.level() {
            Level::Error => "\x1B[1;31m",
            Level::Warn => "\x1B[1;33m",
            Level::Info => "\x1B[1;32m",
            Level::Debug => "\x1B[1;30m",
            Level::Trace => "\x1B[2;30m",
        };

        rprintln!("{}{} - {}\x1B[0m", prefix, record.level(), record.args());
    }

    fn flush(&self) {}
}

```
