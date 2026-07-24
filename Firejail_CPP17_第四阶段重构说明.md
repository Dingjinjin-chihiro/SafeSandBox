# Firejail 0.9.80 CLI-only：现代 C++17 渐进式重构（第四阶段）

## 一、阶段目标

第四阶段继续重构 `src/firejail/profile.cpp` 中接近两千行的 Profile 规则解析与分派逻辑。

本阶段没有一次性重写所有 Profile 指令，而是先把适合类型化、且能够独立测试的规则拆成三个解析层：

1. 基础规则和 D-Bus 规则解析；
2. 延迟文件系统规则解析；
3. Resource Limit 规则解析与数值转换。

保留尚未迁移的网络、Seccomp 高级规则、X11、私有目录、bind 等复杂处理逻辑，继续由原分派代码执行。

这样能够保证每个阶段都满足：

- 完整项目可以编译；
- Firejail 原有命令行仍可使用；
- 可选功能关闭后仍可编译；
- 新解析器可以脱离全局状态单元测试；
- 不需要一次性承担整个 Profile 系统重写风险。

---

## 二、第四阶段新增文件

```text
src/firejail/
├── profile_rule.hpp
├── profile_filesystem_rule.hpp
└── profile_rlimit.hpp

test/cpp17/
├── profile_rule_test.cpp
├── profile_filesystem_rule_test.cpp
└── profile_rlimit_test.cpp
```

修改文件：

```text
Makefile
src/firejail/profile.cpp
test/cpp17/Makefile
```

---

## 三、基础规则改为类型化 `std::variant`

### 3.1 原有问题

旧版 `profile_check_line()` 前半段由大量字符串判断组成：

```cpp
if (strncmp(ptr, "ignore ", 7) == 0) {
    // ...
}
else if (strcmp(ptr, "ipc-namespace") == 0) {
    // ...
}
else if (strcmp(ptr, "nonewprivs") == 0) {
    // ...
}
else if (strncmp(ptr, "dbus-user ", 10) == 0) {
    // ...
}
```

主要问题：

- 文本识别和业务执行混在一起；
- 前缀长度全部手工维护；
- `ptr + N` 容易出现偏移错误；
- 添加规则时必须继续扩张大型 `if/else` 链；
- 很难只测试“文本是否被正确识别”；
- 无法从类型上区分立即执行规则和延迟保存规则。

### 3.2 新的解析结果模型

第四阶段新增：

```cpp
using ParsedRule = std::variant<
    UnknownRule,
    FlagRule,
    ValueRule,
    DeferredRule,
    SpecialRule
>;
```

规则被划分为：

| 类型 | 含义 | 示例 |
|---|---|---|
| `FlagRule` | 无参数、直接修改状态 | `private-dev` |
| `ValueRule` | 携带一个参数、立即处理 | `name sandbox1` |
| `DeferredRule` | 校验后存入 Profile 链表 | `blacklist` 前的 mkdir、D-Bus 规则 |
| `SpecialRule` | 需要特殊检查或兼容逻辑 | `noroot`、`seccomp` |
| `UnknownRule` | 当前解析器不处理，交给旧分派器 | `net none` |

解析入口：

```cpp
ParsedRule parse_rule(std::string_view line) noexcept;
```

调用方使用 `std::visit`：

```cpp
const auto parsed = firejail::profile::parse_rule(line);

return std::visit(
    Overloaded{
        handle_unknown,
        handle_flag,
        handle_value,
        handle_deferred,
        handle_special,
    },
    parsed
);
```

### 3.3 本阶段迁移的基础规则

包括但不限于：

```text
ipc-namespace
nonewprivs
caps
caps.drop all
tracelog
private
tab
private-cwd
allusers
private-cache
private-dev
keep-dev-ntsync
keep-dev-shm
keep-dev-tpm
private-tmp
nogroups
nosound
noautopulse
notv
nodvd
novideo
no3d
noinput
nou2f
```

携带参数的规则：

```text
ignore
warn
keep-fd
xephyr-screen
name
private-home
private-cwd
dbus-user
dbus-system
```

特殊规则：

```text
noroot
seccomp
shell
noprinters
nodbus
notpm
```

---

## 四、D-Bus 规则拆分

D-Bus 规则被区分为两类。

### 4.1 立即修改策略

```text
dbus-user filter
dbus-user none
dbus-system filter
dbus-system none
```

解析后立即修改：

```text
arg_dbus_user
arg_dbus_system
```

并保留原来的策略收紧检查：一旦设置为 block，不能再通过 Profile 放宽为 filter。

### 4.2 校验后延迟保存

```text
dbus-user.see
dbus-user.talk
dbus-user.own
dbus-user.call
dbus-user.broadcast
dbus-system.see
dbus-system.talk
dbus-system.own
dbus-system.call
dbus-system.broadcast
```

名称类规则继续调用：

```cpp
dbus_check_name(...)
```

调用和广播规则继续调用：

```cpp
dbus_check_call_rule(...)
```

校验成功后返回：

```cpp
RuleDisposition::store
```

由上层保存到 Profile 规则链表，在 D-Bus 配置阶段应用。

---

## 五、统一规则处理结果

新增：

```cpp
enum class RuleDisposition {
    not_handled,
    consumed,
    store,
};
```

含义：

| 返回值 | 行为 |
|---|---|
| `not_handled` | 新解析器不认识，继续进入旧分派器 |
| `consumed` | 已立即执行，不写入 Profile 链表 |
| `store` | 已完成校验，需要保存并在后续阶段应用 |

原来的 `0/1` 返回语义只存在于 `profile_check_line()` 最外层；新 C++17 处理器内部不再使用含义不明确的整数。

---

## 六、文件系统延迟规则改为 `std::optional`

新增：

```cpp
std::optional<FilesystemRule>
parse_filesystem_rule(std::string_view line) noexcept;
```

支持：

```text
blacklist
blacklist-nolog
noblacklist
whitelist
nowhitelist
read-only
read-write
noexec
tmpfs
```

解析结果：

```cpp
struct FilesystemRule {
    FilesystemDirective directive;
    std::string_view path;
};
```

旧代码需要逐条修改裸指针：

```cpp
ptr += 10;
```

新代码直接取得：

```cpp
const auto rule = parse_filesystem_rule(line);
const std::string_view path = rule->path;
```

文件名安全检查仍然保留：

- 调用 `invalid_filename()`；
- 允许这些指令使用 glob；
- 拒绝包含 `..` 的路径；
- `whitelist` 继续设置 `arg_whitelist`。

---

## 七、Resource Limit 改用 `std::from_chars`

新增：

```cpp
std::optional<RlimitRule>
parse_rlimit_rule(std::string_view line) noexcept;
```

支持：

```text
rlimit-as
rlimit-cpu
rlimit-fsize
rlimit-nofile
rlimit-nproc
rlimit-sigpending
```

无符号整数不再通过：

```cpp
check_unsigned(...);
sscanf(..., "%llu", ...);
```

而是使用：

```cpp
const auto value =
    parse_unsigned_integer<unsigned long long>(text);
```

其内部使用：

```cpp
std::from_chars
```

优点：

- 不依赖区域设置；
- 不需要格式字符串；
- 可以确认整个字符串均被消费；
- 不会静默接受尾部垃圾字符；
- 通过 `std::optional` 明确表达失败。

`rlimit-as` 和 `rlimit-fsize` 仍使用原项目的 `parse_arg_size()`，继续支持：

```text
K
M
G
```

单位。

---

## 八、条件规则表改用 STL

旧条件表使用带空哨兵的 C 数组：

```cpp
Cond conditionals[] = {
    // ...
    {NULL, NULL}
};
```

新实现：

```cpp
const std::array conditionals{
    Conditional{"HAS_APPIMAGE", check_appimage},
    // ...
};
```

查找使用：

```cpp
std::find_if
```

不再依赖 `{NULL, NULL}` 哨兵。

当前条件包括：

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

---

## 九、Ignore 规则检查使用 `std::string_view`

旧实现使用：

```cpp
strlen
strncmp
ptr + len
```

新实现使用范围遍历和 `std::string_view`：

```cpp
for (const char *ignored : cfg.profile_ignore) {
    // 完整单词前缀匹配
}
```

继续保持原语义：

```text
ignore private
```

可以匹配：

```text
private
private something
```

但不会错误匹配：

```text
private-dev
```

除非忽略项本身就是对应完整命令前缀。

---

## 十、修复语法高亮生成与源码布局的耦合

Firejail 的 Profile 语法关键字列表原本通过扫描 `profile.cpp` 中的：

```text
strcmp
strncmp
```

自动生成。

当指令被移动到类型化头文件以后，仅扫描 `profile.cpp` 会丢失大量关键字。

第四阶段修改顶层 `Makefile`，现在同时扫描：

```text
src/firejail/profile.cpp
src/firejail/profile_rule.hpp
src/firejail/profile_filesystem_rule.hpp
src/firejail/profile_rlimit.hpp
```

并分别提取：

- 无参数规则；
- 有参数规则；
- 条件规则。

已将生成结果与第三阶段逐项比较：

```text
profile_commands_arg0.list：完全一致
profile_commands_arg1.list：完全一致
profile_conditionals.list：完全一致
```

这避免了重构 C++ 代码后，Vim 和 GtkSourceView 语法高亮悄然缺少命令的问题。

---

## 十一、测试扩展

第三阶段有 7 组 C++17 测试。

第四阶段增加 3 组：

```text
profile_rule_test
profile_filesystem_rule_test
profile_rlimit_test
```

现在共有 10 组：

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
```

运行：

```bash
make cpp17-tests
```

新测试覆盖：

- exact rule 识别；
- prefix rule 参数切分；
- `std::variant` 分支类型；
- 未迁移规则回退为 `UnknownRule`；
- 文件系统规则前缀优先级；
- `blacklist` 与 `blacklist-nolog` 不混淆；
- RLimit 类型识别；
- 正常无符号整数；
- 负数拒绝；
- 尾部字符拒绝。

---

## 十二、验证结果

### 12.1 默认完整功能构建

执行：

```bash
./configure \
    --prefix=/usr \
    CXXFLAGS="-g -O2 -Wall -Wextra -Werror"

make -j"$(nproc)"
make cpp17-tests
```

结果：

```text
完整项目构建：通过
C++17 -Werror：通过
C++17 单元测试：10/10 通过
firejail --version：通过
```

### 12.2 Sanitizer

执行：

```bash
make -C test/cpp17 clean
make -C test/cpp17 run \
    CXXFLAGS="-g -O1 -fsanitize=address,undefined -fno-omit-frame-pointer"
```

结果：

```text
AddressSanitizer：通过
UndefinedBehaviorSanitizer：通过
```

### 12.3 可选功能全部关闭的最小构建

执行：

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

make -j"$(nproc)"
make cpp17-tests
```

结果：

```text
最小功能构建：通过
C++17 -Werror：通过
单元测试：10/10 通过
```

这项测试用于确保 `#ifdef HAVE_X11`、`HAVE_DBUSPROXY`、`HAVE_PRIVATE_HOME` 等条件关闭后，新处理器不会产生未使用参数或缺失符号。

### 12.4 模拟安装

执行：

```bash
make DESTDIR=/tmp/firejail-phase4-stage install
```

结果：

```text
DESTDIR 安装：通过
安装文件数量：1443
```

CLI-only 检查通过，未重新引入：

```text
firecfg
fzenity
firejail-welcome
```

---

## 十三、当前迁移边界

第四阶段并没有宣称整个 `profile.cpp` 已经现代化。

已迁移：

```text
基础无参数规则
基础带参数规则
D-Bus 规则
mkdir/mkfile 延迟规则
文件系统 blacklist/whitelist 类规则
Resource Limit 规则
条件查找表
Ignore 前缀匹配
```

暂时保留在旧分派器：

```text
网络 Bridge、IP、MAC、MTU、netfilter
高级 Seccomp 规则
Capability 列表规则
Landlock 规则
X11 启动规则
private-etc/private-bin/private-lib 等复杂组合
bind
join-or-start
hostname、DNS、CPU affinity
部分环境变量和协议规则
```

这种边界是有意设计的：这些规则对全局状态、编译选项和外部辅助函数依赖更强，应当在后续阶段按领域拆分，而不是一次性搬迁。

---

## 十四、下一阶段建议

第五阶段建议优先处理以下两组。

### 14.1 网络规则处理器

拆分为：

```text
NetworkRuleParser
BridgeConfigBuilder
IpAddressParser
NetworkRuleHandler
```

可以使用：

```cpp
std::variant
std::optional
std::array
std::from_chars
std::error_code
```

重点消除：

```text
ptr + 固定偏移
重复 bridge0/1/2/3 判断
IP/MTU 的 sscanf
大量条件编译嵌套
```

### 14.2 Profile 规则容器

当前外部模块仍遍历：

```cpp
ProfileEntry *
```

第五或第六阶段可以建立：

```cpp
class ProfileRules {
public:
    void add(std::string rule);
    std::span<const std::string> rules() const;
};
```

由于项目使用 C++17，没有标准 `std::span`，可以使用：

- 迭代器接口；
- `const std::vector<std::string>&`；
- 自定义轻量视图。

然后逐步迁移：

```text
fs.c
fs_whitelist.c
dbus.c
```

最终删除裸链表和手工节点分配。

---

## 十五、构建命令

```bash
tar -xf firejail-0.9.80-cli-cpp17-phase4.tar.xz
cd firejail-0.9.80-cli-cpp17-phase4

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

完整 Namespace、Mount、SUID、Network 和 Seccomp 安全回归仍建议在普通 Linux 虚拟机或物理机中执行。当前构建环境本身位于受限容器中，不能证明所有内核隔离组合都已经完成真实运行验证。
