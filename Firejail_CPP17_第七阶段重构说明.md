# Firejail 0.9.80 CLI-only：现代 C++17 渐进式重构第七阶段

## 1. 阶段目标

第七阶段不再继续增加彼此独立的 `parse_xxx_rule()` 调用，而是在前六个阶段已经形成的类型化规则解析器之上建立统一入口。

本阶段主要解决三个问题：

1. `profile_check_line()` 仍然按顺序调用多个解析器，控制流分散；
2. Resource Limit 仍在执行层使用旧的 `parse_arg_size()` 和可变 C 字符串；
3. `shell none` 在原类型表中被写成无法命中的精确字符串 `"shell "`。

核心原则：

- 一条 Profile 文本只进入一次统一解析流程；
- 使用 `std::variant` 表达所有规则类别；
- 使用 `std::visit` 进行执行层分派；
- 明确区分“无法识别”和“属于已知规则族但格式错误”；
- 使用 `std::from_chars` 和显式溢出检查解析 K/M/G 大小；
- Profile 与命令行复用同一套大小解析实现；
- 保持外部命令行和 Profile 语法不变。

---

## 2. 新增源码组件

```text
src/firejail/
├── profile_parser.hpp
├── scaled_size.hpp
└── size_parser.cpp
```

新增测试：

```text
test/cpp17/
├── profile_parser_test.cpp
└── scaled_size_test.cpp
```

C++17 单元测试由第六阶段的 14 组增加到 16 组。

---

## 3. 统一 Profile 解析入口

新增：

```cpp
using ProfileRule = std::variant<
    FlagRule,
    ValueRule,
    DeferredRule,
    SpecialRule,
    NetworkRule,
    SecurityRule,
    RuntimeRule,
    IsolationRule,
    RlimitRule,
    FilesystemRule,
    InvalidProfileRule,
    UnrecognizedProfileRule>;
```

统一入口：

```cpp
ProfileRule parse_profile_line(std::string_view line) noexcept;
```

解析优先级被集中在一个位置：

```text
基础精确规则
  ↓
网络规则
  ↓
安全策略规则
  ↓
运行时规则
  ↓
隔离规则
  ↓
Resource Limit
  ↓
延迟文件系统规则
  ↓
无法识别
```

这种顺序用于处理有重叠前缀的语法，例如：

```text
private          与 private DIRECTORY
private-cwd      与 private-cwd DIRECTORY
caps.drop all    与 caps.drop LIST
seccomp          与 seccomp.drop LIST
net none         与 net DEVICE
```

解析顺序现在由 `profile_parser.hpp` 明确表达，不再分散在 `profile_check_line()` 中。

---

## 4. 识别“格式错误”和“未知规则”

第六阶段中，各解析器一般返回：

```cpp
std::optional<Rule>
```

这只能表达：

- 匹配成功；
- 没有匹配。

但下面两行语义不同：

```text
rlimit-unknown 1
future-command value
```

前者明显属于 Resource Limit 规则族，但指令名错误；后者可能是完全未知的新语法。

第七阶段新增：

```cpp
enum class ProfileParseError {
    malformed_resource_limit,
};

struct InvalidProfileRule {
    ProfileParseError error;
    std::string_view text;
};

struct UnrecognizedProfileRule {
    std::string_view text;
};
```

这样执行层可以分别输出有针对性的 Resource Limit 错误和通用 Profile 行错误。

---

## 5. `profile_check_line()` 单次解析

旧结构：

```cpp
dispatch_typed_rule(...);
parse_network_rule(...);
parse_security_rule(...);
parse_runtime_rule(...);
parse_isolation_rule(...);
parse_rlimit_rule(...);
parse_filesystem_rule(...);
```

新结构：

```cpp
const auto parsed = firejail::profile::parse_profile_line(ptr);
const auto disposition = dispatch_profile_rule(parsed, ptr, lineno, fname);
```

执行层使用：

```cpp
std::visit(Overloaded{...}, parsed);
```

统一返回：

```cpp
enum class RuleDisposition {
    not_handled,
    consumed,
    store,
};
```

其中：

- `consumed`：规则已经立即生效；
- `store`：规则需要加入 Profile 链表，稍后在文件系统阶段处理；
- `not_handled`：规则无法识别，进入统一错误报告。

重构后，`profile.cpp` 中已经没有以 `ptr` 为输入的 `strcmp()`/`strncmp()` 规则分派链。

---

## 6. Resource Limit 执行器独立化

新增：

```cpp
RuleDisposition handle_rlimit_rule(const RlimitRule &rule);
```

负责应用：

```text
rlimit-as
rlimit-cpu
rlimit-fsize
rlimit-nofile
rlimit-nproc
rlimit-sigpending
```

普通无符号整数继续使用：

```cpp
std::from_chars
std::optional<unsigned long long>
```

地址空间与文件大小改用新的大小解析器，不再：

```cpp
const_cast<char *>(value)
parse_arg_size(char *)
sscanf(...)
```

---

## 7. 通用 K/M/G 大小解析器

新增：

```text
src/firejail/scaled_size.hpp
```

接口：

```cpp
std::optional<unsigned long long>
parse_scaled_size(std::string_view text) noexcept;
```

支持：

```text
1
4K
8k
64M
2g
```

倍率采用二进制单位：

```text
K = 1024
M = 1024²
G = 1024³
```

明确拒绝：

```text
空字符串
0
+1
-1
K
10KB
1.5G
尾随字符
乘法溢出
```

实现使用：

```cpp
std::from_chars
std::numeric_limits
std::optional
std::string_view
```

不会修改输入字符串，也不会依赖当前 C locale。

---

## 8. C ABI 兼容入口

`main.c` 仍然是 C 模块，因此保留现有函数名：

```c
unsigned long long parse_arg_size(const char *text);
```

它现在由：

```text
src/firejail/size_parser.cpp
```

实现，并使用：

```cpp
extern "C"
```

保持 C 链接名称。

旧实现已从 `util.c` 删除。Profile 和命令行的大小选项现在共享同一套 C++17 解析逻辑。

接口同时从：

```c
parse_arg_size(char *text)
```

收紧为：

```c
parse_arg_size(const char *text)
```

明确函数不会修改调用方数据。

---

## 9. 修复 `shell none` 遗留规则

第六阶段中的规则表包含：

```cpp
ExactSpec<SpecialDirective>{"shell ", ...}
```

但精确匹配要求输入与字符串完全相等，所以正常的：

```text
shell none
```

永远不会命中。

第七阶段修正为：

```cpp
ExactSpec<SpecialDirective>{"shell none", ...}
```

该规则现在会输出原有兼容警告：

```text
"shell none" is done by default now; the "shell" command has been removed
```

语法高亮命令列表仍保留 `shell`，与第六阶段生成结果一致。

---

## 10. 新增测试

### 10.1 `profile_parser_test`

覆盖：

- 基础 Flag 规则；
- `shell none` 特殊规则；
- 网络规则；
- Seccomp 安全规则；
- Runtime 规则；
- Isolation 规则；
- Resource Limit；
- 文件系统规则；
- 格式错误的 Resource Limit；
- 完全未知的规则。

### 10.2 `scaled_size_test`

覆盖：

- 无后缀字节数；
- 大小写 K/M/G；
- 零值；
- 正负号；
- 非法后缀；
- 小数；
- `unsigned long long` 最大值；
- 乘法溢出；
- C ABI `parse_arg_size()`；
- 空指针输入。

---

## 11. 构建与验证

验证项目：

```text
完整功能 C++17 -Werror 构建
最小功能裁剪 C++17 -Werror 构建
16/16 C++17 单元测试
AddressSanitizer
UndefinedBehaviorSanitizer
Profile 兼容启动测试
DESTDIR 模拟安装
语法关键字列表回归比较
最终压缩包独立解压重建
```

最小功能构建关闭：

```text
X11
D-Bus Proxy
private-home
network
Landlock
output
```

语法列表与第六阶段逐项一致：

```text
profile_commands_arg0.list
profile_commands_arg1.list
profile_conditionals.list
```

模拟安装仍为 1443 个文件，且不存在：

```text
firecfg
fzenity
firejail-welcome
```

---

## 12. 当前限制

当前执行环境本身位于受限沙盒中，因此只能完成编译、解析、嵌套启动兼容和安装清单验证，不能代替普通 Linux 主机上的完整安全回归。

仍应在虚拟机或物理机验证：

```text
SUID 安装和降权
Mount Namespace
PID Namespace
Network Namespace
Bridge/veth
iptables/nftables
Seccomp 实际拦截
AppArmor/Landlock 组合
真实桌面或服务器应用 Profile
```

---

## 13. 下一阶段建议

第八阶段可以处理 Profile 数据存储和应用状态：

1. 用 C++ 容器封装 `ProfileEntry` 链表；
2. 明确 Profile 规则节点与字符串的所有权；
3. 为 Profile 重建列表建立独立类型；
4. 减少 `cfg.profile` 在多个 C 模块中的直接写入；
5. 为 Profile 加载结果建立只读迭代接口。
