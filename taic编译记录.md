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