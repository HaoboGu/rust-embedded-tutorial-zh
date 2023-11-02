## 从模板创建Rust嵌入式工程
```bash
cargo install cargo-generate
cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
```
## 修改`Cargo.toml`，添加 hal 依赖
```toml
cortex-m = "0.7.7"
cortex-m-rt = "0.7.3"
panic-halt = "0.2.0"
stm32h7xx-hal = { version = "0.15.0", features = ["stm32h7b0", "rt"] }
```
## 修改`.cargo/config.toml`
```toml
[build]
# Pick ONE of these default compilation targets
# target = "thumbv6m-none-eabi"        # Cortex-M0 and Cortex-M0+
# target = "thumbv7m-none-eabi"        # Cortex-M3
# target = "thumbv7em-none-eabi"       # Cortex-M4 and Cortex-M7 (no FPU)
target = "thumbv7em-none-eabihf"     # Cortex-M4F and Cortex-M7F (with FPU)
# target = "thumbv8m.base-none-eabi"   # Cortex-M23
# target = "thumbv8m.main-none-eabi"   # Cortex-M33 (no FPU)
# target = "thumbv8m.main-none-eabihf" # Cortex-M33 (with FPU)
```
## 修改`main.rs`
```rust
#![no_main]
#![no_std]

use cortex_m_rt::entry;
use panic_halt as _;
use stm32h7xx_hal::{pac, prelude::*};

#[entry]
fn main() -> ! {
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

    loop {
        // 点灯
        led.toggle();
        // 延时500ms
        delay.delay_ms(500_u16);
    }
}

```
## 构建
```rust
cargo build
```
## 烧录
首先修改openocd.cfg
```bash
source [find interface/cmsis-dap.cfg]

source [find target/stm32h7x.cfg]

```
```bash
openocd -f openocd.cfg -c "program target/thumbv7em-none-eabihf/debug/cargo-embedded-demo preverify verify reset exit 0x08000000"
```


## 

