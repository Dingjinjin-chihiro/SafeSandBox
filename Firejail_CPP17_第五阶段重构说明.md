# Firejail 0.9.80 CLI-only：现代 C++17 渐进式重构第五阶段

## 1. 阶段目标

第五阶段继续拆分 `src/firejail/profile.cpp` 中遗留的大型字符串分派链，重点迁移两组高风险配置：

1. 网络配置规则；
2. 安全策略规则。

本阶段不改变 Firejail 的命令行使用方式，仍然可以使用原有 Profile 语法。内部实现从大量 `strcmp`、`strncmp`、固定指针偏移和原地字符串切割，改为类型化 C++17 解析器与独立执行函数。

核心原则：

- 解析层不直接修改全局状态；
- 执行层只接收类型化规则；
- 使用 `std::optional` 表达“不是该类规则”；
- 使用 `std::string_view` 避免无意义复制；
- 使用 `std::from_chars` 解析 MTU；
- 使用 `std::array` 和 STL 算法管理固定数量的网络槽位；
- 需要跨阶段持有的 C 字符串进入集中所有权池；
- 保持可选编译功能关闭时仍能通过严格构建。

---

## 2. 新增源码组件

```text
src/firejail/
├── profile_network_rule.hpp
└── profile_security_rule.hpp
```

新增测试：

```text
test/cpp17/
├── profile_network_rule_test.cpp
└── profile_security_rule_test.cpp
```

C++17 单元测试数量从第四阶段的 10 组增加到 12 组。

---

## 3. 网络规则解析器

`profile_network_rule.hpp` 定义：

```cpp
enum class NetworkDirective {
    netfilter_default,
    netfilter_file,
    netfilter6_file,
    netlock,
    netns,
    disable_network,
    attach_device,
    veth_name,
    ip_range,
    mac,
    mtu,
    netmask,
    ipv4,
    ipv6,
    default_gateway,
    dns,
};

struct NetworkRule {
    NetworkDirective directive;
    std::string_view value;
};
```

解析接口：

```cpp
std::optional<NetworkRule>
parse_network_rule(std::string_view line) noexcept;
```

已迁移规则：

```text
netfilter
netfilter FILE
netfilter6 FILE
netlock
netns NAME
net none
net DEVICE
veth-name NAME
iprange START,END
mac ADDRESS
mtu NUMBER
netmask ADDRESS
ip ADDRESS|none|dhcp
ip6 ADDRESS|dhcp
defaultgw ADDRESS
dns ADDRESS
```

### 3.1 MTU 类型安全解析

旧实现：

```c
sscanf(ptr + 4, "%d", &br->mtu)
```

新实现：

```cpp
std::optional<int> parse_mtu_value(std::string_view text) noexcept;
```

内部使用 `std::from_chars`，并一次性校验 Firejail 允许的范围：

```text
576 <= MTU <= 9198
```

因此能够严格拒绝：

```text
-1
575
9199
1500x
空字符串
```

### 3.2 IP 范围解析

旧实现会在原始 Profile 行中临时写入 `\0`，解析完成后再恢复逗号。

新接口：

```cpp
std::optional<std::pair<std::string_view, std::string_view>>
split_ip_range(std::string_view text) noexcept;
```

不会修改原始输入，也不会依赖可写缓冲区。

### 3.3 固定网络槽位的 STL 管理

Firejail 最多支持四个 Bridge 和四个 Interface。第五阶段用：

```cpp
std::array<Bridge *, 4>
std::array<Interface *, 4>
std::find_if
```

替代重复的：

```c
if (bridge0 ...)
else if (bridge1 ...)
else if (bridge2 ...)
else if (bridge3 ...)
```

`net none` 也通过范围循环统一清理已配置标记。

---

## 4. 安全策略规则解析器

`profile_security_rule.hpp` 定义统一的 `SecurityDirective`，覆盖 AppArmor、Seccomp、Landlock、Capability、协议族和 Namespace 限制。

已迁移规则：

```text
apparmor
apparmor PROFILE
apparmor-replace
apparmor-stack
allow-bwrap
protocol LIST

seccomp LIST
seccomp.32 LIST
seccomp.block-secondary
seccomp.drop LIST
seccomp.32.drop LIST
seccomp.keep LIST
seccomp.32.keep LIST
seccomp-error-action ACTION

landlock.enforce
landlock.fs.read PATH
landlock.fs.write PATH
landlock.fs.makeipc PATH
landlock.fs.makedev PATH
landlock.fs.execute PATH

memory-deny-write-execute
restrict-namespaces
restrict-namespaces LIST
caps.drop LIST
caps.keep LIST
```

解析接口：

```cpp
std::optional<SecurityRule>
parse_security_rule(std::string_view line) noexcept;
```

前缀规则按“更具体规则优先”排列。例如：

```text
seccomp.32.drop
seccomp.32.keep
seccomp.drop
seccomp.keep
seccomp.32
seccomp
```

避免通用 `seccomp ` 前缀抢先匹配更具体规则。

---

## 5. 集中字符串所有权

Firejail 的许多 C 结构仍保存 `char *`，并要求字符串在整个启动阶段保持有效。

第五阶段新增：

```cpp
char *retain_profile_text(std::string_view text);
```

它会：

1. 分配精确长度的可写 C 字符串；
2. 复制 `std::string_view` 内容；
3. 将所有权交给 `retained_profile_lines`；
4. 返回供旧 C 结构使用的稳定地址。

目前用于保存：

```text
网络 Namespace 名称
网络设备名
veth 名称
IPv6 地址
DNS 地址
AppArmor Profile 名称
Capability 列表
Seccomp 错误动作字符串
```

这比让全局字段直接指向临时解析缓冲区更明确，也避免在多个分支中散落 `strdup()`。

---

## 6. `profile.cpp` 的结构变化

第四阶段：

```text
profile.cpp：1973 行
```

第五阶段：

```text
profile.cpp：1851 行
```

虽然新增两个头文件共约 213 行，但大型分派函数净减少 122 行，并且解析规则已可脱离 Firejail 全局状态独立测试。

当前主分派流程：

```text
Profile 原始文本
       │
       ├── 条件规则解析
       ├── ignore 过滤
       ├── 基础类型化规则
       ├── 网络类型化规则
       ├── 安全策略类型化规则
       ├── 尚未迁移的旧规则
       ├── rlimit 类型化规则
       └── 文件系统延迟规则
```

核心调用：

```cpp
if (const auto network_rule = parse_network_rule(ptr)) {
    handle_network_rule(*network_rule, ptr, lineno, fname);
    return 0;
}

if (const auto security_rule = parse_security_rule(ptr)) {
    handle_security_rule(*security_rule, ptr, lineno, fname);
    return 0;
}
```

---

## 7. 语法文件生成修复

Profile 命令列表原先通过扫描 `profile.cpp` 中的 `strcmp/strncmp` 自动生成。规则迁移到头文件后，如果不修改构建规则，会丢失：

```text
netfilter
netlock
apparmor
apparmor-replace
apparmor-stack
allow-bwrap
landlock.enforce
memory-deny-write-execute
restrict-namespaces
seccomp.block-secondary
```

第五阶段将以下文件加入生成源：

```text
profile_rule.hpp
profile_filesystem_rule.hpp
profile_rlimit.hpp
profile_network_rule.hpp
profile_security_rule.hpp
```

并使用通用 `ExactSpec`/`PrefixSpec` 提取方式。

最终与第四阶段逐项比较：

```text
profile_commands_arg0.list：一致
profile_commands_arg1.list：一致
profile_conditionals.list：一致
```

---

## 8. 测试覆盖

当前 12 组 C++17 测试：

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
```

新增测试覆盖：

- 网络精确规则与前缀规则；
- IPv4/IPv6 参数提取；
- MTU 边界值与非法尾缀；
- IP 范围双值拆分；
- AppArmor 规则；
- 32 位 Seccomp 规则优先级；
- Landlock 路径规则；
- Namespace 限制规则；
- Capability 保留规则；
- 未知安全规则返回 `std::nullopt`。

---

## 9. 验证结果

### 9.1 默认完整功能构建

```bash
./configure \
    --prefix=/usr \
    CXXFLAGS="-g -O2 -Wall -Wextra -Werror"

make -j"$(nproc)"
make cpp17-tests
```

结果：

```text
完整编译：通过
C++17 -Werror：通过
单元测试：12/12 通过
```

### 9.2 Sanitizer

```text
AddressSanitizer：通过
UndefinedBehaviorSanitizer：通过
```

### 9.3 最小功能构建

验证参数：

```bash
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
最小功能编译：通过
C++17 -Werror：通过
单元测试：12/12 通过
```

这验证了新处理器不会在关闭 Network、Landlock 等可选宏后产生未使用变量或缺失符号。

### 9.4 安装验证

```text
DESTDIR 模拟安装：通过
安装文件：1443 个
CLI-only 检查：通过
firejail --version：通过
```

未重新引入：

```text
firecfg
fzenity
firejail-welcome
```

---

## 10. 环境限制

当前执行环境本身处于容器或沙盒中，Firejail 会检测到嵌套沙盒并跳过实际隔离。因此本阶段能够验证：

- Profile 解析器单元行为；
- 完整编译和链接；
- 可选模块组合；
- 安装清单；
- 程序基本启动。

但无法在当前环境完整验证：

- SUID 权限切换；
- 真实 Bridge/veth 创建；
- Network Namespace 加入；
- iptables/netfilter 应用；
- Seccomp 对目标程序的实际拦截；
- AppArmor/Landlock 与 Namespace 的完整组合。

这些项目应在普通 Linux 虚拟机或物理机中执行回归。

---

## 11. 下一阶段建议

第六阶段建议继续迁移以下 Profile 规则：

```text
hostname / hostname-randomize
hosts-file
cpu
nice
timeout
join-or-start
private-etc/private-opt/private-srv/private-bin/private-lib
bind
X11 模式规则
```

可以新增：

```text
profile_identity_rule.hpp
profile_runtime_rule.hpp
profile_private_rule.hpp
profile_x11_rule.hpp
```

并进一步把全局 `cfg` 写入集中到 `ProfileApplyContext`，逐步减少解析器对全局变量的直接依赖。
