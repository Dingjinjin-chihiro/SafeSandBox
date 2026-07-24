# Firejail 0.9.80 CLI-only：现代 C++17 渐进式重构（第一阶段）

## 1. 阶段目标

本阶段不尝试将 Firejail 一次性从 C 重写成 C++。Firejail 涉及 SUID、Namespace、Seccomp、挂载和进程创建，一次性改写会显著扩大安全回归面。

第一阶段只建立可持续迁移的基础设施：

1. 支持 C 与 C++17 源文件混合编译；
2. 保持原有 C ABI，未迁移模块无需修改；
3. 引入 `spdlog`，先替换调试级 Syslog 日志；
4. 引入 POSIX 文件描述符 RAII；
5. 将一个真实运行模块 `run_files` 迁移到 C++17；
6. 增加独立、无需 root 的 C++17 单元测试。

## 2. 已完成的源码修改

### 2.1 混合 C/C++17 构建

修改文件：

- `configure.ac`
- `configure`
- `config.mk.in`
- `src/prog.mk`
- `src/firejail/Makefile`

构建系统现在会：

- 使用 `AC_PROG_CXX` 检测 C++ 编译器；
- 编译一个 `std::string_view` 探针，强制要求 C++17；
- `.c` 文件继续由 `CC` 编译；
- `.cpp` 文件由 `CXX -std=c++17` 编译；
- 只有包含 C++ 对象的目标才改用 `CXX` 链接；
- 其他纯 C 辅助程序仍然使用 C 链接器，不被强制引入 `libstdc++`。

### 2.2 C ABI 日志适配层

新增文件：

- `src/firejail/logger_c.h`
- `src/firejail/logger.cpp`

`logger_c.h` 只暴露以下 C 接口：

```c
void fj_log_syslog_info(const char *message);
void fj_log_syslog_error(const char *message);
```

因此原有 C 模块不需要理解 C++ 类型，也不会出现异常跨越 C ABI 的问题。

`src/firejail/util.c` 中的：

- `logmsg`
- `logerr`
- `logsignal`

已经接入同步 `spdlog` Syslog Sink。

为了兼容 Firejail 大量 `fork()` 的执行模型，Logger 不使用异步线程池，也不长期缓存全局 Sink。每次记录日志时创建短生命周期同步 Logger，从而避免把锁状态带入子进程。

所有日志异常都会被捕获，并回退到 libc `syslog()`；日志失败不会终止沙盒启动器。

用户可见的 `fmessage()` 和 `fwarning()` 暂时保持原有 `stdio` 行为，因为它们允许无换行的分段输出，直接切换到普通 spdlog Logger 会改变终端输出语义。

### 2.3 文件描述符 RAII

新增文件：

- `src/firejail/unique_fd.hpp`

`firejail::UniqueFd` 具有以下语义：

- 析构时自动执行 `close()`；
- 禁止复制；
- 支持移动构造和移动赋值；
- 支持 `release()` 将所有权交还给旧 C 代码；
- 支持 `reset()` 替换资源；
- 析构时保存并恢复 `errno`，避免清理动作覆盖真实错误。

### 2.4 `run_files.c` 迁移为 `run_files.cpp`

迁移前：

- 大量 `asprintf/free`；
- 手工维护裸文件描述符；
- 多个高度重复的路径拼接函数；
- `sandbox_lock_fd` 需要显式关闭；
- C++17 不支持的 C99 指定初始化写法。

迁移后：

- 使用 `std::string` 和 `std::string_view`；
- 使用统一的 `run_path()` 构造运行时路径；
- 使用统一的 `write_all()` 处理短写和 `EINTR`；
- 使用 `UniqueFd` 管理状态文件和锁文件；
- `sandbox_lock_fd` 通过移动所有权保存；
- C++ 异常全部在 `extern "C"` 边界内部捕获；
- 对外函数名和调用方式保持不变。

迁移的 C ABI 包括：

```c
void delete_run_files(pid_t pid);
void delete_bandwidth_run_file(pid_t pid);
void set_name_run_file(pid_t pid);
void set_x11_run_file(pid_t pid, int display);
void set_profile_run_file(pid_t pid, const char *fname);
void set_sandbox_run_file(pid_t pid, pid_t child);
void release_sandbox_lock(void);
```

因此 `main.c`、`sandbox.c` 等调用方无需同时迁移。

### 2.5 C/C++ 公共头文件兼容性

修改：

- `src/include/common.h`
- `src/include/pid.h`
- `src/firejail/main.c`
- `src/firejail/network_main.c`

主要处理：

- 防止 `_GNU_SOURCE` 重复定义；
- 将只返回字符串常量的 `in_netrange()` 改为 `const char *`；
- 对应调用点使用 `const char *`。

这既消除了 C++ 编译警告，也强化了常量正确性。

## 3. spdlog 集成方式

依赖位于：

```text
third_party/spdlog/
├── include/spdlog/
└── LICENSE
```

当前使用上传的 spdlog 1.17.0，并采用 header-only 模式，不需要系统提前安装 `libspdlog-dev`。

## 4. 构建与测试

### 4.1 完整构建

```bash
./configure --prefix=/usr
make -j"$(nproc)"
```

配置输出中应出现：

```text
CXX: g++
C++ standard: C++17
```

### 4.2 C++17 RAII 测试

```bash
make cpp17-tests
```

测试覆盖：

- 作用域结束自动关闭；
- 移动所有权；
- `release()`；
- `reset()` 关闭旧描述符。

### 4.3 基本运行检查

```bash
src/firejail/firejail --version
src/firejail/firejail --help
```

实际 Namespace/Seccomp 集成测试应在支持 user namespace、mount namespace 和所需权限的 Linux 主机或虚拟机中执行。

## 5. 第一阶段刻意没有修改的部分

以下内容仍然保留 C 实现：

- 命令行参数解析；
- Profile 解析器；
- Namespace 创建；
- 挂载和文件系统隔离；
- Seccomp 规则加载；
- Capability 管理；
- 网络 Namespace 和 Netfilter；
- 子进程监控和信号处理。

这不是遗漏，而是渐进式迁移策略。每次只迁移一个边界清晰、可以独立验证的模块。

## 6. 建议的后续阶段

### 第二阶段：路径与字符串基础层

建议迁移：

- `paths.c`
- 部分 `env.c`
- 路径拼接和动态字符串工具

目标：用 `std::filesystem::path`、`std::string`、`std::vector` 替换重复的 `malloc/asprintf/free`，但继续对外提供 C ABI。

### 第三阶段：进程与管道 RAII

建议迁移：

- `output.c`
- D-Bus 管道管理的局部代码
- `sbox.c` 中的子进程执行封装

目标：引入 `UniqueFd`、`Pipe`、`ChildProcess` 和 Scope Guard，减少错误路径上的描述符泄漏。

### 第四阶段：强类型配置

将大量全局 `int arg_xxx` 逐步聚合为：

```cpp
struct SandboxOptions;
struct NetworkOptions;
struct FilesystemOptions;
struct SecurityOptions;
```

此阶段不能直接整体替换 `Config cfg`，应先建立只读 View，再逐步移动所有权。

### 第五阶段：Profile 解析器

将 Profile 解析拆分为：

- Lexer/Line Reader；
- 命令模型；
- 校验层；
- 执行层。

使用 `std::variant` 表示不同规则类型，避免依赖大量字符串前缀分支。

## 7. 安全约束

Firejail 可能以 SUID root 方式安装，后续重构必须遵守：

1. 不让 C++ 异常跨越 C ABI；
2. 不在信号处理器中调用非 async-signal-safe 的 C++/spdlog API；
3. 不使用异步日志线程池穿越 `fork()`；
4. 所有文件打开默认考虑 `O_CLOEXEC`、符号链接和所有权；
5. 每个迁移阶段必须同时做普通用户、root、SUID 和 Namespace 回归测试；
6. 不能仅以“编译成功”代替安全验证。

## 8. 当前阶段结论

该版本已经不再是纯 C 工程，但也没有破坏原有模块边界。它建立了一个可重复使用的迁移模式：

```text
原有 C 调用方
      │
      ▼
稳定的 extern "C" 接口
      │
      ▼
C++17 实现：RAII / string / spdlog
```

后续可以按同样方式一次迁移一个模块，而不需要同步重写整个 Firejail。
