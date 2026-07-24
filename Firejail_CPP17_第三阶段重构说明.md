# Firejail 0.9.80 CLI-only：现代 C++17 渐进式重构第三阶段

## 1. 阶段目标

第三阶段继续采用渐进式重构，而不是一次性重写整个 Firejail。重点处理 Profile 读取链路和运行时目录预处理模块：

- 将 `macros.c` 完整迁移为 C++17；
- 将 `preproc.c` 完整迁移为 C++17；
- 将 `profile.c` 改为 C++17 编译单元，并优先重构可独立验证的解析基础设施；
- 保持 Firejail 现有命令行用法和安全策略行为；
- 对 C++ 异常设置明确边界，禁止异常穿过 C 调用链；
- 为新增解析组件建立独立单元测试。

第三阶段没有把 `profile.cpp` 中约两千行规则分派逻辑全部改写成面向对象代码。当前阶段先解决字符串所有权、文件读取、宏展开、条件表达式和列表处理；具体规则类型化将在后续阶段完成。

---

## 2. 主要文件变化

### 2.1 C 文件迁移为 C++17

```text
src/firejail/macros.c
    -> src/firejail/macros.cpp

src/firejail/preproc.c
    -> src/firejail/preproc.cpp

src/firejail/profile.c
    -> src/firejail/profile.cpp
```

### 2.2 新增可测试的头文件组件

```text
src/firejail/macro_expander.hpp
src/firejail/profile_condition.hpp
src/firejail/profile_list.hpp
```

### 2.3 新增单元测试

```text
test/cpp17/macro_expander_test.cpp
test/cpp17/profile_condition_test.cpp
test/cpp17/profile_list_test.cpp
```

现有 C++17 单元测试总数由 4 组增加到 7 组。

---

## 3. 宏展开模块重构

### 3.1 静态宏表

旧实现使用可修改的 C 结构体数组和 `char *` 字段。新实现使用：

```cpp
inline constexpr std::array<Definition, 6> definitions;
```

每个宏定义包含：

- 宏名称；
- `user-dirs.dirs` 中对应的 XDG 前缀；
- 最多 7 个本地化目录候选名称。

支持的目录宏仍然包括：

```text
${DOWNLOADS}
${MUSIC}
${VIDEOS}
${PICTURES}
${DESKTOP}
${DOCUMENTS}
```

### 3.2 纯函数解析

新增的 `macro_expander.hpp` 提供不依赖 Firejail 全局变量的解析函数：

```cpp
is_macro_syntax(...)
find_definition(...)
parse_xdg_line(...)
expand(...)
```

因此以下行为可以直接单元测试：

- `${HOME}` 展开；
- `~` 展开；
- `${CFG}` 展开；
- `${RUNUSER}` 展开；
- XDG 用户目录解析；
- 禁止使用 `$HOME`；
- 未识别字符串保持原样。

### 3.3 文件系统检查

XDG 和本地化目录候选使用：

```cpp
std::filesystem::path
std::filesystem::exists(path, error_code)
```

使用 `std::error_code` 避免普通文件不存在时抛出异常。

### 3.4 有效用户权限 RAII

旧代码通过多个分支手工调用：

```c
EUID_USER();
...
EUID_ROOT();
```

新实现增加 `UserPrivilegeScope`：

```cpp
class UserPrivilegeScope final {
public:
    UserPrivilegeScope();
    ~UserPrivilegeScope();
};
```

只要函数离开作用域，就会恢复原来的 root 有效身份，减少新增返回分支时遗漏恢复操作的风险。

### 3.5 C 调用边界

由于项目仍然是 C/C++ 混合工程，`expand_macros()` 等接口暂时继续返回由 `malloc` 分配的 `char *`，以便未迁移的 C 模块调用。

C++ 内部不再使用 `asprintf()` 拼接路径，而是使用 `std::string`。返回 C 调用方前统一复制为 C 字符串。

---

## 4. 运行时预处理模块重构

### 4.1 删除公开锁文件描述符

旧实现将以下变量放在 `main.c` 中：

```c
int lockfd_directory;
int lockfd_network;
```

第三阶段删除了这两个全局公开变量。锁的文件描述符由 `preproc.cpp` 内部的对象持有：

```cpp
RuntimeLock directory_lock;
RuntimeLock network_lock;
```

调用方只能通过：

```c
preproc_lock_firejail_dir();
preproc_unlock_firejail_dir();
preproc_lock_firejail_network_dir();
preproc_unlock_firejail_network_dir();
```

操作锁，不能直接修改文件描述符状态。

### 4.2 `RuntimeLock` RAII

`RuntimeLock` 负责：

- 使用 `O_CLOEXEC` 打开锁文件；
- 设置 root 所有权；
- 指数退避等待 `flock()`；
- 保存和恢复 `SIGTSTP` 处理器；
- 释放锁；
- 关闭文件描述符。

文件描述符使用第一阶段引入的：

```cpp
firejail::UniqueFd
```

管理。

### 4.3 清理 `/run/firejail` 状态文件

旧实现使用：

```c
malloc + memset
opendir + readdir
strtol
```

新实现使用：

```cpp
std::vector<unsigned char>
std::filesystem::directory_iterator
std::from_chars
std::optional
```

处理流程为：

```text
读取 pid_max
    -> 建立活动 PID 位图
    -> 遍历 /proc
    -> 标记活动 PID
    -> 遍历 profile/name 状态目录
    -> 删除无对应活动进程的状态文件
```

### 4.4 异常边界

`preproc_clean_run()` 可能触发 STL 分配或文件系统异常，因此增加统一异常边界：

```cpp
guard_preproc_cpp(...)
```

任何 C++ 异常都会在 C++ 模块内部转换为 Firejail 的错误退出，不会穿过 C ABI。

---

## 5. Profile 模块重构

## 5.1 `profile.c` 改为 C++17 编译单元

`profile.cpp` 仍然保留现有规则检查主体，以控制本阶段风险，但以下基础部分已经重写：

- Profile 搜索；
- Profile 文件读取；
- include 层级管理；
- 条件语句解析；
- Profile 条目追加；
- 逗号列表压缩和增量修改；
- 被解析行的内存所有权。

### 5.2 Profile 搜索

旧实现通过 `opendir()` 和 `readdir()` 扫描整个目录寻找精确文件名。

新实现直接构造：

```cpp
std::filesystem::path profile_path = directory / profile_name;
```

并检查目标是否存在。搜索顺序保持不变：

```text
~/.config/firejail
    -> /etc/firejail
```

### 5.3 Profile 文件 RAII

Profile 文件由：

```cpp
std::unique_ptr<FILE, fclose>
```

管理。任何返回路径都能自动关闭文件。

每行缓冲区改为：

```cpp
std::array<char, 8193>
```

### 5.4 include 层级 RAII

旧实现：

```c
include_level++;
profile_read(...);
include_level--;
```

新实现：

```cpp
IncludeLevelGuard include_guard;
profile_read(...);
```

递归读取正常返回或因 C++ 异常退出当前作用域时，层级都会恢复。

### 5.5 Profile 行所有权

旧实现存在一个隐式规则：

> 某些 `profile_check_line()` 分支把指针直接保存到全局配置中，因此读取后的行不能释放。

这导致代码依赖“故意泄漏内存”维持字符串有效性。

新实现引入：

```cpp
std::vector<std::unique_ptr<char, FreeDeleter>> retained_profile_lines;
```

只有确实可能被配置结构引用的源行才转移到该所有权容器中。这样字符串生命周期是显式的，不再依赖注释和遗漏 `free()`。

### 5.6 Profile 条目统一复制

`profile_add()` 的接口改为：

```c
void profile_add(const char *str);
```

函数会创建可写副本并保存到 `ProfileEntry`：

```cpp
entry->data = strdup(str);
```

因此：

- 调用方始终保留输入字符串所有权；
- 字符串常量可以直接传入；
- 临时 `std::string::c_str()` 可以安全传入；
- 后续 `bind` 规则仍可以修改条目内部的可写副本。

相关动态调用点已经在复制后释放原字符串。

### 5.7 O(1) 追加 Profile 条目

旧实现每次追加都从链表头遍历到尾部：

```text
第一次 O(1)
第二次 O(1)
第 N 次 O(N)
总计接近 O(N²)
```

新实现维护：

```cpp
ProfileEntry *profile_tail;
```

后续追加为 O(1)。

### 5.8 条件语句解析

新增 `profile_condition.hpp`，将条件语法拆成纯函数：

```cpp
parse_conditional("?HAS_NET: net none")
```

返回：

```cpp
ConditionalLine {
    error,
    name,
    command
}
```

运行时仍支持原有条件：

```text
HAS_APPIMAGE
HAS_NET
HAS_NODBUS
HAS_NOSOUND
HAS_PRIVATE
HAS_X11
BROWSER_DISABLE_U2F
BROWSER_ALLOW_DRM
ALLOW_TRAY
```

条件命令生成的字符串同样进入集中所有权容器，避免悬空指针。

### 5.9 Profile 列表处理

旧实现依赖：

- 原地修改；
- 变长数组；
- 多层指针数组；
- GNU `?:` 扩展。

新增 `profile_list.hpp`，使用：

```cpp
std::vector<std::string>
std::string_view
std::string
```

保持以下语义：

```text
item    添加
+item   添加
-item   删除
=item   清空后添加
=       清空
```

示例：

```text
a,b,c,-a,a  -> b,c,a
a,b,=c,d    -> c,d
```

---

## 6. 构建系统调整

### 6.1 源文件依赖更新

语法文件生成规则已经从：

```text
profile.c
macros.c
```

更新为：

```text
profile.cpp
macros.cpp
macro_expander.hpp
```

### 6.2 去除 `gawk` 专属函数

原 `profile_conditionals.list` 生成规则使用：

```awk
gensub(...)
```

`gensub` 不是 POSIX awk 功能，使用 `mawk` 时会失败。

第三阶段将规则改为 POSIX `sub()`，已经验证只安装标准 `awk` 的环境也能从干净源码生成条件列表。

### 6.3 宏语法列表

宏定义移动到 `macro_expander.hpp` 后，构建规则会同时扫描：

```text
macros.cpp
macro_expander.hpp
```

生成结果包括：

```text
CFG
DESKTOP
DOCUMENTS
DOWNLOADS
HOME
MUSIC
PATH
PICTURES
RUNUSER
VIDEOS
```

---

## 7. 接口变化

### 7.1 修改

```c
void profile_add(char *str);
```

改为：

```c
void profile_add(const char *str);
```

新接口始终复制输入。

### 7.2 删除

```c
extern int lockfd_directory;
extern int lockfd_network;
```

锁状态现在完全封装在 `preproc.cpp` 中。

### 7.3 const 修正

```c
void fwarning(char *fmt, ...);
void fmessage(char *fmt, ...);
```

改为：

```c
void fwarning(const char *fmt, ...);
void fmessage(const char *fmt, ...);
```

格式字符串不会被修改，使用 `const char *` 更准确，也避免 C++ 字符串常量转换警告。

---

## 8. 测试覆盖

执行：

```bash
make cpp17-tests
```

当前测试：

```text
unique_fd_test
path_list_test
output_options_test
pipe_test
macro_expander_test
profile_list_test
profile_condition_test
```

新增覆盖包括：

- 宏识别；
- XDG 行解析；
- `${HOME}`、`${CFG}`、`${RUNUSER}` 和 `~` 展开；
- `$HOME` 拒绝；
- 条件语句合法和错误形式；
- Profile 列表去重、删除、重新添加和重置；
- 最大列表长度检查。

所有测试以：

```text
-std=c++17 -Wall -Wextra -Werror
```

编译。

另外执行了：

```text
AddressSanitizer
UndefinedBehaviorSanitizer
```

测试未发现错误。

---

## 9. 验证结果

第三阶段完成以下验证：

```text
干净 configure                     通过
完整 make                          通过
全部 C++17 模块 -Werror            通过
编译警告                           0
7 组 C++17 单元测试                通过
AddressSanitizer                    通过
UndefinedBehaviorSanitizer          通过
DESTDIR 模拟安装                    通过
firejail --version                  通过
语法列表重新生成                   通过
mawk/POSIX awk 构建                 通过
CLI-only 安装检查                  通过
```

当前执行环境本身位于容器或沙盒内，因此 Firejail 会检测到已有沙盒并跳过二次隔离。以下项目仍需在普通 Linux 虚拟机或物理机中回归：

```text
SUID 启动
Mount Namespace
PID Namespace
Network Namespace
真实 Seccomp 应用
Profile 对实际文件系统挂载的影响
SIGTSTP 锁竞争场景
```

---

## 10. 当前 C++17 迁移状态

主程序已经包含以下 C++17 实现：

```text
logger.cpp
run_files.cpp
paths.cpp
output.cpp
macros.cpp
preproc.cpp
profile.cpp
```

基础组件包括：

```text
unique_fd.hpp
pipe.hpp
path_list.hpp
output_options.hpp
macro_expander.hpp
profile_condition.hpp
profile_list.hpp
```

---

## 11. 第四阶段建议

下一阶段建议继续处理 Profile 数据模型，而不是立刻迁移网络或 Mount 核心：

1. 将 `ProfileEntry` 裸链表替换为 C++ `ProfileStore`；
2. 将字符串命令解析为类型化规则，例如：

```cpp
BlacklistRule
WhitelistRule
BindRule
TmpfsRule
ReadOnlyRule
DbusRule
```

3. 让 `fs.c`、`fs_whitelist.c` 和 `dbus.c` 通过只读迭代接口消费规则；
4. 消除 `split_comma()` 对 Profile 条目字符串的原地修改；
5. 将 `profile_check_line()` 的超长 `if/else` 分派逐步拆成独立处理器；
6. 为规则解析增加不需要 root 权限的集成测试。

这一步完成后，Profile 层才能真正摆脱 C 风格可变字符串和全局裸链表。
