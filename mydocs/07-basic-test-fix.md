# 🔧 RISC-V Basic 测试修复报告

> 📅 生成时间：`2026-06-01 15:01 (GMT+8)`
>
> 目标：诊断并修复 RISC-V 评测中 basic 测试集完全 0 分的问题

---

## 一、问题诊断

### 1.1 现象

运行 `make all && docker run ...` 评测后，`os_serial_out_rv.txt` 中：

```
===== SpringOS Auto Test Runner =====

init: scanning /glibc for *_testcode.sh ...
init: running [basic_testcode.sh]
init: running [basic_testcode.sh]
de.sh ...
box_testcode.sh: applet not found
init: 1 test scripts completed in /glibc
init: scanning /musl for *_testcode.sh ...
init: running [basic_testcode.sh]
init: running [basic_testcode.sh]
e.sh ...
box_testcode.sh: applet not found
init: 1 test scripts completed in /musl

All tests completed, shutting down system...
```

**关键问题：** 串口输出中**完全没有任何** `#### OS COMP TEST GROUP START basic-glibc ####` 标记，也没有任何 `========== START test_xxx ==========` 行。

评测脚本 `judge_basic-glibc.py` 用正则 `r"========== START (.+) =========="` 逐行匹配，匹配不到任何内容 → **32 个 basic 测试全部 0 分**。

### 1.2 根因分析（三条）

```
评测机执行的操作                    内核里实际发生的事情
────────────────                    ──────────────────

Step 1: make all
  → 编译 init-rv.c                  ✅ 编译成功
    其中 test_pre() 函数：
    - 已完整实现，含 32 个测试
    - 含 #### START/END 标记
    - 但 main() 里没调用它 ❌

Step 2: QEMU 启动内核
  → 内核启动 → init 进程            ✅ 正常启动
  → main() 执行路径：
    1. run_dir_testscripts("/glibc")
       → 扫描 basic_testcode.sh
       → execve("busybox", {"busybox",
              "sh","basic_testcode.sh"})
                                    ❌ busybox 启动但
                                       "applet not found"
       原因：busybox 二进制可能
       未编译 CONFIG_SH → 报错
    2. run_dir_testscripts("/musl")
       → 同上，同样失败

Step 3: 评测脚本解析
  → judge_basic-glibc.py            ❌ 找不到任何
    逐行读串口输出                    START/END 标记
  → 所有 32 个测试: score = 0        全部 0 分
```

**三条根因：**

| # | 根因 | 严重度 | 说明 |
|---|------|--------|------|
| 1 | `test_pre()` 在 `init-rv.c` 的 `main()` 中未被调用 | 🔴 致命 | 32 个 basic 测试的完整代码已写好，但从未执行 |
| 2 | `run_dir_testscripts()` 依赖 `busybox sh`，但 busybox 报 `applet not found` | 🔴 致命 | `busybox` 二进制可能未编译 shell applet（`CONFIG_ASH`/`CONFIG_HUSH`），或 argv 传递有 bug |
| 3 | `getdents64()` 返回重复条目 | 🟡 次要 | `init: running [basic_testcode.sh]` 出现两次，说明 VFS/ext4 实现有重复返回 bug |

LoongArch 端 `init-la.c` 同样存在 `test_pre()` / `test_final()` 被注释掉的对称问题。

---

## 二、修复方案

### 2.1 核心思路：直接用 `test_pre()` 而非依赖 busybox

不依赖脚本文件和 busybox shell，改为直接在 init 进程中 `fork()+exec()` 每个测试二进制——init 本身就是 init 进程，可以直接执行测试程序：

```
修复前：
  init → run_dir_testscripts
       → busybox sh basic_testcode.sh  ← busybox sh 不可用 ❌
       → 无输出 → 0 分

修复后：
  init → test_pre()
       → chdir /glibc/basic/
       → fork+exec("brk") → 测试输出 ✅
       → fork+exec("chdir") → 测试输出 ✅
       → ... (共 32 个)
       → #### 标记 → 评测脚本正常匹配 → 得分 ✅
```

### 2.2 具体修改

#### `user/init-rv.c` — main() 调用 test_pre()

```c
// 修复前（只跑 run_dir_testscripts，busybox 不可用 → 全部 0 分）
run_dir_testscripts("/glibc");
run_dir_testscripts("/musl");

// 修复后（先跑 test_pre 直接 exec 测试二进制，run_dir_testscripts 留待 busybox 修复）
test_pre();
// run_dir_testscripts("/glibc");    // 待 busybox 修复后取消注释
// run_dir_testscripts("/musl");
```

**改动说明：**
- `test_pre()` 含有完整的 32 个 basic 测试（brk/chdir/clone/.../yield/mmap/munmap）
- 包含 `#### OS COMP TEST GROUP START basic-glibc ####` 和 `END` 标记——评测脚本正是匹配这些标记
- glibc + musl 两套 C 库的测试都会执行
- `run_dir_testscripts()` 注释保留，等 busybox 的 `sh` applet 问题修复后再启用

#### `user/init-la.c` — 取消注释 test_pre() 和 test_final()

```c
// 修复前
// test_pre();
// test_final();
test_on_site();

// 修复后
test_pre();       // basic 测试（glibc + musl，各 32 个）
test_final();     // interrupts + copy-file-range + splice 测试
// test_on_site();  // git 测试（当前 SD 镜像缺少 git_testcode.sh，暂时保留注释）
```

**改动说明：**
- `test_pre()` 与 `init-rv.c` 相同逻辑，直接 exec basic 测试二进制
- `test_final()` 包含 interrupts（2个测试）、copy-file-range（4个测试）、splice（5个测试），均带 START/END 标记
- `test_on_site()` 中的 git 测试依赖 `git_testcode.sh`，当前镜像缺失，暂保留注释

---

## 三、修复影响分析

### 3.1 正面影响

| 测试组 | 修复前 | 修复后 |
|--------|--------|--------|
| basic-glibc (32 tests) | ❌ 0 分 | ✅ 正常输出标记 + 各测试执行结果 |
| basic-musl (32 tests) | ❌ 0 分 | ✅ 同上 |
| interrupts-musl (2 tests) | ❌ 无输出 | ✅ `test_final()` 含 START/END 标记 |
| interrupts-glibc (2 tests) | ❌ 无输出 | ✅ 同上 |
| copyfilerange-musl (4 tests) | ❌ 无输出 | ✅ 同上 |
| copyfilerange-glibc (4 tests) | ❌ 无输出 | ✅ 同上 |
| splice-musl (5 tests) | ❌ 无输出 | ✅ 同上 |
| splice-glibc (5 tests) | ❌ 无输出 | ✅ 同上 |

### 3.2 不影响的测试组

| 测试组 | 原因 |
|--------|------|
| busybox-glibc/musl | 需要 `busybox sh busybox_testcode.sh`，busybox 的 `sh` applet 仍未修复 |
| lua-glibc/musl | 需要 `busybox sh lua_testcode.sh`，同上 |
| libctest-glibc/musl | 需要 `busybox sh libctest_testcode.sh`，同上 |
| iperf/netperf/cyclictest | 需要完整网络栈 + 实时调度，尚未实现 |

### 3.3 后续依然需要修复的

1. **busybox `sh` applet 问题** — 这是所有后续测试组的基础依赖，需要排查：
   - 检查 SD 卡上的 busybox 二进制是否编译了 shell applet
   - 检查内核 execve() 的 argv 传递是否正确
   
2. **`getdents64()` 重复返回 bug** — `run_dir_testscripts()` 中 `init: running [basic_testcode.sh]` 出现两次

3. **各测试二进制本身的 syscall 实现** — `test_pre()` 能输出标记和基础结果，但每个测试需要对应的 syscall 实现才能通过断言

---

## 四、各级评测脚本需要什么

### 4.1 judge_basic-glibc.py 匹配规则

评测脚本读取两个 `#### OS COMP TEST GROUP START/END basic-glibc ####` 之间的所有行，然后对每个 `========== START test_xxx ==========` / `========== END test_xxx ==========` 区间进行正则匹配。

**每个测试的期望输出举例：**

| 测试 | 匹配模式 | 断言 |
|------|---------|------|
| brk | `Before alloc,heap pos: X` → `After alloc,heap pos: Y` | Y = X + 64, Z = Y + 64 |
| chdir | `chdir ret: 0`, `test_chdir` | ret 为 0，含 `test_chdir` |
| clone | `Child says successfully!`, `pid:数字`, `clone process successfully.` | 3 条都匹配 |
| execve | `I am test_echo.`, `execve success.` | 精确相等 |
| yield | 15 行输出，含 A/B/C 字母循环 | A≥3, B≥3, C≥3 |

完整列表见 `/home/chaos/autotest-for-oskernel/kernel/judge/judge_basic-glibc.py`。

### 4.2 输出格式要求

```
#### OS COMP TEST GROUP START basic-glibc ####   ← 评测脚本开始监听
========== START test_brk ==========              ← 单个测试开始
Before alloc,heap pos: 0                          ← 测试输出
After alloc,heap pos: 64
Alloc again,heap pos: 128
========== END test_brk ==========                ← 单个测试结束
========== START test_chdir ==========
chdir ret: 0
test_chdir
========== END test_chdir ==========
...
#### OS COMP TEST GROUP END basic-glibc ####     ← 评测脚本停止监听并打分
```

---

## 五、验证方式

### 5.1 本地验证

```bash
# 1. 重新编译
cd /home/my-springos && make clean && make all

# 2. 确认 kernel-rv 和 kernel-la 已重新生成
ls -la kernel-rv kernel-la

# 3. 运行评测（如果 Docker 环境已就绪）
cd /home/chaos
sudo docker run --rm \
  -v /home/my-springos:/coursegrader/submit \
  -v /home/chaos/oscomp-testdata:/coursegrader/testdata \
  -v /home/chaos/autotest-for-oskernel:/cg \
  -v /home/chaos/oscomp-testdata:/mnt/cghook/ \
  zhouzhouyi/os-contest:20260510 python3 /cg/kernel.zip

# 4. 检查串口输出中是否出现 #### OS COMP TEST GROUP START basic-glibc ####
grep "OS COMP TEST GROUP" /home/my-springos/os_serial_out_rv.txt
```

### 5.2 预期输出

编译并运行后，应能在 `os_serial_out_rv.txt` 中看到：

```
#### OS COMP TEST GROUP START basic-glibc ####
========== START test_brk ==========
...
========== END test_brk ==========
========== START test_chdir ==========
...
========== END test_chdir ==========
...
#### OS COMP TEST GROUP END basic-glibc ####
#### OS COMP TEST GROUP START basic-musl ####
...
#### OS COMP TEST GROUP END basic-musl ####
```

---

## 六、变更记录

| 文件 | 变更 | 说明 |
|------|------|------|
| `user/init-rv.c` | `main()` 中新增 `test_pre()` 调用；`test_final()`、`test_on_site()`、`run_dir_testscripts()` 用 `#if 0` 包裹 | 绕过 busybox 依赖，直接 exec basic 测试二进制；避免 `-Werror=unused-function` 编译错误 |
| `user/init-la.c` | `main()` 中取消注释 `test_pre()` 和 `test_final()`；`test_on_site()` 用 `#if 0` 包裹 | 启用 basic + interrupts + copy-file-range + splice 测试 |

### 6.1 编译错误修复记录（2026-06-01 15:39）

**现象：** 首次提交后线上评测机报 `Compile Error`：
```
user/init-rv.c:364:13: error: 'run_dir_testscripts' defined but not used [-Werror=unused-function]
cc1: all warnings being treated as errors
```

**原因：** `Makefile` 中 `CFLAGS` 含 `-Werror`，注释掉 `run_dir_testscripts()` 的调用后，函数定义仍存在但未被使用 → 触发 warning-as-error。

**修复：** 将所有暂不使用的函数用 `#if 0 ... #endif` 包裹（而非仅注释调用），编译器不会解析 `#if 0` 内部代码，彻底消除 unused-function 警告：
- `init-rv.c`：`test_final()` + `test_on_site()` + `run_dir_testscripts()` → `#if 0`
- `init-la.c`：`test_on_site()` → `#if 0`

---

> **总结一句话：** `init-rv.c` 中已写好的 `test_pre()` 函数包含完整的 32 个 basic 测试和评测标记格式，修复只需在 `main()` 中调用它即可——本质上是从 "依赖 broken busybox" 切换到 "直接 fork+exec 测试二进制"。
