## 相关链接
- embedded-hal 文档：[https://docs.rs/embedded-hal/latest/embedded_hal](https://docs.rs/embedded-hal/latest/embedded_hal/)
- embedded-hal 代码仓库：[https://github.com/rust-embedded/embedded-hal](https://github.com/rust-embedded/embedded-hal)

## 示例代码
```rust
#![no_main]
#![no_std]

mod rtt_logger;

use cortex_m_rt::entry;
use log::{debug, error, info, warn};
use panic_rtt_target as _;
use stm32h7xx_hal::{
    hal::digital::v2::{InputPin, OutputPin},
    pac,
    prelude::*,
};

fn control_led<In: InputPin<Error = E>, Out: OutputPin<Error = E>, E>(
    input_pin: &In,
    led_pin: &mut Out,
) -> Result<(), E> {
    if input_pin.is_high()? {
        info!("high");
        led_pin.set_high()?;
    } else {
        info!("low");
        led_pin.set_low()?;
    }
    Ok(())
}

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

    let gpioc = dp.GPIOC.split(ccdr.peripheral.GPIOC);
    let key = gpioc.pc13.into_pull_down_input();


    // cortex-m已经实现好了delay函数，直接拿到，下面使用
    let mut delay = cp.SYST.delay(ccdr.clocks);

    loop {
        // 点灯
        control_led(&key, &mut led).unwrap();
        // 延时50ms
        delay.delay_ms(50_u16);
    }
}

```
