# 构建 riscv-gnu-toolchain

## 一次性构建
```
docker build -f Dockerfile-riscv-gnu-toolchain --tag riscv-gnu-toolchain .
```

## 交互式构建
```
docker volume create riscv-build
docker run --rm -it -v riscv-build:/build archlinux:latest bash

pacman -Syu --noconfirm git autoconf automake curl python3 libmpc mpfr gmp gawk base-devel bison flex texinfo gperf libtool patchutils bc zlib expat
git clone https://github.com/riscv-collab/riscv-gnu-toolchain /build/riscv
cd /build/riscv
./configure --prefix=/build/opt/riscv
make

docker run -it \
 -v riscv-build:/build \
 -e PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/riscv/bin \
 --name riscv-gnu-toolchain \
 archlinux:latest
cp -r /build/opt/riscv /riscv
exit
docker commit riscv-gnu-toolchain riscv-gnu-toolchain:latest
docker rm riscv-gnu-toolchain
```

# 构建 ckb-debugger

下载所需的源代码：
```
mkdir build
cd build
git clone https://github.com/nervosnetwork/ckb-binary-patcher.git
git clone https://github.com/nervosnetwork/ckb-standalone-debugger.git
```
需要的话可以对下载的代码做一些修改，比如我希望输出的调试信息简洁一些，就可以把
[这一行](https://github.com/nervosnetwork/ckb-standalone-debugger/blob/eaeb6128837cc3103dbaa5eb61a1f49304935e5a/bins/src/main.rs#L267)
修改为：
```
    debug!("{}", message);
```

构建 docker image：
```
docker build -f Dockerfile-ckb-debugger --tag ckb-debugger:latest .
```

构建得到的 image 中主要相关的目录布局：
```
/
├── riscv
│   ├── bin
│   ├── include
│   ├── lib
│   ├── libexec
│   ├── riscv64-unknown-elf
│   └── share
├── root
│   ├── .cargo
│   │   ├── bin
│   │   └── ...
│   └── .rustup
│       ├── toolchains
│       └── ...
```

# 使用 ckb-debugger

可以用于直接执行 tx 计算，需要配合 `ckb-transaction-dumper` 来构建测试用的数据 `.tmp/debug.json`：
```
docker run --rm -it -v `pwd`:/code -w /code -e RUST_LOG=debug ckb-debugger:latest \
ckb-debugger \
--cell-type input \
--cell-index 0 \
--script-group-type lock \
--tx-file .tmp/debug.json
```

可以通过 `capsule debugger` 来使用（方便之处在于可以从 `build/debug.json` 自动生成
`.tmp/debug.json`，把 script code 的二进制代码填充进去）：
```
# 用 capsule debugger 启动调试（ckb-debugger + gdb）
capsule --env-file env-for-docker debugger start \
--name test-lock \
--template-file build/debug.json \
--cell-type input \
--cell-index 0 \
--script-group-type lock \
--listen 8000

# 用 capsule debugger 执行一遍
capsule --env-file env-for-docker debugger start \
--name test-lock \
--template-file build/debug.json \
--cell-type input \
--cell-index 0 \
--script-group-type lock \
--only-server
```
其中的 `env-for-docker` 文件里面可以包含 `RUST_LOG=debug` 之类的内容，用于影响
`ckb-debugger` 里面的 `env_logger`。

**注意：为了让这里构建出来的 image 能够用于 `capsule debugger` 命令，需要一些准备工作：**
```
# 给它一个特定的 tag
docker image tag ckb-debugger:latest thewawar/ckb-capsule:2021-12-25

# 如果之前使用过 capsule debugger 命令，会产生一个 volume，须删除：
docker volume rm capsule-cache
```

# 参考资料

[RISC-V GNU Compiler Toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain)
| [Arch Linux](https://archlinux.org/)

[ckb-standalone-debugger](https://github.com/nervosnetwork/ckb-standalone-debugger)

[thewawar/ckb-capsule](https://hub.docker.com/layers/thewawar/ckb-capsule/2021-12-25/images/sha256-fa5449c3e08316305518cfe757210b7f967be0bcf455db11c6bc749ce7e6465d?context=explore)
| [Capsule](https://github.com/nervosnetwork/capsule)

[ckb-transaction-dumper](https://github.com/nervosnetwork/ckb-transaction-dumper)
