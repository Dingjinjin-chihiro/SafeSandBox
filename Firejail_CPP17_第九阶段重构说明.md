# Firejail 0.9.80 CLI-only：现代 C++17 渐进式重构第九阶段

## 1. 阶段目标

第九阶段迁移 Firejail 的环境变量子系统。

第八阶段已经将 Profile 链表、Ignore 规则和长期字符串所有权迁移到 C++17 容器，但环境变量仍由 `env.c` 中的裸链表维护：

```c
typedef struct env_t {
    struct env_t *next;
    const char *name;
    const char *value;
    ENV_OP op;
} Env;
```

该实现存在以下问题：

1. 每条记录分别使用 `malloc/calloc + strdup` 分配；
2. 每次追加都遍历链表尾部，复杂度为 O(n)；
3. 节点和字符串没有统一释放责任；
4. `env_get()` 必须从头扫描全部历史操作；
5. `name` 与 `value` 通过修改同一块字符缓冲区进行切分，所有权隐含；
6. IBus 环境文件扫描依赖 `opendir/readdir/asprintf/fgets`；
7. 模块内部数据结构难以脱离完整 Firejail 进行单元测试。

本阶段保留现有 C 调用接口和环境变量操作顺序，将真实所有权迁移到现代 C++17/STL。

---

## 2. 文件变化

```text
src/firejail/env.c
    → src/firejail/env.cpp

新增：
src/firejail/environment_store.hpp

test/cpp17/environment_store_test.cpp
```

C++17 单元测试由第八阶段的 19 组增加到 20 组。

---

## 3. 类型化环境操作

新增命名空间：

```cpp
namespace firejail::environment;
```

环境操作不再使用裸整数状态，而是：

```cpp
enum class Operation {
    set,
    remove,
};
```

单条环境操作表示为：

```cpp
struct Entry {
    std::string name;
    std::string value;
    Operation operation;
};
```

解析结果使用：

```cpp
struct StoreResult {
    bool accepted;
    StoreError error;
};
```

这样可以区分：

- 空输入；
- 空变量名；
- 变量名包含 `=`；
- 设置操作缺少 `=`；
- 合法设置或删除操作。

---

## 4. 使用 `std::deque` 保持地址稳定

环境操作由：

```cpp
std::deque<Entry> entries_;
```

持有。

没有选择普通 `std::vector<Entry>`，原因是 C 接口：

```c
const char *env_get(const char *name);
```

会返回容器中 `std::string::c_str()` 的地址。后续调用 `env_store()` 时，如果使用 `std::vector` 并发生扩容，已有元素可能被移动，先前返回的地址会失效。

`std::deque` 在尾部追加元素时不会移动已有元素，因此适合这一渐进式 C/C++ 边界。

本阶段单元测试会：

1. 获取某个变量值的底层地址；
2. 再追加 1024 条环境操作；
3. 确认原地址仍能读取原值。

---

## 5. 追加复杂度从 O(n) 降为 O(1)

旧实现每次追加都执行：

```text
envlist → node → node → node → nullptr
```

然后把新节点挂到尾部。

当环境操作数量为 n 时，连续追加 n 条记录的总复杂度接近 O(n²)。

新实现直接执行：

```cpp
entries_.push_back(...);
```

单次追加为摊销 O(1)，连续追加为 O(n)。

---

## 6. 保持“最后一次操作生效”语义

Firejail 允许对同一个变量执行多次操作：

```text
set LANG=zh_CN.UTF-8
set LANG=en_US.UTF-8
remove LANG
```

新接口：

```cpp
std::optional<std::string_view>
latest_value(std::string_view name) const noexcept;
```

从容器尾部反向查找，因此：

- 最后一次是 `set`：返回对应值；
- 最后一次是 `remove`：返回 `std::nullopt`；
- 从未出现：返回 `std::nullopt`。

`env_apply_all()` 仍然按原始顺序执行全部操作，不改变设置与删除的实际执行语义。

---

## 7. 更严格的输入验证

设置操作：

```text
NAME=VALUE
```

现在明确要求：

- 输入非空；
- 必须存在第一个 `=`；
- `=` 左侧变量名非空；
- 变量名中不能再次包含 `=`；
- 值可以为空；
- 值可以继续包含 `=`。

例如：

```text
EMPTY=          合法
TOKEN=a=b=c     合法
NO_ASSIGNMENT   非法设置
=value          非法
```

删除操作要求传入纯变量名：

```text
rmenv DISPLAY       合法
rmenv DISPLAY=:0    非法
```

---

## 8. C ABI 保持不变

现有 C 模块仍然调用：

```c
void env_store(const char *str, ENV_OP op);
void env_store_name_val(const char *name, const char *val, ENV_OP op);
void env_apply_all(void);
void env_apply_whitelist(void);
void env_apply_whitelist_sbox(void);
void env_defaults(void);
const char *env_get(const char *name);
void env_ibus_load(void);
```

`env.cpp` 使用 `extern "C"` 导出这些符号，因此无需同步迁移：

```text
main.c
sandbox.c
x11.c
dbus.c
appimage.c
pulseaudio.c
netfilter.c
```

这保持了渐进式重构边界。

---

## 9. 白名单改为 `std::array<std::string_view>`

原来的 C 数组改为：

```cpp
constexpr std::array<std::string_view, 4> environment_whitelist;
constexpr std::array<std::string_view, 7> sbox_environment_whitelist;
```

匹配使用：

```cpp
std::find
```

环境值长度检查继续保留：

```text
name.length + value.length < env_max_len
```

主 Firejail 进程仍然只恢复：

```text
LANG
LANGUAGE
LC_MESSAGES
DISPLAY
```

并重新设置固定安全 PATH。

---

## 10. IBus 环境加载改用 `std::filesystem`

`env_ibus_load()` 不再手工维护：

```text
asprintf
opendir
readdir
fopen
fgets
free
closedir
```

现在使用：

```cpp
std::filesystem::path
std::filesystem::directory_iterator
std::ifstream
std::getline
std::error_code
```

扫描目录仍然是：

```text
~/.config/ibus/bus
```

只处理文件名以：

```text
unix-0
```

结尾的文件，并只加载以 `IBUS_` 开头且包含 `=` 的行。

使用 `std::error_code` 后，目录不存在或单个文件无法打开时仍然保持静默跳过的原行为。

---

## 11. C++ 异常边界

所有可能分配内存或使用 `std::filesystem` 的 C ABI 入口均在异常边界中执行：

```cpp
guard_environment_cpp(...)
```

处理方式：

- `std::bad_alloc`：设置 `errno = ENOMEM`，调用原有 `errExit()`；
- 标准异常：输出明确错误并终止；
- 未知异常：输出通用错误并终止；
- C++ 异常不会跨越 C ABI。

`env_apply_all()` 和 `env_get()` 的核心遍历路径不分配新容器内存。

---

## 12. 单元测试

新增：

```text
test/cpp17/environment_store_test.cpp
```

覆盖：

- 设置变量；
- 同名变量覆盖；
- 删除变量；
- 空字符串值；
- 值中包含多个 `=`；
- 缺少赋值号；
- 空变量名；
- 删除表达式包含 `=`；
- 直接接口中的非法变量名；
- 追加 1024 条记录后的地址稳定性；
- `clear()` 后状态复位。

当前测试数量：

```text
20/20
```

---

## 13. 验证结果

完整功能构建：

```bash
./configure \
    --prefix=/usr \
    CXXFLAGS="-g -O2 -Wall -Wextra -Werror"

make -j2
make cpp17-tests
```

结果：

```text
完整功能构建：通过
C++17 -Werror：通过
单元测试：20/20 通过
AddressSanitizer：通过
UndefinedBehaviorSanitizer：通过
firejail --version：通过
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

结果：

```text
最小功能构建：通过
C++17 -Werror：通过
单元测试：20/20 通过
```

模拟安装结果：

```text
安装文件：1443 个
firecfg：0
fzenity：0
firejail-welcome：0
```

版本输出：

```text
firejail version 0.9.80
```

当前构建环境本身位于已有沙盒中，因此没有把嵌套执行结果作为环境变量功能回归依据。真实 SUID、Namespace、Seccomp、X11 和完整应用环境继承仍需在普通 Linux 虚拟机或物理机中验证。

---

## 14. 第九阶段结果

第九阶段完成后，Firejail 的环境变量子系统具备以下特点：

```text
裸链表            → std::deque<Entry>
malloc/strdup      → std::string 所有权
O(n) 尾部追加      → O(1) 尾部追加
字符缓冲区切分     → std::string_view 解析
C 数组白名单       → std::array<std::string_view>
opendir/readdir    → std::filesystem
env_get 正向全扫   → 反向查找最后操作
隐式错误结果       → StoreResult + StoreError
```

外部命令行和 C 调用接口保持不变，为后续迁移 `checkcfg.c`、命令行环境参数和全局配置状态提供稳定基础。
