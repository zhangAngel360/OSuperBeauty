# 🔁 Docker 测评工作流优化

> 📅 生成时间：`2026-06-02 15:18 (GMT+8)`
>
> 目标：解决测评中 `gzip` 报错残留问题，优化 Makefile 让测评一键化，设置目录权限防止误改。

---

## 一、问题背景：`gzip: sdcard-rv.img already exists`

### 1.1 报错现场

```text
gzip: sdcard-rv.img already exists; not overwritten
Exception in thread Thread-1 (run_qemu):
...
subprocess.CalledProcessError: Command 'gzip -d sdcard-rv.img.gz' returned non-zero exit status 2.
```

### 1.2 根因分析

整个链路是这样的：

```
上次 docker run ──→ 解压 sdcard-rv.img.gz ──→ 生成 sdcard-rv.img
                                                        │
                        ┌───────────────────────────────┘
                        ▼
                  ──rm 容器已退出──                      ← 空文件残留！
                  / 但挂载目录 /home/my-springos 里的
                  \ .img 文件还在！
                        │
                        ▼
下次 docker run ──→ gzip -d 发现同名文件 ──→ 拒绝覆盖 ──→ exit 2 ──→ 崩溃
```

**关键点：**

| 机制 | 说明 |
|------|------|
| `docker run --rm` | 只清理容器本身，**不碰你挂载进去的宿主机目录** |
| `gzip -d`（不带 `-f`） | 默认拒绝覆盖已有文件，返回退出码 2 |
| `run_qemu.py` 用 `subprocess.check_output` | `check=True` 意味着非零退出码直接抛异常 |

两条路线碰在一起 = 每次跑完如果不手动删除 `.img`，下次就跑不起来。

> **为什么只报 `sdcard-rv.img` 不报 `sdcard-la.img`？** 谁残留炸谁。脚本是 **顺序解压** 的，`.rv.img` 先解压碰到残留就炸在它这里，`.la.img` 根本没轮到就崩了。如果你的工作目录里两个都残留了，解压 `.la.img` 时也会炸。

### 1.3 查看残留文件

```bash
ls -lh /home/my-springos/sdcard-*.img
```

---

## 二、解决方案

### 方案 A：每次手动删（简单但不优雅）

```bash
rm -f /home/my-springos/sdcard-*.img && docker run ...
```

### 方案 B：改评测脚本（一劳永逸但需动官方代码）

把 `run_qemu.py` 第 57 行附近的 `gzip -d` 改成 `gzip -df`（`-f` 强制覆盖）：

```bash
# 找到并修改
cd /home/chaos/autotest-for-oskernel/kernel
sed -i "s/gzip -d sdcard/gzip -df sdcard/g" run_qemu.py
# 重新打包
zip ../kernel.zip -r *
```

### ✅ 方案 C：改 Makefile（推荐，本次已实施）

直接在 `/home/my-springos/Makefile` 里加三个目标，让测评工作流一行命令搞定。

#### 新增目标

```makefile
# 只清理测评产生的 img
clean-test:
	rm -f sdcard-*.img

# 一键 RISC-V 测评
test: clean-test kernel-rv
	docker run --rm \
		-v $(PWD):/coursegrader/submit \
		-v /home/zhangshuoyu/oscomp-testdata:/coursegrader/testdata \
		-v /home/zhangshuoyu/autotest-for-oskernel:/cg \
		-v /home/zhangshuoyu/oscomp-testdata:/mnt/cghook/ \
		zhouzhouyi/os-contest:20260510 python3 /cg/kernel.zip

# 一键 LoongArch 测评
test-la: clean-test kernel-la
	docker run --rm \
		-v $(PWD):/coursegrader/submit \
		-v /home/zhangshuoyu/oscomp-testdata:/coursegrader/testdata \
		-v /home/zhangshuoyu/autotest-for-oskernel:/cg \
		-v /home/zhangshuoyu/oscomp-testdata:/mnt/cghook/ \
		zhouzhouyi/os-contest:20260510 python3 /cg/kernel.zip
```

#### 修改说明

| 修改点 | 作用 |
|--------|------|
| 新增 `clean-test` 目标 | 单独删除 `sdcard-*.img`，不动其他构建产物 |
| 新增 `test` 目标 | 先清理 → 构建 RISC-V 内核 → 启动 Docker 测评 |
| 新增 `test-la` 目标 | 同上，针对 LoongArch |
| `clean` 目标追加 `sdcard-*.img` | `make clean` 也会顺带清掉测评残留 |
| `.PHONY` 追加 `clean-test test test-la` | 声明为伪目标，避免和同名文件冲突 |

#### 使用方式

```bash
# 以前（5 步）
cd /home/my-springos
rm -f sdcard-*.img
make all
cd /home/chaos
docker run --rm -v /home/my-springos:... zhouzhouyi/os-contest:20260510 python3 /cg/kernel.zip

# 现在（1 步）
cd /home/my-springos
make test          # RISC-V 测评
make test-la       # LoongArch 测评
```

### 方案 D：把 `sdcard-*.img` 加到 `.gitignore`

避免不小心把几百 MB 的镜像文件提交到 Git：

```bash
echo "sdcard-*.img" >> /home/my-springos/.gitignore
```

---

## 三、附属操作：目录权限设置

### 3.1 需求

将 `/home/my-springos` 设为除 `chaos` 外其他用户**只读**。

### 3.2 操作命令

```bash
# 目录本身
chmod 755 /home/my-springos

# 递归设置子内容（文件 644，目录 755）
find /home/my-springos -type f -exec chmod 644 {} +
find /home/my-springos -type d -exec chmod 755 {} +
```

### 3.3 权限效果对照

```
操作前（775）：  drwxrwxr-x   chaos chaos   ← 同组用户可写
操作后（755）：  drwxr-xr-x   chaos chaos   ← 除 chaos 外全部只读
```

| 角色 | 读 | 写 | 执行（进入目录） | 说明 |
|------|:--:|:--:|:--:|------|
| **chaos** | ✅ | ✅ | ✅ | owner 权限优先，不受组权限影响 |
| chaos 组内其他用户 | ✅ | ❌ | ✅ | 组权限变为 `r-x` |
| 其他用户 | ✅ | ❌ | ✅ | 不改动，本来就是 `r-x` |

### 3.4 注意事项

- 如果 `tools/` 下有需要执行的脚本（`.sh`），权限调完后可能需要补一下 `chmod +x`：
  ```bash
  chmod +x /home/my-springos/tools/*.sh
  ```
- 如果之后从 Git 拉新文件，可能需要重新跑一次上面的 `find ... chmod` 命令。
- **不要在 root 下直接 `chmod -R 755`** — 会把可执行文件的 `x` 位全干掉。上述两条 `find` 命令区分了文件和目录，更安全。

---

## 四、工作流总结

```
┌─────────────────────────────────────────────────┐
│               每次写代码改内核                      │
│                                                     │
│  ① 写代码                                           │
│  ② make test    ← 一键：清理 → 构建 → 测评         │
│  ③ 看分数                                           │
│  ④ git commit -am "xxx"                             │
│  ⑤ 回到 ①                                          │
│                                                     │
└─────────────────────────────────────────────────┘

不需要：
  ✗ 手动 rm sdcard-*.img
  ✗ 手动 cd 到 chaos 目录
  ✗ 手敲一长串 docker run 命令
  ✗ 担心别人误改你的 os 代码
```

---

> **核心收获：** `--rm` 只清容器不清挂载目录；`.img` 每次都从 `.gz` 重新生成，删了没损失；三个 Makefile 目标 + 两条权限命令，让测评工作流从 5 步变 1 步。
