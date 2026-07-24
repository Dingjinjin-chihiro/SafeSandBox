# Firejail 0.9.80 CLI-only：现代 C++17 渐进式重构（第二阶段）

## 1. 本阶段目标

第二阶段继续采用混合 C/C++17 架构，不重写 Namespace、挂载、Seccomp 等高风险核心流程。本阶段集中处理两个边界清晰、但长期存在裸指针和手工资源管理的模块：

- `$PATH` 解析与可执行文件查找；
- `--output` / `--output-stderr` 输出捕获与重执行。

根据当前重构策略，旧接口不再被视为必须保持兼容。本阶段同步修改了所有调用方，只保证项目整体行为和命令行使用方式正常。

## 2. PATH 模块重构

### 2.1 文件变化

```text
src/firejail/paths.c
    ↓
src/firejail/paths.cpp

新增：
src/firejail/path_list.hpp
```

### 2.2 数据结构变化

旧实现长期保存：

```c
static char **paths;
static unsigned int path_cnt;
```

调用者得到可变的 `char **`，并依赖末尾的空指针。新实现使用：

```cpp
std::vector<std::string>
```

路径由 `PathRegistry` 统一持有，调用者只能通过只读接口访问：

```c
size_t path_entry_count(void);
const char *path_entry_at(size_t index);
int path_program_exists(const char *program);
```

这样消除了以下问题：

- 调用者能够意外修改内部数组；
- 数量语义包含或不包含末尾空指针不清晰；
- `calloc`、`strdup`、`strsep` 和全局裸指针混合管理；
- 每个调用方都重复编写空指针终止循环。

### 2.3 使用的 C++17 能力

- `std::vector<std::string>`：明确字符串所有权；
- `std::string_view`：无复制解析 PATH；
- `std::unordered_set`：过滤重复目录；
- `std::filesystem::canonical`：判断 `/bin` 是否解析到 `/usr/bin`；
- `std::filesystem::path`：安全拼接目录与程序名；
- 函数局部静态对象：按需初始化注册表。

### 2.4 行为保持

新实现继续保持原有行为：

- PATH 不存在时写入默认值；
- 忽略相对路径；
- 去掉目录末尾 `/`；
- 忽略根目录 `/`；
- 删除重复项；
- `/bin -> /usr/bin` 时只保留先出现的一个；
- 可执行文件必须通过 `access(X_OK)`；
- 最终目标必须是普通文件，不能仅仅是目录。

### 2.5 调用方调整

同步修改：

```text
src/firejail/fs.c
src/firejail/landlock.c
src/firejail/x11.c
src/firejail/firejail.h
```

`${PATH}` 宏展开现在通过 `path_entry_count()` 和 `path_entry_at()` 完成。X11 服务程序检测改用 `path_program_exists()`。

## 3. 输出重定向模块重构

### 3.1 文件变化

```text
src/firejail/output.c
    ↓
src/firejail/output.cpp

新增：
src/firejail/output_options.hpp
src/firejail/pipe.hpp
```

主程序入口从：

```c
check_output(argc, argv);
```

改为：

```c
output_reexec_if_requested(argc, argv);
```

### 3.2 参数解析分离

`output_options.hpp` 提供无系统副作用的纯 C++17 参数解析：

```cpp
std::optional<Request> find_request(...);
std::vector<char *> filtered_argv(...);
```

其中 `Request` 明确保存：

- 参数位置；
- 输出文件名；
- 是否同时重定向标准错误。

解析逻辑不再依赖多个整数标志和指针偏移。

### 3.3 管道 RAII

新增 `firejail::Pipe`：

```cpp
class Pipe final {
    UniqueFd read_end_;
    UniqueFd write_end_;
};
```

Linux 下使用：

```c
pipe2(..., O_CLOEXEC)
```

两个管道端点由 `UniqueFd` 自动关闭，从而避免：

- 错误分支遗漏 `close()`；
- 父子进程关闭了错误的管道端；
- `exec` 失败时描述符泄漏；
- 同一描述符被重复关闭。

### 3.4 路径与字符串所有权

输出文件名使用：

```cpp
std::string
std::string_view
std::filesystem::path
```

继续保留原有安全检查：

- 拒绝符号链接；
- 拒绝目录；
- 拒绝包含 `..` 的文件名；
- 拒绝控制字符；
- 已存在文件必须属于当前用户；
- 拒绝硬链接文件。

`~` 展开不再使用 `asprintf()`，而是由 `std::string` 管理。

### 3.5 fork/exec 边界

所有 `std::filesystem` 转换和目标路径字符串分配都在 `fork()` 之前完成。子进程只执行必要操作：

```text
dup2
关闭无用管道端
恢复环境白名单
execv(ftee)
```

`execv()` 失败后使用 `_exit(127)`，避免重复刷新父进程继承的标准 I/O 缓冲区。

父进程则：

```text
关闭读端
dup2 写端到 stdout
按需复制 stdout 到 stderr
移除 Firejail 的 output 参数
恢复原始环境
execvp Firejail
```

## 4. 构建系统修正

原配置只搜索 `gawk`：

```m4
AC_CHECK_PROGS([GAWK], [gawk])
```

当系统只有 `awk` 时，变量为空，构建命令会退化为：

```text
f preproc.awk ...
```

第二阶段改为：

```m4
AC_CHECK_PROGS([GAWK], [gawk awk], [awk])
```

因此没有安装 GNU awk 的最小 Linux 环境也能正常生成：

- man 手册；
- Bash 补全；
- Zsh 补全；
- Profile 语法文件。

## 5. 新增测试

`test/cpp17` 现在包含：

```text
unique_fd_test.cpp
path_list_test.cpp
output_options_test.cpp
pipe_test.cpp
```

测试覆盖：

- 文件描述符自动关闭与移动语义；
- PATH 过滤、去重和 `/bin` 合并；
- output 参数识别和参数过滤；
- 管道读写及端点生命周期。

执行：

```bash
make cpp17-tests
```

## 6. 验证结果

完成的验证包括：

```text
完整 configure：通过
完整 make：通过
C++17 使用 -Werror：通过
C++17 单元测试：4/4 通过
AddressSanitizer：通过
UndefinedBehaviorSanitizer：通过
DESTDIR 安装：通过
firejail --version：通过
--output 真实重执行：通过
--output-stderr 真实重执行：通过
CLI-only 安装清单检查：通过
```

`--output` 实际测试会启动已安装的 `ftee`，捕获 Firejail 帮助文本；`--output-stderr` 实际测试会捕获无效参数错误。

## 7. 当前 C++17 模块

经过第一、第二阶段，主程序中已有：

```text
logger.cpp              spdlog 日志适配
run_files.cpp           /run/firejail 状态文件
paths.cpp               PATH 注册表与程序查找
output.cpp              输出捕获与重执行
unique_fd.hpp           文件描述符 RAII
pipe.hpp                管道 RAII
path_list.hpp           PATH 纯解析逻辑
output_options.hpp      输出参数纯解析逻辑
```

## 8. 第三阶段建议

下一阶段适合迁移以下模块之一：

### 方案 A：配置/Profile 字符串层

优先处理：

```text
macros.c
profile.c
preproc.c
```

目标：

- 使用 `std::string` 和 `std::string_view`；
- 将链表配置项改为拥有明确生命周期的容器；
- 将宏展开与 Profile 解析做成可单元测试的纯函数。

### 方案 B：进程启动辅助层

优先处理：

```text
process.c
shutdown.c
```

目标：

- 引入 PID/子进程守卫；
- 统一 `waitpid()` 和信号中断处理；
- 明确父进程、沙盒 PID 1、应用进程的退出状态传播。

在继续迁移前，不建议立即重写 `sandbox.c`、`fs.c` 或 `seccomp.c` 的完整主体。这些模块处于特权边界，应该先建立更多集成测试后再分段迁移。
