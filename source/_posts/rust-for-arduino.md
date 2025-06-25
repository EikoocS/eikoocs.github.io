---
title: 使用 Rust 编写 Arduino Uno 程序
date: 2023-11-05 15:09:14
tags: rust
---

# 使用 Rust 编写 Arduino Uno 程序

## 零、测试环境

使用的环境为

- Windows11 （理论Linux、Mac同理）
- rustc 1.75.0-nightly  （必须为 nightly 版本）
- Arduino Uno R3

测试时间为 2023/11/5 ，如果距离你阅读此文章的时间较远，则实际操作可能需要改变，注意相关文档

因测试环境局限，文章中未明确指出的命令均为在 Windows Cmd 上测试过，其他系统道理相同

## 一、环境配置

在这一步我们需要安装 avr-gcc （链接器）和 avrdude（上传固件），并配置rust项目

### 安装 avr-gcc / avrdude

#### Linux

Linux可以直接使用包管理器安装，包名一般为 avr-gcc （或 gcc-avr ）和 ravedude，具体视发行版的不同而不同

#### Windows

#####　通过 msys2 安装：

```shell
pacman -S mingw-w64-x86_64-avr-gcc
```

##### 通过 scoop 安装：

安装 scoop （已经安装了 scoop 则可以跳过此步骤）

```shell
# 直接安装
irm get.scoop.sh | iex
# 通过指定的代理安装 （适用于无法直接访问的情况）
irm get.scoop.sh -Proxy 'http://<ip:port>' | iex
```

安装 avr-gcc / avrdude

```shell
scoop config proxy <ip:port> # （可选）为 scoop 设置代理
scoop install avr-gcc
scoop install avrdude
```

#### Mac

```shell
xcode-select --install # 如果你还没有这样做
brew tap osx-cross/avr
brew install avr-gcc avrdude
```

### 配置 Rust 项目

新建项目

```shell
cargo new rust-arduino-test
```

修改 `cargo.toml`

```toml
# 包配置自动生成，无需在意是否与此相同
[package]
name = "rust-arduino-test"
version = "0.1.0"
edition = "2021"

# 需要一个恐慌处理器
# 包含以下选择，在此使用最简单的一个
# panic-halt：恐慌会导致程序或当前线程通过进入无限循环而暂停
# panic-abort：恐慌会导致执行中止指令
# panic-itm：恐慌消息将被使用ITM（ARM Cortex-M特定的外围设备）记录
# panic-semihosting：恐慌消息使用半主机技术记录到主机
[dependencies]
panic-halt = "0.2.0"

# 引入 avr-hal/arduino-hal , 并启用 arduino-uno 特性
[dependencies.arduino-hal]
git = "https://github.com/Rahix/avr-hal"
features = ["arduino-uno"]

```

在项目根目录下创建一个文件 `avr-atmega328p.json` 用于为 AVR 平台设置编译元数据

```json
{
  "llvm-target": "avr-unknown-unknown",
  "cpu": "atmega328p",
  "target-endian": "little",
  "target-pointer-width": "16",
  "target-c-int-width": "16",
  "os": "unknown",
  "target-env": "",
  "target-vendor": "unknown",
  "arch": "avr",
  "data-layout": "e-P1-p:16:8-i8:8-i16:8-i32:8-i64:8-f32:8-f64:8-n8-a:8",

  "executables": true,

  "linker": "avr-gcc",
  "linker-flavor": "gcc",
  "pre-link-args": {
    "gcc": ["-Os", "-mmcu=atmega328p"]
  },
  "exe-suffix": ".elf",
  "post-link-args": {
    "gcc": ["-Wl,--gc-sections"]
  },

  "singlethread": false,
  "no-builtins": false,

  "no-default-libraries": false,

  "eh-frame-header": false
}
```

修改项目根目录下 `.cargo/config.toml` 如果不存在则创建

```toml
[build]
target = "avr-atmega328p.json"

[unstable]
build-std = ["core"]
```

**至此，基础的环境配置已经完成，下面将编写用于测试的代码**

## 二、测试代码

修改 `main.rs` 

```rust
#![no_std] // 嵌入式环境并不包含全部标准库，故标记 no_std
#![no_main] // 不使用默认的入口，因为 arduino_hal 将提供宏来访问入口

use panic_halt as _; // 引用恐慌处理器，如果在上文中选择了不同的恐慌处理器，则应该对应修改

#[arduino_hal::entry] // 标记此函数为入口
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

  // 代码作用 使接入到 13 号端口的灯闪烁
    let mut led = pins.d13.into_output();
    
    loop {
        led.toggle();
        arduino_hal::delay_ms(1000);
    }
}
```

使用编译 `cargo` 编译程序

```shell
cargo build
```

若一切正常，则可以得到位于 `target/avr-atmega328p/debug/` 目录下的 `rust-arduino-test.elf` 文件

## 三、上传固件

这里需要使用之前安装的 avrdude 命令

```shell
avrdude^
    -C path\to\avrdude.conf ^
    -p atmega328p ^
    -c arduino ^
    -P COM3 ^
    -D ^
    -U flash:w:target\avr-atmega328p\debug\rust-arduino-test.elf:e
```

逐行解析这条命令 **粗体部分为必须阅读，可能根据环境不同需要手动调整的内容**

- `-C` 指定配置文件的位置，**如果使用 `scoop ` 安装，则位于`{用户目录}\scoop\apps\avrdude\{版本号}\avrdude.conf`，其他安装方法和其他系统可以通过 Everything 等工具搜索 `avrdude.conf` 来找到**
- `-p` 指定 AVR 设备，在这里是 `atmega328p`
- `-c` 指定编程器类型，在这里为 `arduino` 
- `-P` 指定链接端口，**可以通过 `设备管理器>端口（COM 和 LPT）> Arduino Uno` 在其后括号中找到**
- `-D` 禁用闪存的自动擦除功能
- `-U` 执行的内存操作，格式为 `<memtype>:r|w|v:<filename>[:format]` 在这里的效果为指定操作目标为缓存（`flash`），读取指定文件（`target\avr-atmega328p\debug\rust-arduino-test.elf`）并将其写入指定的目标（`w`），文件格式为 ELF 格式（`e`）

运行此命令后直至出现 `avrdude done.` 则上传成功，通过在 13 号端口接入一个 LED 可以检查效果

## 附、参考资料

[如何在 Arduino Uno 中运行 Rust：让你的 LED 灯闪起来 -- n3xtchen](https://n3xtchen.github.io/n3xtchen/rust/2020/08/22/rust-arduino-our-first-blink/)

[Rahix/avr-hal GitHub](https://github.com/Rahix/avr-hal)

[avrdude#Option-Descriptions](https://avrdudes.github.io/avrdude/7.2/avrdude_3.html#Option-Descriptions)
