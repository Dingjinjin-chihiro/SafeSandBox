# Firejail 0.9.80 CLI-only：现代 C++17 渐进式重构第六阶段

## 1. 阶段目标

第六阶段继续收缩 `src/firejail/profile.cpp` 中遗留的字符串分派代码，重点迁移两组规则：

1. 运行时与进程配置规则；
2. X11、私有目录与 bind 隔离规则。

本阶段保持 Firejail 原有命令行和 Profile 使用方式不变，但内部不再依赖大量 `strcmp`、`strncmp`、固定下标、`atoi`、原地逗号切割和分散的 `asprintf` 拼接。

核心原则：

- 使用 `std::optional` 表达“当前文本不属于该规则组”；
- 使用 `enum class` 表达规则类型；
- 使用 `std::string_view` 进行零拷贝解析；
- 使用 `std::from_chars` 严格解析 `nice` 数值；
- 使用 `std::pair<std::string_view, std::string_view>` 解析 bind 两端路径；
- 不修改 Profile 原始输入缓冲区；
- 长期字符串统一进入集中所有权池；
- 保持完整功能和裁剪功能构建均能通过 `-Werror`。

---

## 2. 新增源码组件

```text
src/firejail/
├── profile_runtime_rule.hpp
└── profile_isolation_rule.hpp
```

新增测试：

```text
test/cpp17/
├── profile_runtime_rule_test.cpp
└── profile_isolation_rule_test.cpp
```

C++17 单元测试数量由第五阶段的 12 组增加到 14 组。

---

## 3. 运行时规则解析器

`profile_runtime_rule.hpp` 定义：

```cpp
enum class RuntimeDirective {
    environment_set,
    environment_remove,
    hostname,
    hostname_randomize,
    removed_keep_hostname,
    hosts_file,
    cpu_affinity,
    nice_value,
    writable_etc,
    machine_id,
    keep_config_pulse,
    keep_shell_rc,
    writable_var,
    keep_var_tmp,
    writable_run_user,
    writable_var_log,
    private_directory,
    allow_debuggers,
    timeout,
    join_or_start,
    disable_mnt,
    deterministic_exit_code,
    deterministic_shutdown,
};
```

统一解析接口：

```cpp
std::optional<RuntimeRule>
parse_runtime_rule(std::string_view line) noexcept;
```

已迁移规则：

```text
env NAME=VALUE
rmenv NAME
hostname NAME
hostname-randomize
keep-hostname
hosts-file FILE
cpu LIST
nice VALUE
writable-etc
machine-id
keep-config-pulse
keep-shell-rc
writable-var
keep-var-tmp
writable-run-user
writable-var-log
private DIRECTORY
allow-debuggers
timeout HH:MM:SS
join-or-start NAME
disable-mnt
deterministic-exit-code
deterministic-shutdown
```

### 3.1 `nice` 使用 `std::from_chars`

旧实现：

```c
cfg.nice = atoi(ptr + 5);
```

旧写法无法区分：

```text
nice 0
nice abc
nice 10x
```

因为失败时 `atoi()` 也返回 `0`。

新接口：

```cpp
std::optional<int>
parse_nice_value(std::string_view text) noexcept;
```

它能够严格拒绝：

```text
空字符串
10x
超出 int 范围的数字
```

同时保留原有权限语义：普通用户指定负 Nice 值时仍会被钳制为 `0`。

### 3.2 主机名和沙盒名的所有权

以下字段不再直接指向当前 Profile 行的子串：

```text
cfg.hostname
cfg.name
```

解析完成后会调用：

```cpp
retain_profile_text(rule.value)
```

将内容放入集中所有权池，避免后续 Profile 缓冲区生命周期变化产生悬空指针。

### 3.3 `join-or-start` 的控制流整理

原有逻辑保持不变：

1. 尝试根据名称查找现有沙盒；
2. 找到后直接执行 `join()`；
3. 未找到时将该名称设为新沙盒名称；
4. 功能被禁用时输出配置警告。

循环查找目标程序参数的代码改为明确的 `while`，不再使用带空循环体的 `for (...);` 写法。

---

## 4. 隔离规则解析器

`profile_isolation_rule.hpp` 定义：

```cpp
enum class IsolationDirective {
    x11_none,
    x11_xephyr,
    x11_xorg,
    x11_xpra,
    x11_xvfb,
    x11_auto,
    private_etc_default,
    private_etc_list,
    private_opt,
    private_srv,
    private_bin,
    private_lib_default,
    private_lib_list,
    bind,
};
```

统一解析接口：

```cpp
std::optional<IsolationRule>
parse_isolation_rule(std::string_view line) noexcept;
```

已迁移规则：

```text
x11 none
x11 xephyr
x11 xorg
x11 xpra
x11 xvfb
x11
private-etc
private-etc LIST
private-opt LIST
private-srv LIST
private-bin LIST
private-lib
private-lib LIST
bind SOURCE,TARGET
```

### 4.1 X11 分派类型化

X11 模式不再由六段独立 `strcmp()` 判断，而是先解析为 `IsolationDirective`，再由单个 `switch` 处理。

仍保留：

- `FIREJAIL_X11=yes` 时避免重复启动 X11 辅助进程；
- `x11 xorg` 只设置配置标记；
- Xephyr、Xpra、Xvfb 和自动模式成功启动后退出当前进程；
- 编译时关闭 X11 时仍能正常构建。

### 4.2 私有目录列表统一拼接

旧实现分别在五个分支中重复：

```c
asprintf(&cfg.xxx_private_keep, "%s,%s", ...)
```

新实现使用：

```cpp
append_profile_csv(char **target, std::string_view value)
```

统一处理：

```text
private-etc
private-opt
private-srv
private-bin
private-lib
```

拼接结果进入 `retained_profile_lines` 所有权池，地址在启动阶段保持稳定。

### 4.3 bind 不再修改原始字符串

旧实现：

```c
char *dname2 = split_comma(dname1);
// 临时把逗号改为 '\0'
...
*(dname2 - 1) = ',';
```

新接口：

```cpp
std::optional<
    std::pair<std::string_view, std::string_view>
>
split_bind_directories(std::string_view text) noexcept;
```

优势：

- 不修改输入缓冲区；
- 不需要恢复逗号；
- 两端为空时明确返回 `std::nullopt`；
- 第二个路径中仍可包含额外逗号，与旧版“只切第一个逗号”的行为一致；
- 路径合法性、`..` 和符号链接检查仍保留在执行层。

---

## 5. `profile.cpp` 规模变化

第五阶段：

```text
1851 行
```

第六阶段：

```text
1792 行
```

对应的运行时、X11、私有目录和 bind 旧分支已删除，不保留重复不可达代码。

当前 Profile 解析链路为：

```text
原始 Profile 文本
        │
        ├── 条件规则
        ├── Ignore 规则
        ├── 基础类型化规则
        ├── 网络类型化规则
        ├── 安全策略类型化规则
        ├── 运行时类型化规则
        ├── 隔离类型化规则
        ├── Resource Limit 规则
        └── 文件系统延迟规则
```

---

## 6. 语法高亮生成修复

构建系统的 Profile 关键字源列表新增：

```text
profile_runtime_rule.hpp
profile_isolation_rule.hpp
```

由于 `x11 none`、`x11 xephyr` 等属于“带参数形式的精确规则”，原自动提取正则无法可靠推导命令名，因此将 `x11` 作为显式关键字补充。

已与第五阶段逐项比较：

```text
profile_commands_arg0.list：一致
profile_commands_arg1.list：一致
profile_conditionals.list：一致
```

---

## 7. 单元测试

当前测试共 14 组：

```text
unique_fd_test
path_list_test
output_options_test
pipe_test
macro_expander_test
profile_list_test
profile_condition_test
profile_rule_test
profile_filesystem_rule_test
profile_rlimit_test
profile_network_rule_test
profile_security_rule_test
profile_runtime_rule_test
profile_isolation_rule_test
```

新增覆盖：

- 运行时精确规则与前缀规则；
- `hostname-randomize` 不接受多余参数；
- `nice` 正数、负数、非法后缀和溢出；
- X11 模式识别；
- 私有目录列表识别；
- bind 两端路径拆分；
- bind 第二端包含额外逗号；
- bind 缺少任一端时拒绝。

---

## 8. 验证结果

完整功能构建：

```text
./configure --prefix=/usr \
    CXXFLAGS="-g -O2 -Wall -Wextra -Werror"
make -j2
```

结果：

```text
完整项目编译：通过
C++17 -Werror：通过
C++17 单元测试：14/14 通过
AddressSanitizer：通过
UndefinedBehaviorSanitizer：通过
firejail --version：通过
DESTDIR 模拟安装：通过
CLI-only 安装检查：通过
```

裁剪功能构建：

```text
./configure \
    --prefix=/usr \
    --disable-x11 \
    --disable-dbusproxy \
    --disable-private-home \
    --disable-network \
    --disable-landlock \
    --disable-output \
    CXXFLAGS="-g -O2 -Wall -Wextra -Werror"
```

结果：

```text
最小功能构建：通过
C++17 -Werror：通过
单元测试：14/14 通过
```

模拟安装文件数量：

```text
1443
```

确认未重新引入：

```text
firecfg
fzenity
firejail-welcome
```

当前执行环境本身处于已有沙盒中，因此真实 SUID、Mount Namespace、Bridge/veth、iptables、AppArmor 和 Seccomp 组合仍需在普通 Linux 虚拟机或物理机中回归。

---

## 9. 编译方式

```bash
./configure --prefix=/usr
make -j"$(nproc)"
make cpp17-tests
```

模拟安装：

```bash
rm -rf stage
make DESTDIR="$PWD/stage" install
find stage -type f | sort
```

正式安装：

```bash
sudo make install
```

---

## 10. 下一阶段建议

第七阶段适合迁移：

1. `profile.cpp` 中 Resource Limit 执行逻辑；
2. `private-home` 列表的 `asprintf` 拼接；
3. `profile_add_ignore()` 的固定 C 数组管理；
4. Profile 规则链表改为 C++ 容器并在最终应用阶段提供只读遍历接口；
5. 将错误信息逐步改为结构化解析错误对象，降低解析器直接 `exit()` 的比例。
