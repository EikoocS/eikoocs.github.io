---
title: Boost C++ Libraries 安装指南
date: 2025-06-25 16:20:00
tags: [cpp, boost]
---

## 测试环境

文章撰写时使用以下环境完成，版本差异过大可能导致未知的问题，请注意分辨。

- Windows 11
- Ubuntu 24.04.1
- Boost 1.87.0
- CMake 3.30.3
- MinGW-W64 5.0.0

## 安装

### Windows

> Windows 下配合 MSVC 使用 Boost 时可以使用预构建在 [SourceForge](https://sourceforge.net/projects/boost/files/boost-binaries/) 上的 Boost。请注意选择 MSVC 版本和系统位数，在网页中还有 Visual Studio 对应的 Boost 的版本以供参考。

如果有特殊需要，比如使用 MinGW 而不是 MSVC，那就需要自己手动编译构建：

> Cygwin 和 MinGW 用户须知
>
> 如果您打算在 Windows 命令提示符下使用工具，那您就找对地方了。如果您计划使用 Cygwin bash shell 进行构建，那么您实际上是在 POSIX 平台上运行，应遵循 Unix 变体的入门说明。其他命令 shell（如 MinGW 的 MSYS）不受支持，它们可能工作，也可能不工作。
>
> —— 节选自 Boost 官网

1. 在 [Boost Download](https://www.boost.org/users/download/) 中找到最新版本的 windows 平台，下载 .7z 或 .zip 均可。将其解压到一个合适的文件夹中待用。

2. 运行目录下的 `bootstrap.bat` ，运行结束后目录中会生成一个 b2.exe 文件。

3. 在该目录打开控制台，执行 `b2 toolset=gcc address-model=64 -j8`
   - toolset 选择工具链为 gcc ，有需要也可以指定为 msvc
   - address-model 指定编译为 64 位版本
   - j 指定并行编译数，可以更好的利用多核 CPU
   - 如果有需要，还可以通过 `--prefix="<路径>"` 来指定 boost 的构建路径

4. 设置环境变量 `BOOST_ROOT` 为构建目录

### Linux

> Linux 下优先考虑使用包管理器安装 boost，如 Ubuntu 可以使用 `sudo apt-get install libboost-all-dev` 来轻松安装 Boost。

若有特殊需求，包管理器满足不了需求，也可以编译安装。

```shell title = "install_boost.sh"
#!/bin/bash

# 设置Boost版本
BOOST_VERSION=1.83.0
BOOST_DIR=boost_$(echo \$BOOST_VERSION | tr '.' '_')
INSTALL_DIR=/usr/local  # 修改为你的安装路径

# 下载Boost源码
echo "Downloading Boost version \$BOOST_VERSION..."
wget https://boostorg.jfrog.io/artifactory/main/release/\$BOOST_VERSION/source/\${BOOST_DIR}.tar.gz

# 解压文件
echo "Extracting Boost..."
tar -xvzf \${BOOST_DIR}.tar.gz
cd \${BOOST_DIR}

# 准备Boost Build工具
echo "Bootstrapping Boost Build system..."
./bootstrap.sh

# 编译并安装
echo "Installing Boost to \$INSTALL_DIR..."
./b2 install --prefix=\$INSTALL_DIR

# 配置环境变量（仅在非默认路径时）
if [ "\$INSTALL_DIR" != "/usr/local" ]; then
  echo "Configuring environment variables for custom install directory..."
  echo "export LD_LIBRARY_PATH=\$INSTALL_DIR/lib:\$LD_LIBRARY_PATH" >> ~/.bashrc
  echo "export BOOST_ROOT=\$INSTALL_DIR" >> ~/.bashrc
  echo "export BOOST_INCLUDEDIR=\$INSTALL_DIR/include" >> ~/.bashrc
  echo "export BOOST_LIBRARYDIR=\$INSTALL_DIR/lib" >> ~/.bashrc
  source ~/.bashrc
fi

echo "Boost \$BOOST_VERSION installed successfully!"

```

然后执行以上脚本

```shell
# 赋予脚本执行权限
chmod +x install_boost.sh

# 执行脚本
./install_boost.sh
```

文章中的版本为 2025-1-5 时的最新版本，安装时请将 `BOOST_VERSION` 变量设置为自己想安装的版本。

## 测试

编写测试代码保存为 `main.cpp`

```cpp
#include <iostream>
#include <boost/array.hpp>

using namespace std;

int main()
{
    boost::array arr = {{1, 2, 3, 4}};
    std::cout << "hi " << arr[0] << std::endl;

    return 0;
}
```

编写 `CMakeLists.txt`

```CMake
cmake_minimum_required(VERSION 3.28)
project(boost_demo)

set(CMAKE_CXX_STANDARD 20)

find_package(Boost REQUIRED)

add_executable(boost_demo main.cpp)

target_include_directories(boost_demo PRIVATE ${Boost_INCLUDE_DIRS})
target_link_libraries(boost_demo PRIVATE ${Boost_LIBRARIES})
```

执行命令

```shell
mkdir build
cd build
cmake ..
# Windows MinGW 环境
# cmake -G "MinGW Makefiles" ..
make
./boost_demo
```

期望得到输出

```null
hi 1
```

Boost C++ Libraries 安装成功
