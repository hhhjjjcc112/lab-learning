首先肯定是clone taic的代码库
```bash
git clone git@github.com:taic-repo/taic.git
cd taic
```
然后下载所有子模块
```bash
git submodule update --init --recursive
```

看了一下，完全没有运行这个项目的说明，所以只能靠自己摸索了。
makefile里定义了如下命令
```makefile
build
build_payload
upload_payload
run_on_qemu
build_fpga
upload_fpga
clean_test
```
能大致感受到3个部分：
1. build build_payload upload_payload：编译测试程序并生成镜像
2. run_on_qemu：在qemu上运行
3. build_fpga upload_fpga：编译并上传到fpga

所以我先尝试了make build \
然后喜提如下错误
```
Updating git repository `https://github.com/Starry-OS/axprocess.git`
warning: spurious network error (3 tries remaining): failed to receive HTTP 200 response: got 401; class=Net (12)
warning: spurious network error (2 tries remaining): failed to receive HTTP 200 response: got 401; class=Net (12)
warning: spurious network error (1 tries remaining): failed to receive HTTP 200 response: got 401; class=Net (12)
error: failed to get `axprocess` as a dependency of package `user_test v0.1.0 (/home/hjch/taic/taic-test/apps/user_test)`
```
点进去那个git链接，发现根本没有这个仓库。点开与之相近的`axstd`仓库，发现路径已经从`Starry-OS/axstd.git`变成`Starry-Old/axstd.git`了。 \
因此猜测axprocess也可能是类似的情况。 \
在Starry-Old下找到了`axprocess-old`仓库，修改`taic-test`下所有相关的cargo.toml后，再次遇到问题
```
Updating git repository `https://github.com/Starry-OS/axerrno.git`
warning: spurious network error (3 tries remaining): could not read from remote repository; class=Net (12); code=Eof (-20)
warning: spurious network error (2 tries remaining): could not read from remote repository; class=Net (12); code=Eof (-20)
warning: spurious network error (1 tries remaining): could not read from remote repository; class=Net (12); code=Eof (-20)
error: failed to get `axerrno` as a dependency of package `axstd v0.1.0 (https://github.com/Starry-OS/axstd.git#0b0595b3)`
    ... which satisfies git dependency `axstd` (locked to 0.1.0) of package `enq_deq_test v0.1.0 (/home/hjch/taic/taic-test/apps/enq_deq_test)`

Caused by:
  failed to load source for dependency `axerrno`

Caused by:
  Unable to update https://github.com/Starry-OS/axerrno.git#3e2372cd

Caused by:
  failed to fetch into: /home/hjch/.cargo/git/db/axerrno-56b0b79a4e69071d

Caused by:
  network failure seems to have happened
  if a proxy or similar is necessary `net.git-fetch-with-cli` may help here
  https://doc.rust-lang.org/cargo/reference/config.html#netgit-fetch-with-cli

Caused by:
  could not read from remote repository; class=Net (12); code=Eof (-20)
make[1]: *** [scripts/make/build.mk:35: _cargo_build] Error 101
make[1]: Leaving directory '/home/hjch/taic/taic-test'
make: *** [Makefile:12: build] Error 2
```
这里就很有意思了，`Starry-OS/axerrno`仓库是存在的，尽管看起来它更应该是`Starry-Old/axerrno` \
为什么还会失败呢?我尝试根据提示设置`net.git-fetch-with-cli`，但是依然失败。 \
只不过报错信息变成了
```
Updating git repository `https://github.com/Starry-OS/axerrno.git`
fatal: remote error: upload-pack: not our ref 3e2372cde388aec5a666afd3456d59e8b77bf3bf
error: failed to get `axerrno` as a dependency of package `axstd v0.1.0 (https://github.com/Starry-OS/axstd.git#0b0595b3)`
    ... which satisfies git dependency `axstd` (locked to 0.1.0) of package `enq_deq_test v0.1.0 (/home/hjch/taic/taic-test/apps/enq_deq_test)`
```
这里的`not our ref`提示我可能是commit id不对，我看了下`Starry-OS/axerrno`中没有这个commit id， \
但是`Starry-Old/axerrno`中有这个commit id。 \
为什么`cargo.toml`中没写commit id, 但是它会指定到这个commit id呢？ \
是因为顶层的`cargo.lock`中给出了该依赖的commit id。 \

最开始，因为对`Starry-OS/axerrno`仓库的依赖是写在`axstd`仓库中的，我不能直接修改依赖项的内容 \
所以我只能fork了一份`axstd`，然后修改依赖项指向`Starry-Old/axerrno`， \
再把`taic-test`中对`axstd`的依赖改成我fork的版本。


但是没想到的是，依赖于`axerrno`的仓库还有`axprocess`, `linux_syscall_api`, `arceos_posix_api`等大量仓库 \
在我已经fork5个相关仓库后，我感觉这么干不是个头 \
而且这样做还间接导致`cargo.lock`中的记录无效，引入了新的依赖问题（因为默认拉取最新的commit，过程中`elf-parser`仓库被改名为`kernel-elf-parser`，导致cargo找不到`elf-parser`crate）

`cargo.toml`提供了一个可配置项`patch`，可以用来重定向依赖项 \
因此我最终将这些依赖重定向到正确的仓库，新增的内容如下：
```toml
[patch."https://github.com/Starry-OS/axerrno.git".axerrno]
git = "https://github.com/Starry-Old/axerrno.git"

[patch."https://github.com/Starry-OS/allocator.git".allocator]
git = "https://github.com/Starry-Old/allocator.git"

[patch."https://github.com/Starry-OS/axio.git".axio]
git = "https://github.com/Starry-Old/axio.git"

[patch."https://github.com/Starry-OS/axprocess.git".axprocess]
git = "https://github.com/Starry-Old/axprocess-old.git"

[patch."https://github.com/Starry-OS/axsignal.git".axsignal]
git = "https://github.com/Starry-Old/axsignal-old.git"
```

至此，`make build`成功，在此之后`make run_on_qemu`也成功运行起来了。

接下来尝试编译FPGA代码，执行`make build_fpga` \
结果报错
```
hjch@LAPTOP-RF714NVU:~/taic$ make build_fpga
make -C taic-rocket-chip build
make[1]: Entering directory '/home/hjch/taic/taic-rocket-chip'
/home/hjch/taic/taic-rocket-chip/rocket-chip/Makefrag:3: *** Please set environment variable RISCV. Please take a look at README.  Stop.
make[1]: Leaving directory '/home/hjch/taic/taic-rocket-chip'
make: *** [Makefile:36: build_fpga] Error 2
```
根据readme的提示，我需要先编译一套riscv工具链, 来自项目`https://github.com/chipsalliance/rocket-tools`

还是一样，先拉取所有子模块（因为网络原因折腾了一阵...）
```bash
git submodule update --init --recursive
```
然后按照说明安装所有依赖项
```bash
sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev libusb-1.0-0-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev device-tree-compiler pkg-config libexpat-dev libfl-dev
```
然后设置环境变量`RISCV`作为工具链安装地址，接着运行编译脚本
```bash
./build.sh
```
但是遇到问题`Error: unrecognized opcode fence.i'`

在该项目的issue中找到了解决方案 [issue#85](https://github.com/chipsalliance/rocket-tools/issues/85) \
将`build.sh`中的`CC= CXX= build_project riscv-pk --prefix=$RISCV --host=riscv64-unknown-elf`后添加一个`--with-arch=rv64gc_zifencei`参数

修改后重新运行`./build.sh`，成功编译

接着回到`taic`目录，设置环境变量`RISCV`为刚编译好的工具链路径，执行`make build_fpga`,显示没有找到`vivado`命令 \
`vivado`是Xilinx的FPGA开发工具，需要先安装, 根据`scripts/fpga.mk`中的提示，我选择安装了`2022.2`版本的个人版`Vivado`。在此之后还安装了`jdk17`用于运行vivado

此后尝试打开`vivado`，出现了一个奇怪的报错，大致是说不支持地区设置为`C.UTF-8` \
最开始我通过设置`export LC_ALL=en_US.UTF-8`，但是依然报错 \
```
hjch@LAPTOP-RF714NVU:~/taic$ vivado
/bin/bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
/bin/bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
/opt/Xilinx/Vivado/2022.2/bin/rdiArgs.sh: line 31: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
/bin/bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
terminate called after throwing an instance of 'std::runtime_error'
  what():  locale::facet::_S_create_c_locale name not valid
/opt/Xilinx/Vivado/2022.2/bin/rdiArgs.sh: line 312:  2247 Aborted                 (core dumped) "$RDI_PROG" "$@"
```
通过搜索，发现有人遇到了相同的问题 [链接](https://adaptivesupport.amd.com/s/question/0D54U00006FYojlSAD/vivado-20222-on-ubuntu-with-error-lcall-cannot-change-locale-enusutf8?language=en_US) \
解决方案是执行
```bash
sudo locale-gen "en_US.UTF-8"
sudo update-locale LANG=en_US.UTF-8
```
（我猜大致含义是生成en_US.UTF-8相关的语言环境） \
配置后又出现如下错误
```
vivado
application-specific initialization failed: couldn't load file "librdi_commontasks.so": libtinfo.so.5: cannot open shared object file: No such file or directory
```
使用apt安装libtinfo5(`sudo apt install libtinfo5`)之后，成功打开了vivado gui 

在`source /opt/Xilinx/Vivado/2022.2/settings64.sh`后，重新执行`make build_fpga`, 出现如下问题
```
/usr/bin/env: ‘python’: Permission denied
```
```
ERROR: [Common 17-69] Command failed: Part 'xczu15eg-ffvb1156-2-i' not found
```
对于第一个问题，通过创建`python3`的软链接`python`解决 \
对于第二个问题，是因为我安装时的选项选择了非付费版本，没有包含`xczu15eg-ffvb1156-2-i`这个型号的FPGA \
因此重新安装时选择了完整版vivado，然后使用了第三方证书激活成功 \

但是之后又遇到了新的问题
```
INFO: [Common 17-14] Message 'Synth 8-7052' appears 100 times and further instances of the messages will be disabled. Use the Tcl command set_msg_config to change the current settings.
/opt/Xilinx/Vivado/2022.2/bin/rdiArgs.sh: line 312: 32165 Killed                  "$RDI_PROG" "$@"
[Sat Dec 20 13:38:03 2025] synth_1 finished
WARNING: [Vivado 12-13638] Failed runs(s) : 'synth_1'
wait_on_runs: Time (s): cpu = 01:30:00 ; elapsed = 00:51:58 . Memory (MB): peak = 3206.840 ; gain = 0.000 ; free physical = 1838 ; free virtual = 3483
# launch_runs impl_1 -to_step write_bitstream -jobs 6
ERROR: [Common 17-70] Application Exception: Failed to launch run 'impl_1' due to failures in the following run(s):
synth_1
These failed run(s) need to be reset prior to launching 'impl_1' again.

INFO: [Common 17-206] Exiting Vivado at Sat Dec 20 13:38:04 2025...
make[1]: *** [Makefile:103: bitstream] Error 1
make[1]: Leaving directory '/home/hjch/taic/taic-rocket-chip'
make: *** [Makefile:38: build_fpga] Error 2
```
这个问题是因为内存不足导致的，我的真实机有32GB内存， \
但是我是在WSL2中运行的，WSL2默认只分配了16GB内存（也就是一半） \
因此我调整了WSL2的内存配置，增加到32GB \
之后`make build_fpga`，缺少了`xsct`命令

经过搜索，发现`xsct`是vitis工具集中的一个命令行工具 \
因此我重新安装了`Vitis`（基本全是安装的坑），最后成功了