# Firejail 0.9.80 CLI-only：现代 C++17 渐进式重构第八阶段

## 1. 阶段目标

第八阶段处理 Profile 数据的所有权与跨模块访问方式。

前七个阶段已经把规则文本解析逐步迁移为 `std::variant`、`std::optional`、`std::string_view` 和 `std::from_chars`，但解析完成后的规则仍保存在传统裸链表中：

```c
Config cfg;
cfg.profile -> ProfileEntry -> ProfileEntry -> ...
```

该结构存在以下问题：

1. `ProfileEntry` 通过 `new + strdup` 分别分配，所有权分散；
2. 链表头保存在庞大的全局 `Config` 中，多个 C 模块直接读取和修改；
3. 尾指针由 `profile.cpp` 的另一个全局变量维护；
4. Profile ignore 规则保存在固定 `char *[32]` 中；
5. 长期配置字符串由多个 `malloc`、`strdup` 或读取缓冲区间接维持；
6. `/etc` 重建链表只被写入，从未被读取，持续产生无效分配。

本阶段的原则是：

- 保留 C 模块能够使用的 `ProfileEntry` 数据布局；
- 将节点和字符串的真实所有权收回 C++17 容器；
- 让遗留 C 模块通过明确访问器读取 Profile；
- 不在本阶段一次性改写 `fs.c`、`fs_whitelist.c` 和 `dbus.c`；
- 保持命令行、Profile 语法和安装布局不变。

---

## 2. 新增组件

```text
src/firejail/
├── profile_entry.h
├── profile_store.hpp
├── profile_ignore.hpp
└── stable_string_pool.hpp
```

新增测试：

```text
test/cpp17/
├── profile_store_test.cpp
├── profile_ignore_test.cpp
└── stable_string_pool_test.cpp
```

C++17 单元测试由第七阶段的 16 组增加到 19 组。

---

## 3. 提取 C 兼容的 Profile 节点定义

`TopDir` 和 `ProfileEntry` 从 `firejail.h` 中提取到：

```text
src/firejail/profile_entry.h
```

该文件保持纯 C 兼容：

```c
typedef struct profile_entry_t {
    struct profile_entry_t *next;
    char *data;

    struct wparam_t {
        char *file;
        char *link;
        TopDir *top;
    } *wparam;
} ProfileEntry;
```

这样可以同时满足：

- `fs.c`、`dbus.c`、`fs_whitelist.c` 继续以 C 结构访问节点；
- C++17 `EntryStore` 能直接创建和管理同一种节点；
- 单元测试无需包含完整的 `firejail.h`。

---

## 4. 使用 C++17 容器拥有 Profile 链表

新增：

```cpp
firejail::profile::EntryStore
```

内部所有权结构：

```cpp
std::vector<std::unique_ptr<EntryOwner>> entries_;
ProfileEntry *head_;
ProfileEntry *tail_;
```

每个 `EntryOwner` 同时拥有：

```cpp
ProfileEntry entry;
std::unique_ptr<char[]> buffer;
```

`entry.data` 指向 `buffer`，因此仍然是遗留 C 模块可使用的、以 `\0` 结尾的可写缓冲区。

追加规则：

```cpp
ProfileEntry *append(std::string_view command);
```

追加过程具有以下性质：

- 节点真实所有权由 `std::unique_ptr` 管理；
- `std::vector` 扩容只移动 `unique_ptr`，不会移动其指向的节点；
- `ProfileEntry *` 地址保持稳定；
- `head_` 和 `tail_` 在同一个对象中维护；
- 追加复杂度保持为 `O(1)`；
- `vector::push_back` 失败时不会把未拥有节点挂入链表。

旧实现中的：

```cpp
new ProfileEntry
strdup
profile_tail
cfg.profile
```

已经从 `profile.cpp` 中移除。

---

## 5. 移除 `Config` 对 Profile 主链表的所有权

`Config` 中已经删除：

```c
ProfileEntry *profile;
```

新增 C 兼容访问器：

```c
const ProfileEntry *profile_entries(void);
ProfileEntry *profile_entries_mutable(void);
size_t profile_entry_count(void);
```

用途划分：

- `dbus.c` 使用 `profile_entries()` 只读扫描规则；
- `fs.c` 使用 `profile_entries()` 只读应用黑名单和挂载规则；
- `fs_whitelist.c` 使用 `profile_entries_mutable()` 填充和释放 `wparam`；
- 只有 `profile.cpp` 能直接访问 `EntryStore`。

这减少了 Profile 存储与庞大全局配置结构之间的耦合。

---

## 6. bind 规则不再破坏存储文本

旧 `fs.c` 会直接调用：

```c
split_comma(entry->data + 5);
```

`split_comma()` 会把逗号改成 `\0`，永久修改 Profile 节点中的命令文本。

第八阶段改为：

```c
char *bind_spec = strdup(entry->data + 5);
char *dname2 = split_comma(bind_spec);
```

只修改临时副本，Profile Store 中的原始规则保持完整。

这使 Profile 主链表可以被视为加载完成后的稳定规则集合。

---

## 7. 使用 `std::vector<std::string>` 管理 ignore 规则

新增：

```cpp
firejail::profile::IgnoreList
```

内部使用：

```cpp
std::vector<std::string>
```

提供：

```cpp
bool add(std::string_view rule);
bool matches(std::string_view line) const noexcept;
bool full() const noexcept;
```

匹配语义保持不变：

```text
ignore apparmor
```

可以匹配：

```text
apparmor
apparmor firejail-default
```

但不会错误匹配：

```text
apparmor-replace
```

`Config` 中的：

```c
char *profile_ignore[MAX_PROFILE_IGNORE];
```

已删除。

最大 32 条规则的限制仍然保留。

---

## 8. 使用稳定字符串池统一长期 C 字符串

新增：

```cpp
firejail::StableStringPool
```

内部使用：

```cpp
std::deque<std::string>
```

接口：

```cpp
char *retain(std::string_view text);
```

选择 `std::deque` 的原因：

- 在尾部增加元素不会使既有元素引用失效；
- 字符串存入后不再修改；
- 返回的 `data()` 指针可以长期提供给遗留 C 配置字段；
- 不再需要为每个字符串单独 `malloc` 和维护自定义释放链表。

该字符串池现在负责保存：

- 条件规则展开后的文本；
- Sandbox 名称；
- AppArmor Profile；
- Capability 和 Seccomp 列表；
- 网络配置文本；
- X11 参数；
- 私有目录 CSV 列表；
- 从 Profile 文件读取并需要长期保留的规则文本。

原有 `retained_profile_lines` 和 `retain_profile_line()` 已删除。

---

## 9. 清理无效的 `/etc` 重建链表

旧 `Config` 中包含：

```c
ProfileEntry *profile_rebuild_etc;
```

`fs.c` 在成功屏蔽 `/etc/...` 路径后会向该链表持续追加节点，但整个项目中没有任何读取、遍历或释放该链表的代码。

第八阶段删除：

- `Config::profile_rebuild_etc`；
- 对应的 `malloc + strdup` 追加逻辑。

这不会改变运行行为，因为该数据此前从未参与任何后续处理，但可以消除无效分配和误导性状态。

---

## 10. Profile 入口进一步收敛

`profile.cpp` 行数变化：

```text
第七阶段：1810 行
第八阶段：1752 行
减少：58 行
```

主要减少内容：

- 固定 ignore 数组扫描；
- `strdup` 型 ignore 追加；
- 裸 `new ProfileEntry`；
- 手工尾指针恢复；
- `cfg.profile` 同步；
- Profile 长期字符串的 `malloc` 复制；
- `private-home` 的 `asprintf` 拼接。

---

## 11. 新增测试

### 11.1 `profile_store_test`

覆盖：

- 空 Store；
- 多节点追加；
- `head/next` 链接顺序；
- 可写 C 字符串缓冲区；
- 4096 次追加后的节点地址稳定性；
- `clear()` 后状态重置。

### 11.2 `profile_ignore_test`

覆盖：

- 空规则拒绝；
- 精确匹配；
- 以空格分隔的前缀匹配；
- 相似命令不误匹配；
- 容量上限。

### 11.3 `stable_string_pool_test`

覆盖：

- 字符串保留；
- 多次追加；
- 4096 次追加后旧 C 指针仍然有效；
- `clear()`。

---

## 12. 构建与验证

已验证：

```text
完整功能 C++17 -Werror 构建
最小功能裁剪 C++17 -Werror 构建
19/19 C++17 单元测试
AddressSanitizer
UndefinedBehaviorSanitizer
Profile 文件加载兼容测试
Profile 语法关键字列表回归比较
DESTDIR 模拟安装
最终压缩包独立解压重建
第七阶段补丁应用验证
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

安装结果仍为：

```text
1443 个文件
```

没有重新引入：

```text
firecfg
fzenity
firejail-welcome
```

语法列表与第七阶段逐项一致：

```text
profile_commands_arg0.list
profile_commands_arg1.list
profile_conditionals.list
```

---

## 13. 当前限制

当前环境位于已有沙盒中，因此不能替代普通 Linux 主机上的完整安全回归。

仍需要在虚拟机或物理机验证：

```text
SUID 安装与权限降级
Mount Namespace
PID Namespace
Network Namespace
Bridge/veth
iptables/nftables
Seccomp 实际拦截
AppArmor/Landlock 组合
长时间运行后的 Profile 节点生命周期
真实应用 Profile
```

---

## 14. 下一阶段建议

第九阶段可以迁移 Profile 的消费端：

1. 将 `fs.c` 中对规则文本的重复 `strncmp` 分派迁移为 C++17 类型化文件系统操作；
2. 用 `std::vector<std::string>` 替代 `noblacklist` 的 `char ** + realloc`；
3. 将 bind、blacklist、read-only、tmpfs 等操作转换为 `std::variant`；
4. 把 `ProfileEntry::wparam` 白名单准备状态迁移为独立 C++ 类型；
5. 最终让 Profile Store 保存类型化规则，而不仅是原始文本。
