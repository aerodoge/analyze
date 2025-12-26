# Rust 高级工程师面试指南

> 深入理解所有权、生命周期、并发模型和异步编程，助你通过高级Rust工程师面试

## 目录

- [1. 所有权系统 (Ownership)](#1-所有权系统-ownership)
- [2. 生命周期 (Lifetime)](#2-生命周期-lifetime)
- [3. 并发模型 (Concurrency)](#3-并发模型-concurrency)
- [4. 异步编程 (Async Programming)](#4-异步编程-async-programming)
- [5. 高级主题](#5-高级主题)
- [6. 面试真题与解答](#6-面试真题与解答)
- [7. 实战项目示例](#7-实战项目示例)

---

## 1. 所有权系统 (Ownership)

### 1.1 所有权三大规则

```
┌─────────────────────────────────────────────────────────────┐
│                   Rust 所有权核心规则                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  规则 1: 每个值都有一个所有者 (Owner)                           │
│         ├── 变量就是所有者                                    │
│         └── 值在内存中的唯一拥有者                              │
│                                                             │
│  规则 2: 同一时间只能有一个所有者                               │
│         ├── 防止double free                                 │
│         └── 编译时保证内存安全                                 │
│                                                             │
│  规则 3: 当所有者离开作用域，值被丢弃 (Drop)                     │
│         ├── 自动调用 Drop trait                              │
│         └── RAII (Resource Acquisition Is Initialization)   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Move 语义

```rust
// ❌ 错误示例：值被移动后不能再使用
fn ownership_move_error() {
    let s1 = String::from("hello");
    let s2 = s1; // s1 的所有权移动到 s2

    // println!("{}", s1); // ❌ 编译错误：value borrowed here after move
    println!("{}", s2); // ✅ 正确
}

// ✅ 正确理解：堆数据的 move
fn ownership_move_heap() {
    let s1 = String::from("hello");
    // String 结构：
    // Stack: ptr(指针), len(长度), capacity(容量)
    // Heap: "hello" 实际数据

    let s2 = s1; // 只复制栈上的指针，不复制堆数据
    // s1 失效，防止 double free
}

// ✅ 栈数据的 Copy
fn ownership_copy_stack() {
    let x = 5; // i32 实现了 Copy trait
    let y = x; // 完整复制值

    println!("x = {}, y = {}", x, y); // ✅ 两个都可以使用
}
```

**面试重点**：
1. **为什么String是move而i32是copy？**
   - `i32`完全在栈上，复制成本低，实现了`Copy`trait
   - `String`在堆上分配，复制成本高，只移动所有权

2. **哪些类型实现了Copy？**
   - 所有整数、浮点数类型
   - `bool`、`char`
   - 元组（如果所有元素都实现了Copy）
   - 不可变引用`&T`

### 1.3 借用规则 (Borrowing Rules)

```rust
// ✅ 不可变借用：多个读取者
fn immutable_borrow() {
    let s = String::from("hello");

    let r1 = &s; // ✅
    let r2 = &s; // ✅ 可以有多个不可变引用
    let r3 = &s; // ✅

    println!("{}, {}, {}", r1, r2, r3);
}

// ✅ 可变借用：唯一写入者
fn mutable_borrow() {
    let mut s = String::from("hello");

    let r1 = &mut s; // ✅ 唯一的可变引用
    r1.push_str(", world");

    println!("{}", r1);
}

// ❌ 错误示例：不能同时存在可变和不可变引用
fn borrow_error() {
    let mut s = String::from("hello");

    let r1 = &s;     // ✅ 不可变借用
    let r2 = &s;     // ✅ 不可变借用
    // let r3 = &mut s; // ❌ 编译错误：不能在有不可变借用时创建可变借用

    println!("{} and {}", r1, r2);
    // r1 和 r2 的作用域在这里结束

    let r3 = &mut s; // ✅ 现在可以了
    r3.push_str(", world");
}

// ✅ Non-Lexical Lifetimes (NLL) - Rust 2018+
fn nll_example() {
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    println!("{} and {}", r1, r2);
    // r1 和 r2 在最后一次使用后就失效了

    let r3 = &mut s; // ✅ 这里可以创建可变引用
    r3.push_str(", world");
}
```

**借用规则总结**：
```
┌─────────────────────────────────────────┐
│         借用规则 (Borrowing Rules)       │
├─────────────────────────────────────────┤
│                                         │
│  规则 1: 多个不可变引用 OR 一个可变引用      │
│         &T + &T + &T ... ✅             │
│         &mut T (唯一)    ✅              │
│         &T + &mut T      ❌             │
│                                         │
│  规则 2: 引用必须总是有效的                 │
│         ├── 不能引用已释放的内存           │
│         └── 不能创建悬垂引用               │
│                                         │
│  原因：防止数据竞争 (Data Race)            │
│  ├── 两个或多个指针同时访问同一数据          │
│  ├── 至少一个指针用于写入                  │
│  └── 没有同步访问数据的机制                 │
│                                         │
└─────────────────────────────────────────┘
```

### 1.4 所有权模式 (Ownership Patterns)

```rust
// 模式 1: 函数参数所有权转移
fn takes_ownership(s: String) {
    println!("{}", s);
} // s 在这里被 drop

fn takes_borrow(s: &String) {
    println!("{}", s);
} // 只是借用，不会 drop

fn ownership_transfer() {
    let s = String::from("hello");
    takes_ownership(s);
    // println!("{}", s); // ❌ s 已经被移动

    let s2 = String::from("world");
    takes_borrow(&s2);
    println!("{}", s2); // ✅ s2 仍然有效
}

// 模式 2: 返回值转移所有权
fn gives_ownership() -> String {
    String::from("hello")
}

fn takes_and_gives_back(s: String) -> String {
    s // 返回所有权
}

// 模式 3: Clone 显式复制
fn explicit_clone() {
    let s1 = String::from("hello");
    let s2 = s1.clone(); // 深拷贝堆数据

    println!("s1 = {}, s2 = {}", s1, s2); // ✅ 都有效
}

// 模式 4: 使用引用避免所有权转移
fn calculate_length(s: &String) -> usize {
    s.len()
} // s 离开作用域，但不会 drop（因为没有所有权）

// 模式 5: 可变引用修改值
fn change(s: &mut String) {
    s.push_str(", world");
}

fn reference_patterns() {
    let mut s = String::from("hello");
    change(&mut s);
    println!("{}", s); // "hello, world"
}
```

### 1.5 内部可变性 (Interior Mutability)

```rust
use std::cell::{Cell, RefCell};
use std::rc::Rc;

// Cell<T>: 适用于 Copy 类型
fn cell_example() {
    let x = Cell::new(42);

    let r1 = &x;
    let r2 = &x;

    r1.set(100); // ✅ 通过不可变引用修改值
    println!("{}", r2.get()); // 100
}

// RefCell<T>: 运行时借用检查
fn refcell_example() {
    let data = RefCell::new(String::from("hello"));

    {
        let mut r = data.borrow_mut(); // 可变借用
        r.push_str(", world");
    } // r 在这里释放

    let r = data.borrow(); // 不可变借用
    println!("{}", r); // "hello, world"
}

// ❌ RefCell 运行时 panic
fn refcell_panic() {
    let data = RefCell::new(String::from("hello"));

    let r1 = data.borrow_mut();
    // let r2 = data.borrow_mut(); // ❌ 运行时 panic: already borrowed
}

// Rc<RefCell<T>>: 共享可变所有权
fn rc_refcell_example() {
    let data = Rc::new(RefCell::new(vec![1, 2, 3]));

    let data1 = Rc::clone(&data);
    let data2 = Rc::clone(&data);

    data1.borrow_mut().push(4);
    data2.borrow_mut().push(5);

    println!("{:?}", data.borrow()); // [1, 2, 3, 4, 5]
}
```

**内部可变性对比**：
```
┌──────────────────────────────────────────────────────────┐
│              内部可变性类型对比                             │
├──────────────────────────────────────────────────────────┤
│ 类型          │ 检查时机  │ 适用场景        │ 开销      │
├──────────────┼──────────┼────────────────┼──────────┤
│ Cell<T>      │ 编译时    │ Copy 类型       │ 零开销    │
│ RefCell<T>   │ 运行时    │ 非 Copy 类型    │ 轻微开销  │
│ Mutex<T>     │ 运行时    │ 多线程共享      │ 较大开销  │
│ RwLock<T>    │ 运行时    │ 多读少写        │ 较大开销  │
└──────────────────────────────────────────────────────────┘
```

### 1.6 内部可变性应用场景

内部可变性是Rust中一个强大的模式，允许在拥有不可变引用的情况下修改数据。以下是常见的实际应用场景：

#### 场景 1: 缓存和记忆化

```rust
use std::cell::RefCell;
use std::collections::HashMap;

// 缓存计算结果（记忆化）
struct Cacher<T>
where
    T: Fn(u32) -> u32,
{
    calculation: T,
    cache: RefCell<HashMap<u32, u32>>, // 内部可变
}

impl<T> Cacher<T>
where
    T: Fn(u32) -> u32,
{
    fn new(calculation: T) -> Cacher<T> {
        Cacher {
            calculation,
            cache: RefCell::new(HashMap::new()),
        }
    }

    fn value(&self, arg: u32) -> u32 {
        // 注意：这里 self 是不可变引用
        let mut cache = self.cache.borrow_mut();

        match cache.get(&arg) {
            Some(&result) => result,
            None => {
                let result = (self.calculation)(arg);
                cache.insert(arg, result);
                result
            }
        }
    }
}

fn caching_example() {
    let expensive_calculation = |x: u32| {
        println!("Calculating for {}...", x);
        std::thread::sleep(std::time::Duration::from_secs(2));
        x * 2
    };

    let cacher = Cacher::new(expensive_calculation);

    println!("Result: {}", cacher.value(5)); // 计算并缓存
    println!("Result: {}", cacher.value(5)); // 直接从缓存获取
    println!("Result: {}", cacher.value(10)); // 计算新值
}
```

**为什么需要内部可变性**：
- `value` 方法只需要 `&self`，符合直觉（查询操作）
- 但内部需要修改缓存，这就需要内部可变性

#### 场景 2: 延迟初始化（Lazy Initialization）

```rust
use std::cell::OnceCell;

struct Configuration {
    // 延迟加载的配置，只初始化一次
    settings: OnceCell<HashMap<String, String>>,
}

impl Configuration {
    fn new() -> Self {
        Configuration {
            settings: OnceCell::new(),
        }
    }

    fn get_settings(&self) -> &HashMap<String, String> {
        // 第一次调用时初始化，之后直接返回
        self.settings.get_or_init(|| {
            println!("Loading configuration...");
            let mut map = HashMap::new();
            map.insert("host".to_string(), "localhost".to_string());
            map.insert("port".to_string(), "8080".to_string());
            map
        })
    }
}

fn lazy_init_example() {
    let config = Configuration::new();

    // 第一次访问，触发初始化
    println!("{:?}", config.get_settings());

    // 后续访问，直接返回
    println!("{:?}", config.get_settings());
}
```

#### 场景 3: 共享可变状态（单线程场景）

```rust
use std::cell::RefCell;
use std::rc::Rc;

// 树结构中的父子节点相互引用
struct Node {
    value: i32,
    children: RefCell<Vec<Rc<Node>>>,
    // 如果需要父节点引用，使用 Weak
    // parent: RefCell<Weak<Node>>,
}

impl Node {
    fn new(value: i32) -> Rc<Node> {
        Rc::new(Node {
            value,
            children: RefCell::new(vec![]),
        })
    }

    fn add_child(&self, child: Rc<Node>) {
        self.children.borrow_mut().push(child);
    }
}

fn tree_example() {
    let root = Node::new(1);
    let child1 = Node::new(2);
    let child2 = Node::new(3);

    root.add_child(child1);
    root.add_child(child2);

    println!("Root has {} children", root.children.borrow().len());
}
```

**为什么需要内部可变性**：
- `add_child` 只需要 `&self`（语义上不转移所有权）
- 但需要修改 `children` 列表
- `Rc<RefCell<T>>` 是常见组合模式

#### 场景 4: 观察者模式

```rust
use std::cell::RefCell;
use std::rc::Rc;

trait Observer {
    fn update(&self, message: &str);
}

struct Subject {
    observers: RefCell<Vec<Rc<dyn Observer>>>,
}

impl Subject {
    fn new() -> Self {
        Subject {
            observers: RefCell::new(vec![]),
        }
    }

    fn attach(&self, observer: Rc<dyn Observer>) {
        self.observers.borrow_mut().push(observer);
    }

    fn notify(&self, message: &str) {
        for observer in self.observers.borrow().iter() {
            observer.update(message);
        }
    }
}

struct ConcreteObserver {
    id: u32,
}

impl Observer for ConcreteObserver {
    fn update(&self, message: &str) {
        println!("Observer {} received: {}", self.id, message);
    }
}

fn observer_pattern_example() {
    let subject = Subject::new();

    let observer1 = Rc::new(ConcreteObserver { id: 1 });
    let observer2 = Rc::new(ConcreteObserver { id: 2 });

    subject.attach(observer1);
    subject.attach(observer2);

    subject.notify("Hello, observers!");
}
```

#### 场景 5: Mock 对象和测试

```rust
use std::cell::RefCell;

// 用于测试的 Mock HTTP 客户端
struct MockHttpClient {
    // 记录被调用的次数和参数
    call_log: RefCell<Vec<String>>,
}

impl MockHttpClient {
    fn new() -> Self {
        MockHttpClient {
            call_log: RefCell::new(vec![]),
        }
    }

    fn get(&self, url: &str) -> String {
        // 记录调用（内部可变性）
        self.call_log.borrow_mut().push(url.to_string());

        // 返回模拟响应
        format!("Mock response for {}", url)
    }

    fn verify_called(&self, url: &str) -> bool {
        self.call_log.borrow().contains(&url.to_string())
    }

    fn call_count(&self) -> usize {
        self.call_log.borrow().len()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_http_client() {
        let client = MockHttpClient::new();

        client.get("https://api.example.com/users");
        client.get("https://api.example.com/posts");

        assert_eq!(client.call_count(), 2);
        assert!(client.verify_called("https://api.example.com/users"));
    }
}
```

#### 场景 6: 引用计数中的可变数据

```rust
use std::cell::RefCell;
use std::rc::Rc;

// 共享的可变配置
struct AppConfig {
    settings: Rc<RefCell<HashMap<String, String>>>,
}

impl AppConfig {
    fn new() -> Self {
        AppConfig {
            settings: Rc::new(RefCell::new(HashMap::new())),
        }
    }

    fn clone_config(&self) -> Self {
        AppConfig {
            settings: Rc::clone(&self.settings),
        }
    }

    fn set(&self, key: &str, value: &str) {
        self.settings
            .borrow_mut()
            .insert(key.to_string(), value.to_string());
    }

    fn get(&self, key: &str) -> Option<String> {
        self.settings.borrow().get(key).cloned()
    }
}

fn shared_config_example() {
    let config1 = AppConfig::new();
    let config2 = config1.clone_config(); // 共享相同的配置

    config1.set("theme", "dark");
    config2.set("language", "en");

    // 两个实例看到相同的配置
    println!("Config1 theme: {:?}", config1.get("theme")); // Some("dark")
    println!("Config2 theme: {:?}", config2.get("theme")); // Some("dark")
    println!("Config1 lang: {:?}", config1.get("language")); // Some("en")
}
```

#### 场景 7: 图结构和循环引用

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

struct GraphNode {
    id: u32,
    // 强引用指向子节点
    neighbors: RefCell<Vec<Rc<GraphNode>>>,
    // 弱引用避免循环引用导致内存泄漏
    back_edges: RefCell<Vec<Weak<GraphNode>>>,
}

impl GraphNode {
    fn new(id: u32) -> Rc<Self> {
        Rc::new(GraphNode {
            id,
            neighbors: RefCell::new(vec![]),
            back_edges: RefCell::new(vec![]),
        })
    }

    fn add_neighbor(&self, node: Rc<GraphNode>) {
        self.neighbors.borrow_mut().push(node);
    }

    fn add_back_edge(&self, node: Weak<GraphNode>) {
        self.back_edges.borrow_mut().push(node);
    }
}

fn graph_example() {
    let node1 = GraphNode::new(1);
    let node2 = GraphNode::new(2);
    let node3 = GraphNode::new(3);

    // 构建图
    node1.add_neighbor(Rc::clone(&node2));
    node2.add_neighbor(Rc::clone(&node3));
    node3.add_back_edge(Rc::downgrade(&node1)); // 避免循环引用

    println!("Node 1 has {} neighbors", node1.neighbors.borrow().len());
}
```

#### 场景 8: 计数器和统计信息

```rust
use std::cell::Cell;

struct RequestHandler {
    request_count: Cell<u64>, // 使用 Cell 因为是 Copy 类型
}

impl RequestHandler {
    fn new() -> Self {
        RequestHandler {
            request_count: Cell::new(0),
        }
    }

    fn handle_request(&self, _req: &str) {
        // 增加计数（内部可变性）
        let count = self.request_count.get();
        self.request_count.set(count + 1);

        println!("Handling request #{}", count + 1);
    }

    fn total_requests(&self) -> u64 {
        self.request_count.get()
    }
}

fn counter_example() {
    let handler = RequestHandler::new();

    handler.handle_request("GET /users");
    handler.handle_request("POST /users");
    handler.handle_request("GET /posts");

    println!("Total requests: {}", handler.total_requests());
}
```

**内部可变性使用总结**：

```
┌─────────────────────────────────────────────────────────┐
│              何时使用内部可变性                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  使用场景：                                              │
│  ✅ 缓存/记忆化（不想暴露可变性给调用者）                  │
│  ✅ 延迟初始化（OnceCell, Lazy）                        │
│  ✅ 共享可变状态（单线程，Rc<RefCell<T>>）               │
│  ✅ 图结构/循环引用（需要运行时管理）                      │
│  ✅ 观察者模式（修改观察者列表）                          │
│  ✅ Mock 对象（记录调用信息）                            │
│  ✅ 计数器/统计（Cell<u64>）                             │
│                                                         │
│  不要使用：                                              │
│  ❌ 能用普通可变性解决的场景                              │
│  ❌ 多线程场景（用 Mutex/RwLock）                        │
│  ❌ 过度使用导致代码难以理解                              │
│                                                         │
│  选择指南：                                              │
│  • Copy 类型 → Cell<T>                                 │
│  • 非 Copy 类型 → RefCell<T>                           │
│  • 需要共享 → Rc<RefCell<T>>                           │
│  • 多线程 → Arc<Mutex<T>> / Arc<RwLock<T>>            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**面试要点**：
1. **理解动机**：为什么需要内部可变性？（绕过借用规则的限制）
2. **安全保证**：RefCell 如何保证安全？（运行时借用检查）
3. **性能考虑**：Cell 零开销，RefCell 轻微开销
4. **常见模式**：`Rc<RefCell<T>>` 用于共享可变状态
5. **与 Mutex 区别**：RefCell 单线程，Mutex 多线程

### 1.7 智能指针与引用计数（Rc vs Arc）

#### 为什么需要引用计数？

**Rust 的所有权规则问题**：

```rust
// 问题：一个值只能有一个所有者
fn problem_example() {
    let data = vec![1, 2, 3];

    let x = data;  // data 的所有权移动到 x
    // let y = data;  // ❌ 编译错误：value used after move

    // 问题：如果多个地方需要共享同一数据怎么办？
}

// 尝试用引用？也不行
fn reference_problem() {
    let data = vec![1, 2, 3];
    let x = &data;
    let y = &data;

    // 问题：data 必须比 x 和 y 活得更久
    // 如果想让数据动态共享，生命周期无法满足需求
}
```

**解决方案：引用计数（Reference Counting）**

```
核心思想：
- 多个所有者共同拥有数据
- 维护一个计数器记录所有者数量
- 当计数器降为 0 时释放数据

┌─────────────────────────────────────────────────────────────┐
│                     引用计数工作原理                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Rc::new(data)        →  计数 = 1                           │
│  Rc::clone(&rc1)      →  计数 = 2                           │
│  Rc::clone(&rc1)      →  计数 = 3                           │
│  rc2 离开作用域         →  计数 = 2                           │
│  rc3 离开作用域         →  计数 = 1                           │
│  rc1 离开作用域         →  计数 = 0  →  释放数据              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### Rc<T>：单线程引用计数

**设计目的**：
- 在单线程环境下共享数据
- 使用非原子操作，性能最优
- 无线程安全开销

**应用场景**：

```rust
use std::rc::Rc;
use std::cell::RefCell;

// 场景 1：图结构（多个节点指向同一节点）
struct GraphNode {
    id: usize,
    neighbors: Vec<Rc<GraphNode>>,  // 多个边指向同一节点
}

fn graph_example() {
    let node1 = Rc::new(GraphNode { id: 1, neighbors: vec![] });
    let node2 = Rc::new(GraphNode { id: 2, neighbors: vec![] });

    let node3 = Rc::new(GraphNode {
        id: 3,
        neighbors: vec![Rc::clone(&node1), Rc::clone(&node2)],
    });

    // node1 和 node2 被 node3 共享
    println!("node1 引用计数: {}", Rc::strong_count(&node1)); // 2
}

// 场景 2：UI 组件树（父子组件共享数据）
struct UIComponent {
    name: String,
    data: Rc<RefCell<AppState>>,  // 多个组件共享应用状态
}

struct AppState {
    theme: String,
    user: String,
}

fn ui_example() {
    let state = Rc::new(RefCell::new(AppState {
        theme: "dark".to_string(),
        user: "Alice".to_string(),
    }));

    let header = UIComponent {
        name: "Header".to_string(),
        data: Rc::clone(&state),
    };

    let sidebar = UIComponent {
        name: "Sidebar".to_string(),
        data: Rc::clone(&state),
    };

    // header 和 sidebar 共享同一状态
    state.borrow_mut().theme = "light".to_string();
    println!("引用计数: {}", Rc::strong_count(&state)); // 3
}

// 场景 3：不可变共享配置
struct Config {
    db_url: String,
    timeout: u64,
}

struct ServiceA {
    config: Rc<Config>,
}

struct ServiceB {
    config: Rc<Config>,
}

fn config_sharing() {
    let config = Rc::new(Config {
        db_url: "localhost:5432".to_string(),
        timeout: 30,
    });

    let service_a = ServiceA { config: Rc::clone(&config) };
    let service_b = ServiceB { config: Rc::clone(&config) };

    // 两个服务共享配置，避免重复拷贝
}
```

**Rc 的性能特点**：

```rust
use std::rc::Rc;

// Rc::clone() 只增加引用计数，不拷贝数据
let data = Rc::new(vec![1, 2, 3, 4, 5]);

// ✅ 高效：只拷贝指针 + 增加计数器（非原子操作）
let clone1 = Rc::clone(&data);  // ~5 纳秒
let clone2 = Rc::clone(&data);  // ~5 纳秒

// ❌ 低效：完整拷贝数据
// let clone = (*data).clone();  // 拷贝整个 Vec
```

#### Arc<T>：多线程原子引用计数

**设计目的**：
- 在多线程环境下安全共享数据
- 使用原子操作保证线程安全
- 实现 Send + Sync trait

**为什么不能在多线程中用 Rc？**

```rust
use std::rc::Rc;
use std::thread;

fn why_not_rc() {
    let data = Rc::new(vec![1, 2, 3]);

    // ❌ 编译错误！Rc 没有实现 Send
    // let handle = thread::spawn(move || {
    //     println!("{:?}", data);
    // });

    /*
    错误原因：
    1. Rc 的引用计数是普通整数，非原子操作
    2. 多线程同时 clone 会产生数据竞争：

    线程 1：读取计数 = 1
    线程 2：读取计数 = 1
    线程 1：写入计数 = 2
    线程 2：写入计数 = 2  ❌ 应该是 3！

    3. 导致计数错误 → 提前释放 / 内存泄漏
    */
}
```

**Arc 的解决方案**：

```rust
use std::sync::Arc;
use std::thread;
use std::sync::atomic::{AtomicUsize, Ordering};

// Arc 内部使用原子操作
struct ArcInternals<T> {
    data: T,
    strong_count: AtomicUsize,  // 原子计数器
}

impl<T> Arc<T> {
    fn clone(&self) -> Arc<T> {
        // 原子增加计数（线程安全）
        self.strong_count.fetch_add(1, Ordering::Relaxed);
        // 返回新的 Arc
    }
}
```

**Arc 的应用场景**：

```rust
use std::sync::Arc;
use std::sync::Mutex;
use std::thread;

// 场景 1：多线程共享只读数据
fn arc_readonly_example() {
    let config = Arc::new(vec![1, 2, 3, 4, 5]);

    let mut handles = vec![];

    for i in 0..5 {
        let config = Arc::clone(&config);

        let handle = thread::spawn(move || {
            println!("Thread {}: {:?}", i, config);
        });

        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}

// 场景 2：多线程共享可变数据（Arc + Mutex）
fn arc_mutable_example() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);

        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });

        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap()); // 10
}

// 场景 3：线程池共享工作队列
use std::sync::mpsc;

struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

type Job = Box<dyn FnOnce() + Send + 'static>;

impl ThreadPool {
    fn new(size: usize) -> ThreadPool {
        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            let receiver = Arc::clone(&receiver);  // Arc 共享接收端

            let thread = thread::spawn(move || loop {
                let job = receiver.lock().unwrap().recv().unwrap();
                job();
            });

            workers.push(Worker { id, thread });
        }

        ThreadPool { workers, sender }
    }
}

// 场景 4：异步运行时中的任务共享
use tokio::sync::Mutex as TokioMutex;

async fn async_example() {
    let data = Arc::new(TokioMutex::new(vec![]));

    let mut handles = vec![];

    for i in 0..10 {
        let data = Arc::clone(&data);

        let handle = tokio::spawn(async move {
            let mut vec = data.lock().await;
            vec.push(i);
        });

        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }
}
```

#### Rc vs Arc 对比

| 特性 | Rc<T> | Arc<T> |
|------|-------|--------|
| **线程安全** | ❌ 单线程 | ✅ 多线程 |
| **Send trait** | ❌ 不实现 | ✅ 实现 |
| **Sync trait** | ❌ 不实现 | ✅ 实现 |
| **计数器类型** | 普通整数 | AtomicUsize |
| **clone 成本** | ~5 纳秒 | ~20-30 纳秒（原子操作） |
| **适用场景** | 单线程数据共享 | 多线程数据共享 |
| **内存开销** | 16 字节（指针 + 计数） | 16 字节（指针 + 原子计数） |

**性能对比测试**：

```rust
use std::rc::Rc;
use std::sync::Arc;
use std::time::Instant;

fn benchmark_rc_vs_arc() {
    const ITERATIONS: usize = 1_000_000;

    // Rc clone 性能
    let rc = Rc::new(42);
    let start = Instant::now();
    for _ in 0..ITERATIONS {
        let _ = Rc::clone(&rc);
    }
    let rc_time = start.elapsed();

    // Arc clone 性能
    let arc = Arc::new(42);
    let start = Instant::now();
    for _ in 0..ITERATIONS {
        let _ = Arc::clone(&arc);
    }
    let arc_time = start.elapsed();

    println!("Rc:  {:?} ({:.2}ns per clone)", rc_time, rc_time.as_nanos() as f64 / ITERATIONS as f64);
    println!("Arc: {:?} ({:.2}ns per clone)", arc_time, arc_time.as_nanos() as f64 / ITERATIONS as f64);

    /*
    典型输出（x86-64）：
    Rc:  5.2ms  (5.2ns per clone)
    Arc: 28.6ms (28.6ns per clone)

    Arc 慢约 5-6 倍（原子操作开销）
    */
}
```

#### 为什么设计成两种类型？

**设计哲学：零成本抽象**

```
Rust 的核心原则：
"不为不需要的功能付出代价"

如果只有 Arc：
- 单线程程序也要承担原子操作开销
- 违背零成本抽象原则

如果只有 Rc：
- 多线程程序无法安全共享数据
- 编译器无法保证线程安全

因此设计成两种类型：
- Rc：单线程最优性能
- Arc：多线程安全保证

编译器通过 Send/Sync trait 在编译期检查，
防止在错误的场景使用错误的类型。
```

**编译器如何防止误用**：

```rust
use std::rc::Rc;
use std::sync::Arc;

// ❌ 编译错误：Rc 不能跨线程
fn misuse_rc() {
    let rc = Rc::new(42);

    // std::thread::spawn(move || {
    //     println!("{}", rc);
    // });

    // error[E0277]: `Rc<i32>` cannot be sent between threads safely
    // help: the trait `Send` is not implemented for `Rc<i32>`
}

// ✅ 正确：Arc 可以跨线程
fn correct_arc() {
    let arc = Arc::new(42);

    std::thread::spawn(move || {
        println!("{}", arc);
    });
}

// ✅ 正确：单线程用 Rc
fn correct_rc() {
    let rc = Rc::new(42);
    let rc2 = Rc::clone(&rc);
    // 单线程中使用，性能最优
}
```

#### 选择指南

```
┌─────────────────────────────────────────────────────────────┐
│                  Rc vs Arc 选择流程图                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  需要共享数据吗？                                              │
│      ├─ 否 → 使用 Box 或普通所有权                             │
│      └─ 是 ↓                                                │
│                                                             │
│         是否跨线程？                                          │
│         ├─ 否 → 使用 Rc<T>                                  │
│         └─ 是 → 使用 Arc<T>                                 │
│                                                             │
│  需要修改数据吗？                                              │
│      ├─ 否 → 直接使用 Rc<T> 或 Arc<T>                        │
│      └─ 是 ↓                                                │
│                                                             │
│         单线程：Rc<RefCell<T>>                               │
│         多线程：Arc<Mutex<T>> 或 Arc<RwLock<T>>              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**面试重点总结**：

1. **为什么需要引用计数？**
   - Rust 所有权规则限制一个值只能有一个所有者
   - 引用计数允许多个所有者共享数据

2. **为什么有 Rc 和 Arc 两种类型？**
   - Rc：单线程，普通计数器，性能最优
   - Arc：多线程，原子计数器，线程安全
   - 体现 Rust 零成本抽象原则

3. **性能差异？**
   - Rc clone：~5 纳秒
   - Arc clone：~25-30 纳秒（原子操作开销）

4. **如何防止误用？**
   - Rc 不实现 Send trait，编译器禁止跨线程传递
   - Arc 实现 Send + Sync trait，允许跨线程共享

5. **常见组合模式？**
   - 单线程可变：`Rc<RefCell<T>>`
   - 多线程可变：`Arc<Mutex<T>>` 或 `Arc<RwLock<T>>`

**基础示例**：

```rust
use std::rc::Rc;
use std::sync::Arc;

// Box<T>: 堆分配
fn box_example() {
    let b = Box::new(5);
    println!("b = {}", b);
} // b 离开作用域，Box 和值都被释放

// 递归类型必须用 Box
enum List {
    Cons(i32, Box<List>),
    Nil,
}

// Rc<T>: 引用计数（单线程）
fn rc_example() {
    let a = Rc::new(String::from("hello"));
    println!("count = {}", Rc::strong_count(&a)); // 1

    let b = Rc::clone(&a);
    println!("count = {}", Rc::strong_count(&a)); // 2

    {
        let c = Rc::clone(&a);
        println!("count = {}", Rc::strong_count(&a)); // 3
    }

    println!("count = {}", Rc::strong_count(&a)); // 2
}

// Arc<T>: 原子引用计数（多线程）
use std::thread;

fn arc_example() {
    let data = Arc::new(vec![1, 2, 3]);

    let mut handles = vec![];

    for i in 0..3 {
        let data = Arc::clone(&data);
        let handle = thread::spawn(move || {
            println!("Thread {}: {:?}", i, data);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

### 1.8 循环引用问题及解决方案

**问题说明**：使用 `Rc<T>` 或 `Arc<T>` 时，如果两个对象互相持有对方的强引用，会导致引用计数永远不为 0，造成内存泄漏。

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,
    prev: Option<Rc<RefCell<Node>>>, // ❌ 循环引用
}

fn memory_leak_example() {
    let node1 = Rc::new(RefCell::new(Node {
        value: 1,
        next: None,
        prev: None,
    }));

    let node2 = Rc::new(RefCell::new(Node {
        value: 2,
        next: None,
        prev: None,
    }));

    // 创建循环引用
    node1.borrow_mut().next = Some(Rc::clone(&node2));
    node2.borrow_mut().prev = Some(Rc::clone(&node1)); // 循环！

    // node1 和 node2 离开作用域时：
    // - node1 引用计数 = 1 (被 node2.prev 持有)
    // - node2 引用计数 = 1 (被 node1.next 持有)
    // ❌ 内存永远无法释放
}
```

**解决方案一：使用 Weak 指针（推荐）**

核心思想：打破循环，一方使用强引用（Rc），另一方使用弱引用（Weak）

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,      // 强引用
    prev: Option<Weak<RefCell<Node>>>,    // ✅ 弱引用
}

impl Node {
    fn new(value: i32) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(Node {
            value,
            next: None,
            prev: None,
        }))
    }
}

fn correct_example() {
    let node1 = Node::new(1);
    let node2 = Node::new(2);

    // 建立双向链接
    node1.borrow_mut().next = Some(Rc::clone(&node2));
    node2.borrow_mut().prev = Some(Rc::downgrade(&node1)); // Weak

    // 使用弱引用时需要 upgrade
    if let Some(prev_rc) = node2.borrow().prev.as_ref().and_then(|w| w.upgrade()) {
        println!("node2 的前驱值: {}", prev_rc.borrow().value);
    }

    // ✅ 离开作用域后正常释放：
    // 1. node1 强引用计数降为 0 → 释放
    // 2. node2 的 prev 是 Weak，不阻止释放
    // 3. node2 强引用计数降为 0 → 释放
}
```

**Weak 的关键特性**：
- `Rc::downgrade(&rc)` → 创建 Weak 指针
- `weak.upgrade()` → 返回 `Option<Rc<T>>`（如果对象已释放则为 None）
- Weak 不增加强引用计数，不阻止对象释放

**解决方案二：树结构的标准模式**

父节点持有子节点的强引用，子节点持有父节点的弱引用：

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct TreeNode {
    value: i32,
    children: Vec<Rc<RefCell<TreeNode>>>,    // 强引用
    parent: Option<Weak<RefCell<TreeNode>>>, // 弱引用
}

impl TreeNode {
    fn new(value: i32) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(TreeNode {
            value,
            children: Vec::new(),
            parent: None,
        }))
    }

    fn add_child(parent: &Rc<RefCell<Self>>, child: Rc<RefCell<Self>>) {
        child.borrow_mut().parent = Some(Rc::downgrade(parent));
        parent.borrow_mut().children.push(child);
    }
}

fn tree_example() {
    let root = TreeNode::new(1);
    let child1 = TreeNode::new(2);
    let child2 = TreeNode::new(3);

    TreeNode::add_child(&root, child1);
    TreeNode::add_child(&root, child2);

    // ✅ 当 root 释放时，children 也会被释放
    // child 的 parent 是 Weak，不会阻止 root 释放
}
```

**解决方案三：使用索引代替引用**

适用于所有节点存储在同一容器中的场景：

```rust
use std::cell::RefCell;

struct Graph {
    nodes: Vec<RefCell<GraphNode>>,
}

struct GraphNode {
    value: i32,
    neighbors: Vec<usize>, // ✅ 使用索引而非引用
}

impl Graph {
    fn add_edge(&mut self, from: usize, to: usize) {
        self.nodes[from].borrow_mut().neighbors.push(to);
    }

    fn get_neighbors(&self, index: usize) -> Vec<i32> {
        self.nodes[index]
            .borrow()
            .neighbors
            .iter()
            .map(|&i| self.nodes[i].borrow().value)
            .collect()
    }
}

fn graph_example() {
    let mut graph = Graph {
        nodes: vec![
            RefCell::new(GraphNode { value: 1, neighbors: vec![] }),
            RefCell::new(GraphNode { value: 2, neighbors: vec![] }),
        ],
    };

    graph.add_edge(0, 1);
    graph.add_edge(1, 0); // ✅ 循环图，但不会内存泄漏
}
```

**解决方案四：多线程场景使用 Arc + Weak**

```rust
use std::sync::{Arc, Weak};
use std::sync::Mutex;

struct AsyncNode {
    value: i32,
    next: Option<Arc<Mutex<AsyncNode>>>,
    prev: Option<Weak<Mutex<AsyncNode>>>, // Weak 打破循环
}

fn concurrent_example() {
    let node1 = Arc::new(Mutex::new(AsyncNode {
        value: 1,
        next: None,
        prev: None,
    }));

    let node2 = Arc::new(Mutex::new(AsyncNode {
        value: 2,
        next: None,
        prev: None,
    }));

    node1.lock().unwrap().next = Some(Arc::clone(&node2));
    node2.lock().unwrap().prev = Some(Arc::downgrade(&node1));

    // ✅ 正常释放
}
```

**检测循环引用**：

```rust
use std::rc::Rc;
use std::cell::RefCell;

fn detect_leak() {
    let node = Rc::new(RefCell::new(Node {
        value: 1,
        next: None,
        prev: None,
    }));

    println!("强引用计数: {}", Rc::strong_count(&node)); // 1
    println!("弱引用计数: {}", Rc::weak_count(&node));   // 0

    // 创建循环后
    node.borrow_mut().prev = Some(Rc::downgrade(&node));
    println!("弱引用计数: {}", Rc::weak_count(&node));   // 1
}
```

**最佳实践总结**：

| 场景 | 推荐方案 | 说明 |
|------|---------|------|
| 双向链表 | 一方 Rc，一方 Weak | prev 用 Weak |
| 树结构 | 父→子 Rc，子→父 Weak | parent 用 Weak |
| 图结构 | 使用索引 | 存储在 Vec 中 |
| 需要遍历 | Rc + Weak | 主方向用 Rc |
| 缓存/观察者 | Weak | 避免阻止释放 |

**核心原则**：
1. ✅ 优先考虑重新设计数据结构（避免循环）
2. ✅ 使用 Weak 打破循环（明确所有权方向）
3. ✅ 使用索引代替引用（适合容器场景）
4. ❌ 避免使用裸指针（除非必要且能保证安全）

**面试重点**：
- **为什么 Rc 会导致循环引用？** 因为强引用计数不为 0 时不会释放
- **Weak 和 Rc 的区别？** Weak 不增加强引用计数，不阻止对象释放
- **如何选择使用 Weak？** 明确所有权方向，反向引用用 Weak
- **索引方案的优缺点？** 优点是避免引用，缺点是需要统一容器管理

---

## 2. 生命周期 (Lifetime)

### 2.1 生命周期基础

```rust
// ❌ 悬垂引用（编译器阻止）
fn dangling_reference() {
    let r;
    {
        let x = 5;
        // r = &x; // ❌ `x` does not live long enough
    }
    // println!("r: {}", r);
}

// ✅ 正确的引用
fn valid_reference() {
    let x = 5;
    let r = &x;
    println!("r: {}", r);
}

// 生命周期注解语法
fn lifetime_annotation<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

/*
为什么必须标注生命周期？

问题：如果不标注会编译失败
----------------------------------------
fn lifetime_annotation(x: &str, y: &str) -> &str {
    // ❌ 编译错误：missing lifetime specifier
    // 编译器不知道返回值的生命周期是x还是y
}

原因分析：
1. 有两个引用参数 → 不满足生命周期省略规则
2. 返回值可能来自x，也可能来自y
3. 编译器必须知道返回值能活多久

生命周期标注的含义：
fn lifetime_annotation<'a>(x: &'a str, y: &'a str) -> &'a str
                      ^^^^    ^^          ^^          ^^
                       |       |           |           |
                       |       +-----+-----+           |
                       |             |                 |
    定义生命周期参数 ----+      输入约束        输出约束

含义：返回值的生命周期 <= min(x的生命周期, y的生命周期)
*/

// 使用示例：演示生命周期约束
fn use_lifetime_annotation() {
    let string1 = String::from("long string");
    let result;
    {
        let string2 = String::from("short");
        result = lifetime_annotation(&string1, &string2);
        println!("The longest string is {}", result);
        // result 的生命周期被限制在这个作用域内
        // 因为 string2 在这里结束，而 'a = min(string1, string2)
    }
    // ❌ 如果在这里使用 result 会出错
    // println!("{}", result); // string2 已经被释放，result 不再有效
}

// 对比：不需要标注生命周期的情况
fn no_lifetime_needed(x: &str) -> &str {
    // ✅ 只有一个引用参数，满足省略规则2
    // 编译器自动推断：fn no_lifetime_needed<'a>(x: &'a str) -> &'a str
    x
}

struct Context {
    data: String,
}

impl Context {
    fn get_data(&self) -> &str {
        // ✅ 有 &self，满足省略规则3
        // 编译器自动推断：fn get_data<'a>(&'a self) -> &'a str
        &self.data
    }
}
```

**生命周期的本质是什么？**

```
┌─────────────────────────────────────────────────────────────────┐
│           生命周期是"约束"而不是"关系"                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ 常见误解：                                                   │
│     "生命周期确定参数与返回值之间的关系"                          │
│                                                                 │
│  ✅ 正确理解：                                                   │
│     "生命周期是程序员对编译器的约束承诺"                          │
│                                                                 │
│  工作流程：                                                      │
│  ┌───────────────────────────────────────────────────────┐    │
│  │                                                       │    │
│  │  1. 程序员标注生命周期                                 │    │
│  │     → "我承诺返回值不会比输入活得更久"                 │    │
│  │                                                       │    │
│  │  2. 编译器记录这个约束                                 │    │
│  │     → 'a 必须满足某些条件                             │    │
│  │                                                       │    │
│  │  3. 在每个调用点验证                                   │    │
│  │     → 检查是否违反了约束                              │    │
│  │                                                       │    │
│  │  4. 违反约束 → 编译失败（防止悬垂引用）                 │    │
│  │                                                       │    │
│  └───────────────────────────────────────────────────────┘    │
│                                                                 │
│  关键点：                                                        │
│  • 生命周期是编译时静态检查                                      │
│  • 不产生任何运行时开销                                          │
│  • 描述的是"允许的使用范围"而非"实际的关系"                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```rust
// 深入例子：理解"约束"vs"关系"
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

/*
这个签名不是说：
❌ "x 和 y 必须有相同的生命周期"
❌ "返回值和 x、y 有某种关系"

而是说：
✅ "我承诺返回的引用不会比 x 和 y 中较短的那个活得更久"
✅ "编译器，请在调用点检查这个承诺"
*/

fn demonstrate_constraint() {
    let x = String::from("long string");

    {
        let y = String::from("short");
        let result = longest(&x, &y);

        // 编译器的推理过程：
        // - x 的生命周期：整个函数
        // - y 的生命周期：内部作用域
        // - 根据约束：'a = min(x的生命周期, y的生命周期)
        // - 因此：'a = 内部作用域
        // - 结论：result 只能在内部作用域使用

        println!("{}", result); // ✅ 满足约束
    } // result 在这里失效

    // println!("{}", result); // ❌ 违反约束：y 已被释放
    // 编译器利用生命周期标注检测到这个错误
}

// 例子：不同的约束表达不同的保证
fn only_depends_on_first<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    x // 只返回 x
}

fn use_different_lifetimes() {
    let x = String::from("hello");
    let result;

    {
        let y = String::from("world");
        result = only_depends_on_first(&x, &y);
    } // y 被释放

    println!("{}", result); // ✅ OK！
    // 因为约束说明返回值只依赖 'a（x），不依赖 'b（y）
}
```

**类比理解**：

```
生命周期标注 = 图书馆借书规则

fn borrow_book<'library>(book: &'library Book) -> &'library Book

不是："book 和返回值有关系"
而是："我承诺，返回的书必须在图书馆营业时间内归还"

编译器的工作：
- 检查你是否在闭馆前还书
- 如果你试图在闭馆后使用书，编译失败
```

**生命周期原理**：
```
┌─────────────────────────────────────────────────────────┐
│                   生命周期工作原理                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  生命周期参数 'a 表示：                                    │
│  ├── 程序员的承诺：返回值不会超出输入的最短生命周期         │
│  ├── 编译器的约束：'a <= min(所有标注为 'a 的输入)        │
│  └── 验证工具：在调用点检查是否违反约束                    │
│                                                         │
│  fn foo<'a>(x: &'a str, y: &'a str) -> &'a str          │
│                                                         │
│  含义：返回值的生命周期 <= min(x的生命周期, y的生命周期)    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 2.2 生命周期省略规则

```rust
// 规则 1: 每个引用参数都有自己的生命周期
fn rule1(s: &str) -> &str { s }
// 编译器推断为:
// fn rule1<'a>(s: &'a str) -> &'a str { s }

// 规则 2: 如果只有一个输入生命周期，它被赋给所有输出
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    &s[..]
}

// 规则 3: 如果有多个输入生命周期，且其中一个是 &self 或 &mut self
// 那么 self 的生命周期被赋给所有输出
struct ImportantExcerpt<'a> {
    part: &'a str,
}

impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }

    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
    // 编译器推断为:
    // fn announce_and_return_part<'a, 'b>(&'a self, announcement: &'b str) -> &'a str
}
```

### 2.3 复杂生命周期场景

```rust
// 场景 1: 结构体中的生命周期
struct Context<'a> {
    text: &'a str,
}

impl<'a> Context<'a> {
    fn new(text: &'a str) -> Self {
        Context { text }
    }

    fn get_text(&self) -> &'a str {
        self.text
    }
}

// 场景 2: 多个生命周期参数
fn longest_with_an_announcement<'a, 'b>(
    x: &'a str,
    y: &'a str,
    ann: &'b str,
) -> &'a str {
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

// 场景 3: 生命周期约束
fn longest_with_constraint<'a, 'b: 'a>(x: &'a str, y: &'b str) -> &'a str {
    // 'b: 'a 表示 'b 至少要活得和 'a 一样长
    x
}

// 场景 4: 静态生命周期
fn static_lifetime() {
    let s: &'static str = "I have a static lifetime.";
    // 字符串字面值直接存储在程序二进制文件中
}

// 场景 5: 生命周期子类型（Lifetime Subtyping）
fn subtype_example<'a, 'b: 'a>(x: &'a str, y: &'b str) -> &'a str {
    x // 'b 必须比 'a 活得更长
}

// 场景 6: 高阶生命周期（Higher-Ranked Trait Bounds - HRTB）
fn hrtb_example() {
    let closure = |x: &str| x.len();
    // 编译器推断为: for<'a> Fn(&'a str) -> usize
}

// 实际应用 HRTB
fn apply_to_str<F>(f: F, s: &str) -> usize
where
    F: for<'a> Fn(&'a str) -> usize,
{
    f(s)
}
```

### 2.4 生命周期与闭包

```rust
// 闭包捕获引用
fn closure_lifetime<'a>(s: &'a str) -> impl Fn() -> &'a str {
    move || s
}

// 复杂的闭包生命周期
fn complex_closure() {
    let string1 = String::from("hello");

    {
        let string2 = String::from("world");

        let closure = || {
            println!("{} {}", string1, string2);
        };

        closure();
    } // string2 在这里被释放

    // closure 不能在这里使用（如果它捕获了 string2 的引用）
}

// 返回闭包的生命周期
fn returns_closure() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}
```

---

## 3. 并发模型 (Concurrency)

### 3.1 线程基础

```rust
use std::thread;
use std::time::Duration;

// 基本线程创建
fn basic_thread() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }

    handle.join().unwrap(); // 等待线程完成
}

// move 闭包转移所有权
fn thread_move() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("Here's a vector: {:?}", v);
    });

    // println!("{:?}", v); // ❌ v 已经被移动

    handle.join().unwrap();
}
```

### 3.2 消息传递并发

```rust
use std::sync::mpsc; // multiple producer, single consumer
use std::thread;

// 基本通道使用
fn basic_channel() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
        // println!("{}", val); // ❌ val 已经被发送（移动）
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}

// 发送多个值
fn multiple_messages() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
    });

    for received in rx {
        println!("Got: {}", received);
    }
}

// 多个生产者
fn multiple_producers() {
    let (tx, rx) = mpsc::channel();

    let tx1 = tx.clone();
    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx1.send(val).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
    });

    thread::spawn(move || {
        let vals = vec![
            String::from("more"),
            String::from("messages"),
            String::from("for"),
            String::from("you"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```

### 3.3 共享状态并发

```rust
use std::sync::{Arc, Mutex, RwLock};
use std::thread;

// Mutex 基本使用
fn mutex_basic() {
    let m = Mutex::new(5);

    {
        let mut num = m.lock().unwrap();
        *num = 6;
    } // MutexGuard 在这里被 drop，锁被释放

    println!("m = {:?}", m);
}

// Arc + Mutex 多线程共享
fn arc_mutex() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}

// RwLock: 读写锁
fn rwlock_example() {
    let lock = RwLock::new(5);

    // 多个读取者
    {
        let r1 = lock.read().unwrap();
        let r2 = lock.read().unwrap();
        println!("r1 = {}, r2 = {}", *r1, *r2);
    }

    // 单个写入者
    {
        let mut w = lock.write().unwrap();
        *w += 1;
    }

    println!("lock = {:?}", lock);
}

// 死锁示例（反面教材）
fn deadlock_example() {
    let mutex1 = Arc::new(Mutex::new(1));
    let mutex2 = Arc::new(Mutex::new(2));

    let mutex1_clone = Arc::clone(&mutex1);
    let mutex2_clone = Arc::clone(&mutex2);

    let handle1 = thread::spawn(move || {
        let _guard1 = mutex1_clone.lock().unwrap();
        thread::sleep(Duration::from_millis(100));
        let _guard2 = mutex2_clone.lock().unwrap();
        // ❌ 可能死锁
    });

    let handle2 = thread::spawn(move || {
        let _guard2 = mutex2.lock().unwrap();
        thread::sleep(Duration::from_millis(100));
        let _guard1 = mutex1.lock().unwrap();
        // ❌ 可能死锁
    });

    handle1.join().unwrap();
    handle2.join().unwrap();
}
```

### 3.4 Send 和 Sync Trait

```
┌─────────────────────────────────────────────────────────┐
│                 Send 和 Sync Trait                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Send: 类型可以安全地在线程间转移所有权                    │
│  ├── 几乎所有类型都实现了 Send                           │
│  ├── Rc<T> 没有实现 Send（使用 Arc<T> 代替）             │
│  └── 裸指针没有实现 Send                                │
│                                                         │
│  Sync: 类型可以安全地在多个线程间共享引用                  │
│  ├── T 是 Sync ⇔ &T 是 Send                            │
│  ├── 基本类型都是 Sync                                  │
│  ├── Mutex<T> 和 Arc<T> 是 Sync                        │
│  └── Rc<T>、RefCell<T>、Cell<T> 不是 Sync              │
│                                                         │
│  关系：                                                 │
│  Send + Sync → 可以安全地跨线程使用                      │
│  !Send → 不能跨线程转移（如 Rc）                         │
│  !Sync → 不能跨线程共享引用（如 Cell）                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```rust
use std::marker::{Send, Sync};

// 自定义类型的 Send 和 Sync
struct MyBox<T> {
    value: T,
}

// 编译器自动实现（如果 T: Send）
// unsafe impl<T: Send> Send for MyBox<T> {}

// 手动实现 Sync
unsafe impl<T: Sync> Sync for MyBox<T> {}

// 验证类型是否实现了 Send/Sync
fn assert_send<T: Send>() {}
fn assert_sync<T: Sync>() {}

fn check_traits() {
    assert_send::<i32>();
    assert_sync::<i32>();

    assert_send::<Arc<i32>>();
    assert_sync::<Arc<i32>>();

    // assert_send::<Rc<i32>>(); // ❌ 编译错误
    // assert_sync::<Cell<i32>>(); // ❌ 编译错误
}
```

### 3.5 原子类型

```rust
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

// 原子计数器
fn atomic_counter() {
    let counter = Arc::new(AtomicUsize::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            for _ in 0..1000 {
                counter.fetch_add(1, Ordering::SeqCst);
            }
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", counter.load(Ordering::SeqCst));
}

// 原子布尔标志
fn atomic_flag() {
    let flag = Arc::new(AtomicBool::new(false));
    let flag_clone = Arc::clone(&flag);

    let handle = thread::spawn(move || {
        thread::sleep(Duration::from_secs(1));
        flag_clone.store(true, Ordering::SeqCst);
    });

    while !flag.load(Ordering::SeqCst) {
        println!("Waiting...");
        thread::sleep(Duration::from_millis(100));
    }

    println!("Flag is set!");
    handle.join().unwrap();
}
```

**内存顺序 (Memory Ordering)**:
```
┌─────────────────────────────────────────────────────────┐
│              Atomic 内存顺序                             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Relaxed: 最弱的顺序保证，只保证原子性                     │
│  ├── 性能最好                                           │
│  └── 适用于简单计数器                                    │
│                                                         │
│  Acquire: 读操作，禁止后续的读写重排到此操作之前           │
│  Release: 写操作，禁止之前的读写重排到此操作之后           │
│  AcqRel: 读-修改-写操作，结合 Acquire 和 Release         │
│                                                         │
│  SeqCst: 最强的顺序保证，全局顺序一致                     │
│  ├── 性能开销最大                                        │
│  └── 最容易理解，默认选择                                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 4. 异步编程 (Async Programming)

### 4.1 异步基础概念

```
┌─────────────────────────────────────────────────────────┐
│              同步 vs 异步 vs 并发 vs 并行                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  同步 (Synchronous):                                    │
│  Task1 ━━━━━━━━━━━━━━━━━━━━→                          │
│                                Task2 ━━━━━━━━━→        │
│  顺序执行，阻塞等待                                      │
│                                                         │
│  异步 (Asynchronous):                                   │
│  Task1 ━━━━━━━━━━━━━━━━━━━━→                          │
│  Task2       ━━━━━━━━━━━━━━━━→                        │
│  非阻塞，任务可以交错执行                                 │
│                                                         │
│  并发 (Concurrent):                                     │
│  Task1 ━━ ━━ ━━ ━━                                     │
│  Task2   ━━ ━━ ━━                                      │
│  多个任务在同一时间段内执行（可能在单核上）                 │
│                                                         │
│  并行 (Parallel):                                       │
│  Task1 ━━━━━━━━━━━━━━━━  (Core 1)                     │
│  Task2 ━━━━━━━━━━━━━━━━  (Core 2)                     │
│  多个任务在同一时刻真正同时执行（多核）                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Future 和 async/await

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// 基本的 async 函数
async fn hello_world() {
    println!("Hello, world!");
}

// async 函数返回 Future
async fn async_function() -> i32 {
    42
}

// 使用 .await
async fn use_async() {
    let result = async_function().await;
    println!("Result: {}", result);
}

// 手动实现 Future
struct MyFuture {
    count: u32,
}

impl Future for MyFuture {
    type Output = u32;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        self.count += 1;

        if self.count < 3 {
            cx.waker().wake_by_ref(); // 请求再次轮询
            Poll::Pending
        } else {
            Poll::Ready(self.count)
        }
    }
}

// 并发执行多个 Future
async fn concurrent_futures() {
    use futures::join;

    let future1 = async {
        println!("Future 1");
        1
    };

    let future2 = async {
        println!("Future 2");
        2
    };

    let (result1, result2) = join!(future1, future2);
    println!("Results: {} and {}", result1, result2);
}
```

#### 4.2.1 从操作系统层面理解异步和Future

**核心问题：为什么需要异步？**

从操作系统视角看，主要解决的是**I/O等待时的CPU浪费问题**。

```
传统同步阻塞模型的问题：
┌──────────────────────────────────────────────────────────────┐
│ Thread 1                                                     │
│ ━━━━━━━ 执行代码 ━━━━━━━                                      │
│                         ⏸ 等待磁盘 I/O (阻塞) ⏸              │
│                                                  ━━━━ 继续    │
│                                                               │
│ 问题：线程阻塞期间，CPU 完全空闲！                              │
│ 解决：创建更多线程 → 但线程有成本！                             │
└──────────────────────────────────────────────────────────────┘

线程的成本（Linux 示例）：
- 栈内存：默认 8MB/线程
- 上下文切换：约 1-2 微秒
- 1 万个线程 = 80GB 内存（不可接受）

异步模型的优势：
┌──────────────────────────────────────────────────────────────┐
│ 单个线程执行多个任务（协作式调度）                              │
│                                                               │
│ Task1 ━━━━━ [I/O] ━━━━━ [I/O] ━━━━                          │
│ Task2       ━━━━        ━━━━                                 │
│ Task3            ━━━━        ━━━━                            │
│                                                               │
│ 优势：                                                        │
│ - 内存占用小（每个 Future ~100 字节）                          │
│ - 无上下文切换成本                                             │
│ - 1 万个任务 = ~1MB 内存                                      │
└──────────────────────────────────────────────────────────────┘
```

**操作系统 I/O 模型演进**：

```rust
// 1. 阻塞 I/O (Blocking I/O) - 传统模型
fn blocking_io() {
    let data = read_from_disk(); // ⏸ 线程阻塞，等待系统调用完成
    process(data);
}
// 系统调用：read() 阻塞直到数据就绪
// 问题：一个线程只能处理一个连接

// 2. 非阻塞 I/O (Non-blocking I/O)
fn non_blocking_io() {
    loop {
        match try_read_from_disk() {
            Ok(data) => {
                process(data);
                break;
            }
            Err(WouldBlock) => {
                // 继续轮询（CPU 空转）
                continue;
            }
        }
    }
}
// 系统调用：read() 设置 O_NONBLOCK 标志
// 问题：轮询浪费 CPU

// 3. I/O 多路复用 (I/O Multiplexing) - 现代异步基础
fn io_multiplexing() {
    let epoll_fd = epoll_create(); // Linux
    epoll_ctl(epoll_fd, ADD, socket1);
    epoll_ctl(epoll_fd, ADD, socket2);

    loop {
        let events = epoll_wait(epoll_fd); // ⏸ 阻塞，直到任意 fd 就绪
        for event in events {
            handle(event); // 处理就绪的 I/O
        }
    }
}
// 系统调用：epoll (Linux), kqueue (BSD/macOS), IOCP (Windows)
// 优势：一个线程监控多个 I/O 源
```

**Rust Future 的本质：零成本状态机**

```rust
// 这个异步函数：
async fn example() {
    let data1 = read_file("a.txt").await;  // 等待点 1
    let data2 = read_file("b.txt").await;  // 等待点 2
    process(data1, data2);
}

// 编译器转换为状态机：
enum ExampleStateMachine {
    State0,  // 初始状态
    State1 { data1: String },  // 完成第一个 await
    State2 { data1: String, data2: String },  // 完成第二个 await
    Finished,
}

impl Future for ExampleStateMachine {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context) -> Poll<()> {
        loop {
            match *self {
                State0 => {
                    // 尝试读取 a.txt
                    match read_file("a.txt").poll(cx) {
                        Poll::Ready(data1) => {
                            *self = State1 { data1 };
                            continue;  // 立即进入下一状态
                        }
                        Poll::Pending => return Poll::Pending,  // I/O 未就绪
                    }
                }
                State1 { ref data1 } => {
                    // 尝试读取 b.txt
                    match read_file("b.txt").poll(cx) {
                        Poll::Ready(data2) => {
                            *self = State2 { data1: data1.clone(), data2 };
                            continue;
                        }
                        Poll::Pending => return Poll::Pending,
                    }
                }
                State2 { ref data1, ref data2 } => {
                    process(data1, data2);
                    *self = Finished;
                    return Poll::Ready(());
                }
                Finished => panic!("polled after completion"),
            }
        }
    }
}
```

**关键概念**：
1. **状态机在栈上分配**（或内联），无堆分配
2. **Poll::Pending** = I/O 未就绪，任务让出 CPU
3. **Poll::Ready** = 任务完成，返回结果
4. **Waker** = 当 I/O 就绪时唤醒任务

**Tokio 运行时架构（与操作系统的交互）**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Tokio 异步运行时                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                │
│  │  Worker    │  │  Worker    │  │  Worker    │  <- 线程池     │
│  │  Thread 1  │  │  Thread 2  │  │  Thread N  │                │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘                │
│         │                │                │                      │
│         └────────────────┴────────────────┘                      │
│                          │                                       │
│                  ┌───────▼────────┐                              │
│                  │  Task Scheduler │  <- 任务调度器              │
│                  │  (Work Stealing)│     (协作式调度)             │
│                  └───────┬────────┘                              │
│                          │                                       │
│         ┌────────────────┼────────────────┐                      │
│         │                │                │                      │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐             │
│  │   Task 1    │  │   Task 2    │  │   Task N    │  <- Future  │
│  │  (Future)   │  │  (Future)   │  │  (Future)   │     状态机  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                      │
│         └────────────────┴────────────────┘                      │
│                          │                                       │
│                  ┌───────▼────────┐                              │
│                  │  I/O Reactor   │  <- 事件循环                 │
│                  │  (mio 库)      │                              │
│                  └───────┬────────┘                              │
└──────────────────────────┼─────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼───────┐  ┌───────▼───────┐  ┌──────▼──────┐
│ epoll (Linux) │  │ kqueue (BSD)  │  │ IOCP (Win)  │ <- 操作系统
└───────────────┘  └───────────────┘  └─────────────┘
```

**详细工作流程**：

```rust
// 示例：异步网络 I/O
async fn handle_connection(socket: TcpStream) {
    let mut buf = [0; 1024];

    // 1. 调用 read() - 注册到 I/O Reactor
    let n = socket.read(&mut buf).await;  // await 点

    // ... 处理数据
}

// 底层发生的事情：
//
// 1. socket.read() 返回一个 Future
//
// 2. 首次 poll 这个 Future：
//    - 调用系统调用 read(fd, buf, len)
//    - 如果返回 EAGAIN/EWOULDBLOCK (未就绪)：
//      ├─ 将 socket fd 注册到 epoll: epoll_ctl(ADD, fd)
//      ├─ 关联一个 Waker（唤醒器）
//      └─ 返回 Poll::Pending
//
// 3. 任务让出 CPU，调度器运行其他任务
//
// 4. 后台 I/O Reactor 线程执行事件循环：
//    loop {
//        let events = epoll_wait(epoll_fd, timeout); // 阻塞等待
//        for event in events {
//            // I/O 就绪！
//            event.waker.wake();  // 唤醒对应的任务
//        }
//    }
//
// 5. 被唤醒的任务重新进入调度队列
//
// 6. 再次 poll Future：
//    - 调用 read()，此时数据已就绪
//    - 返回 Poll::Ready(n)
```

**Waker 机制的关键作用**：

```rust
use std::task::{Context, Poll, Waker};
use std::sync::Arc;

// Waker 的本质：一个可以跨线程唤醒任务的句柄
struct MyWaker {
    task_id: usize,
    scheduler: Arc<Scheduler>,
}

impl MyWaker {
    fn wake(&self) {
        // 将任务重新加入调度队列
        self.scheduler.enqueue(self.task_id);
    }
}

// 在 I/O 就绪时调用
fn on_io_ready(waker: &Waker) {
    waker.wake_by_ref();  // 跨线程唤醒任务
}
```

**性能对比（操作系统视角）**：

```
场景：10,000 并发连接的 Web 服务器

┌─────────────────┬──────────────┬──────────────┬─────────────┐
│     模型        │  线程/任务数  │   内存占用   │  延迟      │
├─────────────────┼──────────────┼──────────────┼─────────────┤
│ 1 线程/连接     │  10,000      │  ~80 GB      │  高（切换） │
│ 线程池 (100)    │  100         │  ~800 MB     │  中等       │
│ 异步 (Tokio)   │  8 (CPU核心)  │  ~100 MB     │  低         │
└─────────────────┴──────────────┴──────────────┴─────────────┘

为什么异步更快？
1. 无上下文切换成本（协作式调度）
2. 更好的 CPU 缓存局部性
3. 更少的系统调用（批量注册 I/O 事件）
```

**与其他语言对比**：

```
┌──────────┬────────────────┬────────────────┬─────────────┐
│  语言    │  异步模型       │  成本          │  调度方式   │
├──────────┼────────────────┼────────────────┼─────────────┤
│ Rust     │  Future (惰性) │  零成本状态机   │  显式 await │
│ Go       │  Goroutine     │  ~2KB/协程     │  隐式调度   │
│ JavaScript│ Promise       │  堆分配        │  事件循环   │
│ Python   │  asyncio       │  堆分配        │  生成器     │
└──────────┴────────────────┴────────────────┴─────────────┘

Rust 的优势：
- 零成本抽象（状态机在编译时生成）
- 无垃圾回收压力
- 明确的异步边界（async/await）
```

**系统调用层面的优化**：

```rust
// Tokio 的批量 I/O 优化
//
// 不高效的做法（频繁系统调用）：
for socket in sockets {
    epoll_ctl(ADD, socket);  // N 次系统调用
}

// Tokio 的优化（批处理）：
let mut batch = Vec::new();
for socket in sockets {
    batch.push(socket);
}
// 一次性注册多个 fd（减少系统调用）
epoll_ctl_batch(ADD, batch);
```

**面试重点总结**：

1. **Future 本质是什么？**
   - 编译时生成的状态机，零堆分配

2. **Poll::Pending 时发生了什么？**
   - fd 注册到 epoll，任务让出 CPU，关联 Waker

3. **Waker 的作用？**
   - I/O 就绪时，epoll_wait 返回事件，调用 Waker 唤醒任务

4. **为什么异步比线程快？**
   - 无上下文切换、内存占用小、更好的缓存局部性

5. **Tokio 如何与操作系统交互？**
   - 通过 mio 库，封装 epoll/kqueue/IOCP

6. **Future 是惰性的什么意思？**
   - 不调用 `.await` 或 `poll()`，Future 不会执行（零成本）

7. **协作式 vs 抢占式调度？**
   - 协作式：任务主动 await 让出 CPU
   - 抢占式：操作系统强制切换线程（线程模型）

### 4.3 Tokio 运行时

```rust
use tokio;

// 基本的 Tokio 异步主函数
#[tokio::main]
async fn main() {
    println!("Hello from Tokio!");
}

// 或者手动创建运行时
fn manual_runtime() {
    let runtime = tokio::runtime::Runtime::new().unwrap();

    runtime.block_on(async {
        println!("Running in Tokio runtime");
    });
}

// 多线程运行时
#[tokio::main(flavor = "multi_thread", worker_threads = 4)]
async fn multi_thread_main() {
    // 4 个工作线程
}

// 当前线程运行时
#[tokio::main(flavor = "current_thread")]
async fn current_thread_main() {
    // 单线程运行时
}
```

### 4.4 异步任务生成

```rust
use tokio::task;
use tokio::time::{sleep, Duration};

async fn spawning_tasks() {
    let handle = task::spawn(async {
        sleep(Duration::from_secs(1)).await;
        println!("Task completed");
        42
    });

    let result = handle.await.unwrap();
    println!("Result: {}", result);
}

// 生成阻塞任务
async fn spawn_blocking_task() {
    let result = task::spawn_blocking(|| {
        // 执行CPU密集或阻塞I/O操作
        std::thread::sleep(Duration::from_secs(1));
        42
    }).await.unwrap();

    println!("Result: {}", result);
}

// 使用 JoinHandle
async fn multiple_tasks() {
    let mut handles = vec![];

    for i in 0..10 {
        let handle = task::spawn(async move {
            sleep(Duration::from_millis(100)).await;
            i * i
        });
        handles.push(handle);
    }

    for (i, handle) in handles.into_iter().enumerate() {
        let result = handle.await.unwrap();
        println!("Task {} result: {}", i, result);
    }
}
```

### 4.5 异步通道

```rust
use tokio::sync::{mpsc, oneshot, broadcast};

// mpsc 通道
async fn mpsc_example() {
    let (tx, mut rx) = mpsc::channel(32); // 缓冲区大小 32

    task::spawn(async move {
        for i in 0..10 {
            tx.send(i).await.unwrap();
        }
    });

    while let Some(value) = rx.recv().await {
        println!("Received: {}", value);
    }
}

// oneshot 通道（一次性）
async fn oneshot_example() {
    let (tx, rx) = oneshot::channel();

    task::spawn(async move {
        tx.send(42).unwrap();
    });

    let result = rx.await.unwrap();
    println!("Result: {}", result);
}

// broadcast 通道（广播）
async fn broadcast_example() {
    let (tx, mut rx1) = broadcast::channel(16);
    let mut rx2 = tx.subscribe();

    task::spawn(async move {
        for i in 0..5 {
            tx.send(i).unwrap();
        }
    });

    task::spawn(async move {
        while let Ok(value) = rx1.recv().await {
            println!("Receiver 1: {}", value);
        }
    });

    task::spawn(async move {
        while let Ok(value) = rx2.recv().await {
            println!("Receiver 2: {}", value);
        }
    });

    sleep(Duration::from_secs(1)).await;
}
```

### 4.6 异步互斥锁

```rust
use tokio::sync::{Mutex, RwLock};
use std::sync::Arc;

// Tokio Mutex（异步）
async fn tokio_mutex() {
    let data = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let data = Arc::clone(&data);
        let handle = task::spawn(async move {
            let mut lock = data.lock().await; // 异步等待锁
            *lock += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }

    println!("Result: {}", *data.lock().await);
}

// Tokio RwLock
async fn tokio_rwlock() {
    let lock = Arc::new(RwLock::new(5));

    // 多个读取者
    let lock1 = Arc::clone(&lock);
    let handle1 = task::spawn(async move {
        let r = lock1.read().await;
        println!("Read: {}", *r);
    });

    let lock2 = Arc::clone(&lock);
    let handle2 = task::spawn(async move {
        let r = lock2.read().await;
        println!("Read: {}", *r);
    });

    // 一个写入者
    let lock3 = Arc::clone(&lock);
    let handle3 = task::spawn(async move {
        let mut w = lock3.write().await;
        *w += 1;
        println!("Write: {}", *w);
    });

    handle1.await.unwrap();
    handle2.await.unwrap();
    handle3.await.unwrap();
}
```

**std::sync::Mutex vs tokio::sync::Mutex**:
```
┌─────────────────────────────────────────────────────────┐
│          std::sync::Mutex vs tokio::sync::Mutex         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  std::sync::Mutex:                                      │
│  ├── 阻塞式锁                                           │
│  ├── 在异步代码中会阻塞整个线程                           │
│  ├── 适用于临界区很短的情况                              │
│  └── 性能更好（如果锁竞争不激烈）                         │
│                                                         │
│  tokio::sync::Mutex:                                    │
│  ├── 异步锁，使用 .await                                │
│  ├── 不会阻塞线程，让出执行权                            │
│  ├── 适用于异步代码和长时间持有锁的情况                    │
│  └── 轻微性能开销                                       │
│                                                         │
│  原则：                                                 │
│  • 在 async 函数中跨 .await 持有锁 → tokio::Mutex      │
│  • 临界区不包含 .await → std::Mutex                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 4.7 Select 和超时

```rust
use tokio::time::{sleep, timeout, Duration};
use tokio::select;

// select! 宏
async fn select_example() {
    let mut interval = tokio::time::interval(Duration::from_millis(100));
    let mut counter = 0;

    loop {
        select! {
            _ = interval.tick() => {
                counter += 1;
                println!("Counter: {}", counter);
                if counter >= 5 {
                    break;
                }
            }
            _ = sleep(Duration::from_secs(1)) => {
                println!("Timeout reached");
                break;
            }
        }
    }
}

// 超时控制
async fn timeout_example() {
    let result = timeout(Duration::from_secs(1), async {
        sleep(Duration::from_secs(2)).await;
        42
    }).await;

    match result {
        Ok(value) => println!("Value: {}", value),
        Err(_) => println!("Timeout!"),
    }
}

// 竞速（race）
async fn race_example() {
    use futures::future::select;
    use futures::pin_mut;

    let future1 = async {
        sleep(Duration::from_secs(1)).await;
        "Future 1"
    };

    let future2 = async {
        sleep(Duration::from_millis(500)).await;
        "Future 2"
    };

    pin_mut!(future1, future2);

    match select(future1, future2).await {
        futures::future::Either::Left((result, _)) => {
            println!("Future 1 won: {}", result);
        }
        futures::future::Either::Right((result, _)) => {
            println!("Future 2 won: {}", result);
        }
    }
}
```

### 4.8 异步 I/O

```rust
use tokio::fs::File;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

// 异步文件读取
async fn read_file() -> Result<String, std::io::Error> {
    let mut file = File::open("example.txt").await?;
    let mut contents = String::new();
    file.read_to_string(&mut contents).await?;
    Ok(contents)
}

// 异步文件写入
async fn write_file() -> Result<(), std::io::Error> {
    let mut file = File::create("output.txt").await?;
    file.write_all(b"Hello, async world!").await?;
    Ok(())
}

// 异步 TCP 服务器
use tokio::net::TcpListener;

async fn tcp_server() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, addr) = listener.accept().await?;

        task::spawn(async move {
            let mut buf = [0; 1024];

            loop {
                match socket.read(&mut buf).await {
                    Ok(0) => return, // 连接关闭
                    Ok(n) => {
                        if socket.write_all(&buf[0..n]).await.is_err() {
                            return;
                        }
                    }
                    Err(_) => return,
                }
            }
        });
    }
}

// 异步 HTTP 客户端（使用 reqwest）
async fn http_request() -> Result<(), Box<dyn std::error::Error>> {
    let response = reqwest::get("https://api.github.com/repos/rust-lang/rust")
        .await?
        .text()
        .await?;

    println!("Response: {}", response);
    Ok(())
}
```

### 4.9 Stream 和异步迭代器

```rust
use futures::stream::{self, StreamExt};
use tokio_stream::wrappers::ReceiverStream;

// 基本 Stream 使用
async fn basic_stream() {
    let stream = stream::iter(vec![1, 2, 3, 4, 5]);

    stream
        .for_each(|x| async move {
            println!("Value: {}", x);
        })
        .await;
}

// Stream 转换
async fn stream_transform() {
    let stream = stream::iter(vec![1, 2, 3, 4, 5])
        .map(|x| x * 2)
        .filter(|x| {
            let result = x % 4 == 0;
            async move { result }
        });

    let results: Vec<_> = stream.collect().await;
    println!("Results: {:?}", results);
}

// 从通道创建 Stream
async fn stream_from_channel() {
    let (tx, rx) = mpsc::channel(32);
    let stream = ReceiverStream::new(rx);

    task::spawn(async move {
        for i in 0..10 {
            tx.send(i).await.unwrap();
        }
    });

    stream
        .for_each(|x| async move {
            println!("Received: {}", x);
        })
        .await;
}
```

---

## 5. 高级主题

### 5.1 Pin和Unpin

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

// 自引用结构体需要 Pin
struct SelfReferential {
    data: String,
    pointer: *const String, // 指向 data
    _pin: PhantomPinned,
}

impl SelfReferential {
    fn new(text: String) -> Pin<Box<Self>> {
        let mut boxed = Box::pin(SelfReferential {
            data: text,
            pointer: std::ptr::null(),
            _pin: PhantomPinned,
        });

        let self_ptr: *const String = &boxed.data;
        unsafe {
            let mut_ref = Pin::as_mut(&mut boxed);
            Pin::get_unchecked_mut(mut_ref).pointer = self_ptr;
        }

        boxed
    }
}

// Pin 的作用
fn pin_explanation() {
    // Pin<P> 保证：
    // 1. P指向的值不会在内存中移动
    // 2. 对于 !Unpin 类型，这个保证是强制的
    // 3. 对于 Unpin 类型，Pin 没有额外约束
}
```

### 5.2 Phantom Types

```rust
use std::marker::PhantomData;

// 类型状态模式
struct Locked;
struct Unlocked;

struct Door<State> {
    _state: PhantomData<State>,
}

impl Door<Locked> {
    fn new() -> Self {
        Door { _state: PhantomData }
    }

    fn unlock(self) -> Door<Unlocked> {
        Door { _state: PhantomData }
    }
}

impl Door<Unlocked> {
    fn lock(self) -> Door<Locked> {
        Door { _state: PhantomData }
    }

    fn open(&self) {
        println!("Door opened");
    }
}

fn phantom_example() {
    let door = Door::<Locked>::new();
    // door.open(); // ❌ 编译错误：Locked 状态不能打开

    let door = door.unlock();
    door.open(); // ✅ Unlocked 状态可以打开
}
```

### 5.3 高级 Trait

```rust
// Associated Types（关联类型）
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        if self.count < 6 {
            Some(self.count)
        } else {
            None
        }
    }
}

// Default Type Parameters
use std::ops::Add;

#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

// Operator Overloading
struct Millimeters(u32);
struct Meters(u32);

impl Add<Meters> for Millimeters {
    type Output = Millimeters;

    fn add(self, other: Meters) -> Millimeters {
        Millimeters(self.0 + (other.0 * 1000))
    }
}

// Trait Object
trait Draw {
    fn draw(&self);
}

struct Screen {
    components: Vec<Box<dyn Draw>>,
}

impl Screen {
    fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}

// Trait 约束
fn generic_function<T: Display + Clone>(item: T) {
    println!("{}", item);
    let _cloned = item.clone();
}

// Where 子句
fn complex_generic<T, U>(t: T, u: U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    println!("{}", t);
    println!("{:?}", u);
    42
}
```

### 5.4 宏编程

```rust
// 声明宏
macro_rules! vec_of_strings {
    ($($element:expr),*) => {
        {
            let mut v = Vec::new();
            $(
                v.push($element.to_string());
            )*
            v
        }
    };
}

fn declarative_macro() {
    let v = vec_of_strings!["hello", "world"];
    println!("{:?}", v);
}

// 过程宏示例（需要单独的 crate）
// derive 宏
#[derive(Debug, Clone)]
struct MyStruct {
    field: String,
}

// 属性宏
#[tokio::main]
async fn attribute_macro_example() {
    // ...
}

// 函数式宏
// println!("Hello, {}!", name);
```

---

## 6. 面试真题与解答

### 6.1 所有权问题

**Q1: 解释为什么下面的代码无法编译？**

```rust
fn problem1() {
    let s = String::from("hello");
    let s1 = s;
    println!("{}", s);
}
```

**A1:**
```
错误原因：s 的所有权已经移动到 s1，s 不再有效。

解决方案：
1. 克隆：let s1 = s.clone();
2. 借用：let s1 = &s;
3. 使用 s1 而不是 s
```

**Q2: 下面代码的输出是什么？为什么？**

```rust
fn problem2() {
    let mut v = vec![1, 2, 3];
    let r = &v[0];
    v.push(4);
    println!("{}", r);
}
```

**A2:**
```
编译错误！

原因：
1. r 是对 v[0] 的不可变借用
2. v.push(4) 需要可变借用 v
3. 违反了借用规则：不能同时存在可变和不可变借用

为什么 push 会导致问题：
- push 可能导致 vector 重新分配内存
- 如果重新分配，r 指向的内存可能失效（悬垂指针）
- Rust 在编译时防止这种情况

解决方案：
1. 在 push 之前不要持有引用
2. 先 push，再获取引用
3. 使用索引而不是引用
```

### 6.2 生命周期问题

**Q3: 修复下面的生命周期问题**

```rust
fn problem3() -> &str {
    let s = String::from("hello");
    &s
}
```

**A3:**
```rust
// 问题：返回指向局部变量的引用

// 解决方案 1：返回 String（转移所有权）
fn solution3_1() -> String {
    String::from("hello")
}

// 解决方案 2：使用 'static 生命周期
fn solution3_2() -> &'static str {
    "hello" // 字符串字面值
}

// 解决方案 3：接受生命周期参数
fn solution3_3<'a>(s: &'a str) -> &'a str {
    s
}
```

**Q4: 解释生命周期省略规则**

**A4:**
```rust
// 规则 1：每个引用参数都有自己的生命周期
fn rule1(s: &str) {}
// 等价于：fn rule1<'a>(s: &'a str) {}

// 规则 2：如果只有一个输入生命周期，赋给所有输出
fn rule2(s: &str) -> &str { s }
// 等价于：fn rule2<'a>(s: &'a str) -> &'a str { s }

// 规则 3：如果有 &self，self 的生命周期赋给所有输出
impl MyStruct {
    fn rule3(&self, s: &str) -> &str { &self.field }
    // 等价于：
    // fn rule3<'a, 'b>(&'a self, s: &'b str) -> &'a str
}

// 需要显式标注的情况：
fn explicit<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    x // 返回 x 的生命周期
}
```

### 6.3 并发问题

**Q5: Send 和 Sync 的区别是什么？举例说明。**

**A5:**
```rust
/*
Send: 可以安全地在线程间转移所有权
- 实现 Send 的类型可以被 move 到另一个线程
- 例子：大多数类型都是 Send
- 反例：Rc<T> 不是 Send（用 Arc<T> 代替）

Sync: 可以安全地在多个线程间共享引用
- T 是 Sync ⇔ &T 是 Send
- 实现 Sync 的类型的引用可以被多个线程访问
- 例子：Arc<T>, Mutex<T>
- 反例：Cell<T>, RefCell<T> 不是 Sync
*/

// 示例 1：Rc 不是 Send
fn not_send() {
    let rc = Rc::new(5);
    // thread::spawn(move || { // ❌ Rc<i32> cannot be sent
    //     println!("{}", rc);
    // });
}

// 示例 2：Arc 是 Send + Sync
fn send_sync() {
    let arc = Arc::new(5);
    thread::spawn(move || {
        println!("{}", arc); // ✅
    });
}

// 示例 3：RefCell 不是 Sync
fn not_sync() {
    let cell = RefCell::new(5);
    let cell_ref = &cell;

    // thread::spawn(move || { // ❌ RefCell 不是 Sync
    //     cell_ref.borrow_mut();
    // });
}
```

**Q6: 如何避免死锁？**

**A6:**
```rust
/*
死锁预防策略：

1. 锁顺序一致性
   - 所有线程按相同顺序获取锁

2. 超时机制
   - 使用 try_lock_for() 而不是 lock()

3. 避免嵌套锁
   - 尽量不要在持有一个锁时获取另一个锁

4. 使用更高级的抽象
   - 消息传递而不是共享内存
   - 使用 channel 代替 Mutex
*/

// 错误示例：可能死锁
fn may_deadlock() {
    let mutex1 = Arc::new(Mutex::new(1));
    let mutex2 = Arc::new(Mutex::new(2));

    let m1 = Arc::clone(&mutex1);
    let m2 = Arc::clone(&mutex2);

    thread::spawn(move || {
        let _g1 = m1.lock().unwrap();
        thread::sleep(Duration::from_millis(10));
        let _g2 = m2.lock().unwrap(); // 死锁！
    });

    let _g2 = mutex2.lock().unwrap();
    thread::sleep(Duration::from_millis(10));
    let _g1 = mutex1.lock().unwrap(); // 死锁！
}

// 正确示例：锁顺序一致
fn no_deadlock() {
    let mutex1 = Arc::new(Mutex::new(1));
    let mutex2 = Arc::new(Mutex::new(2));

    // 总是先锁 mutex1，再锁 mutex2
    fn lock_both(m1: &Mutex<i32>, m2: &Mutex<i32>) {
        let _g1 = m1.lock().unwrap();
        let _g2 = m2.lock().unwrap();
        // 使用锁
    }
}
```

### 6.4 异步问题

**Q7: async/await 是如何工作的？**

**A7:**
```rust
/*
async/await 工作原理：

1. async函数返回一个实现了Future trait的类型
2. Future是一个状态机，代表可能还未完成的计算
3. .await暂停Future的执行，让出控制权
4. 运行时负责轮询Future并在准备好时恢复执行

状态转换：
┌─────────┐  .await  ┌─────────┐  ready  ┌─────────┐
│ Created │ ───────→ │ Pending │ ──────→ │  Ready  │
└─────────┘          └─────────┘         └─────────┘
                           ↑                  │
                           └──── poll ────────┘
*/

// async 函数编译后的伪代码
async fn example() -> i32 {
    let x = some_async_fn().await;
    x + 1
}

// 编译器生成的状态机（简化版）
enum ExampleFuture {
    Initial,
    WaitingForSomeAsyncFn(/* future */),
    Done,
}

impl Future for ExampleFuture {
    type Output = i32;

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<i32> {
        // 状态机逻辑
        match self {
            Initial => {
                // 开始执行 some_async_fn
            }
            WaitingForSomeAsyncFn(fut) => {
                // 轮询内部 future
                match fut.poll(cx) {
                    Poll::Ready(x) => Poll::Ready(x + 1),
                    Poll::Pending => Poll::Pending,
                }
            }
            Done => panic!("polled after completion"),
        }
    }
}
```

**Q8: 什么时候使用 tokio::spawn 而不是 std::thread::spawn？**

**A8:**
```rust
/*
tokio::spawn vs std::thread::spawn:

tokio::spawn (异步任务):
优点：
- 轻量级，可以创建数百万个任务
- 由少量 OS 线程调度
- 适合 I/O 密集型操作
- 自动与 Tokio 运行时集成

缺点：
- 需要 async 代码
- 只能在 Tokio 运行时中使用

std::thread::spawn (OS 线程):
优点：
- 可以运行同步代码
- 真正的并行执行（CPU 密集型）
- 不需要运行时

缺点：
- 创建成本高
- 数量受限于系统资源
- 不适合大量并发

选择原则：
- I/O 操作（网络、文件） → tokio::spawn
- CPU 密集型计算 → std::thread::spawn 或 tokio::task::spawn_blocking
- 大量并发 → tokio::spawn
- 需要阻塞操作 → tokio::task::spawn_blocking
*/

// I/O 密集型：使用 tokio::spawn
async fn io_intensive() {
    for _ in 0..10000 {
        tokio::spawn(async {
            // 网络请求
            reqwest::get("https://example.com").await.unwrap();
        });
    }
}

// CPU 密集型：使用 spawn_blocking
async fn cpu_intensive() {
    tokio::task::spawn_blocking(|| {
        // 复杂计算
        (0..1000000).sum::<i32>()
    }).await.unwrap();
}
```

---

## 7. 实战项目示例

### 7.1 并发 Web 爬虫

```rust
use tokio;
use reqwest;
use std::sync::Arc;
use tokio::sync::Semaphore;
use futures::stream::{self, StreamExt};

struct WebCrawler {
    client: reqwest::Client,
    semaphore: Arc<Semaphore>,
}

impl WebCrawler {
    fn new(max_concurrent: usize) -> Self {
        WebCrawler {
            client: reqwest::Client::new(),
            semaphore: Arc::new(Semaphore::new(max_concurrent)),
        }
    }

    async fn fetch(&self, url: &str) -> Result<String, reqwest::Error> {
        let _permit = self.semaphore.acquire().await.unwrap();
        println!("Fetching: {}", url);

        let response = self.client.get(url).send().await?;
        let body = response.text().await?;

        Ok(body)
    }

    async fn crawl(&self, urls: Vec<String>) -> Vec<Result<String, reqwest::Error>> {
        stream::iter(urls)
            .map(|url| self.fetch(&url))
            .buffer_unordered(10) // 并发数
            .collect()
            .await
    }
}

#[tokio::main]
async fn web_crawler_example() {
    let crawler = WebCrawler::new(5);

    let urls = vec![
        "https://example.com".to_string(),
        "https://rust-lang.org".to_string(),
        // ...更多 URL
    ];

    let results = crawler.crawl(urls).await;

    for (i, result) in results.iter().enumerate() {
        match result {
            Ok(body) => println!("URL {}: {} bytes", i, body.len()),
            Err(e) => eprintln!("URL {}: Error: {}", i, e),
        }
    }
}
```

### 7.2 生产者-消费者模式

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[derive(Debug)]
struct Job {
    id: usize,
    data: String,
}

async fn producer(tx: mpsc::Sender<Job>, count: usize) {
    for i in 0..count {
        let job = Job {
            id: i,
            data: format!("Job {}", i),
        };

        if tx.send(job).await.is_err() {
            eprintln!("Receiver dropped");
            return;
        }

        sleep(Duration::from_millis(100)).await;
    }
}

async fn consumer(mut rx: mpsc::Receiver<Job>, id: usize) {
    while let Some(job) = rx.recv().await {
        println!("Consumer {} processing {:?}", id, job);

        // 模拟处理时间
        sleep(Duration::from_millis(200)).await;

        println!("Consumer {} completed job {}", id, job.id);
    }

    println!("Consumer {} shutting down", id);
}

#[tokio::main]
async fn producer_consumer_example() {
    let (tx, rx) = mpsc::channel(32);

    // 启动生产者
    let producer_handle = tokio::spawn(producer(tx, 20));

    // 启动多个消费者
    let rx = Arc::new(Mutex::new(rx));
    let mut consumer_handles = vec![];

    for i in 0..3 {
        let rx = Arc::clone(&rx);
        let handle = tokio::spawn(async move {
            loop {
                let job = {
                    let mut rx = rx.lock().await;
                    rx.recv().await
                };

                if let Some(job) = job {
                    println!("Consumer {} processing {:?}", i, job);
                    sleep(Duration::from_millis(200)).await;
                } else {
                    break;
                }
            }
        });
        consumer_handles.push(handle);
    }

    producer_handle.await.unwrap();

    for handle in consumer_handles {
        handle.await.unwrap();
    }
}
```

### 7.3 异步 Actor 系统

```rust
use tokio::sync::mpsc;

// Actor 消息
enum Message {
    Increment,
    Decrement,
    GetValue(oneshot::Sender<i32>),
}

// Actor
struct CounterActor {
    receiver: mpsc::Receiver<Message>,
    count: i32,
}

impl CounterActor {
    fn new(receiver: mpsc::Receiver<Message>) -> Self {
        CounterActor {
            receiver,
            count: 0,
        }
    }

    async fn run(mut self) {
        while let Some(msg) = self.receiver.recv().await {
            match msg {
                Message::Increment => {
                    self.count += 1;
                }
                Message::Decrement => {
                    self.count -= 1;
                }
                Message::GetValue(respond_to) => {
                    let _ = respond_to.send(self.count);
                }
            }
        }
    }
}

// Actor Handle
#[derive(Clone)]
struct CounterHandle {
    sender: mpsc::Sender<Message>,
}

impl CounterHandle {
    fn new() -> Self {
        let (sender, receiver) = mpsc::channel(32);
        let actor = CounterActor::new(receiver);

        tokio::spawn(actor.run());

        Self { sender }
    }

    async fn increment(&self) {
        let _ = self.sender.send(Message::Increment).await;
    }

    async fn decrement(&self) {
        let _ = self.sender.send(Message::Decrement).await;
    }

    async fn get_value(&self) -> i32 {
        let (send, recv) = oneshot::channel();
        let _ = self.sender.send(Message::GetValue(send)).await;
        recv.await.unwrap_or(0)
    }
}

#[tokio::main]
async fn actor_example() {
    let counter = CounterHandle::new();

    let counter1 = counter.clone();
    let handle1 = tokio::spawn(async move {
        for _ in 0..5 {
            counter1.increment().await;
        }
    });

    let counter2 = counter.clone();
    let handle2 = tokio::spawn(async move {
        for _ in 0..3 {
            counter2.decrement().await;
        }
    });

    handle1.await.unwrap();
    handle2.await.unwrap();

    let value = counter.get_value().await;
    println!("Final value: {}", value); // 2
}
```

### 7.4 连接池实现

```rust
use tokio::sync::{Semaphore, Mutex};
use std::sync::Arc;
use tokio::time::{sleep, Duration};

struct Connection {
    id: usize,
}

impl Connection {
    async fn query(&self, sql: &str) -> Result<String, String> {
        println!("Connection {} executing: {}", self.id, sql);
        sleep(Duration::from_millis(100)).await;
        Ok(format!("Result from connection {}", self.id))
    }
}

struct ConnectionPool {
    connections: Arc<Mutex<Vec<Connection>>>,
    semaphore: Arc<Semaphore>,
}

impl ConnectionPool {
    fn new(size: usize) -> Self {
        let connections = (0..size)
            .map(|i| Connection { id: i })
            .collect();

        ConnectionPool {
            connections: Arc::new(Mutex::new(connections)),
            semaphore: Arc::new(Semaphore::new(size)),
        }
    }

    async fn get_connection(&self) -> PooledConnection {
        let permit = self.semaphore.clone().acquire_owned().await.unwrap();
        let conn = self.connections.lock().await.pop().unwrap();

        PooledConnection {
            conn: Some(conn),
            pool: self.connections.clone(),
            _permit: permit,
        }
    }
}

struct PooledConnection {
    conn: Option<Connection>,
    pool: Arc<Mutex<Vec<Connection>>>,
    _permit: tokio::sync::OwnedSemaphorePermit,
}

impl PooledConnection {
    async fn query(&self, sql: &str) -> Result<String, String> {
        self.conn.as_ref().unwrap().query(sql).await
    }
}

impl Drop for PooledConnection {
    fn drop(&mut self) {
        let conn = self.conn.take().unwrap();
        let pool = self.pool.clone();

        tokio::spawn(async move {
            pool.lock().await.push(conn);
        });
    }
}

#[tokio::main]
async fn pool_example() {
    let pool = Arc::new(ConnectionPool::new(3));
    let mut handles = vec![];

    for i in 0..10 {
        let pool = Arc::clone(&pool);
        let handle = tokio::spawn(async move {
            let conn = pool.get_connection().await;
            let result = conn.query(&format!("SELECT * FROM table_{}", i)).await;
            println!("Query {} result: {:?}", i, result);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }
}
```

---

## 8. Unsafe Rust 与 FFI

### 8.1 Unsafe 的五种超能力

```rust
/*
Unsafe Rust 允许的五种操作：

1. 解引用裸指针
2. 调用 unsafe 函数或方法
3. 访问或修改可变静态变量
4. 实现 unsafe trait
5. 访问 union 的字段
*/

// 1. 裸指针 (Raw Pointers)
fn raw_pointers() {
    let mut num = 5;

    // 创建裸指针是安全的
    let r1 = &num as *const i32;
    let r2 = &mut num as *mut i32;

    // 解引用裸指针需要 unsafe
    unsafe {
        println!("r1 is: {}", *r1);
        println!("r2 is: {}", *r2);
    }
}

// 裸指针与引用的区别
fn raw_vs_ref_comparison() {
    /*
    裸指针：
    - 允许同时存在不可变和可变指针
    - 不保证指向有效内存
    - 可以为 null
    - 没有自动清理

    引用：
    - 遵守借用规则
    - 保证总是有效
    - 不可为 null
    - 自动清理（通过 Drop）
    */

    let mut num = 5;
    let r1 = &num as *const i32;
    let r2 = &mut num as *mut i32;
    // ✅ 同时存在可变和不可变裸指针（只有解引用时才 unsafe）
}

// 2. Unsafe 函数
unsafe fn dangerous() {
    println!("This is unsafe!");
}

fn call_unsafe() {
    unsafe {
        dangerous();
    }
}

// 3. 安全抽象封装 unsafe 代码
fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();

    assert!(mid <= len);

    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}

// 4. extern 函数调用 (FFI)
extern "C" {
    fn abs(input: i32) -> i32;
}

fn call_c_function() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}

// 5. 可变静态变量
static mut COUNTER: u32 = 0;

fn add_to_count(inc: u32) {
    unsafe {
        COUNTER += inc;
    }
}

// 6. Unsafe trait 实现
unsafe trait Foo {
    // trait 方法
}

unsafe impl Foo for i32 {
    // 实现
}
```

### 8.2 常见 Unsafe 使用场景

```rust
// 场景 1: 与 C 库交互
use std::os::raw::c_char;
use std::ffi::CString;

extern "C" {
    fn printf(format: *const c_char, ...) -> i32;
}

fn use_printf() {
    let c_string = CString::new("Hello, %s!\n").unwrap();
    let world = CString::new("World").unwrap();

    unsafe {
        printf(c_string.as_ptr(), world.as_ptr());
    }
}

// 场景 2: 优化性能（避免边界检查）
fn unchecked_access() {
    let v = vec![1, 2, 3, 4, 5];

    unsafe {
        // 跳过边界检查（仅在确定索引有效时使用）
        let elem = v.get_unchecked(2);
        println!("{}", elem);
    }
}

// 场景 3: 实现自定义数据结构
struct MyVec<T> {
    ptr: *mut T,
    len: usize,
    capacity: usize,
}

impl<T> MyVec<T> {
    fn new() -> Self {
        MyVec {
            ptr: std::ptr::NonNull::dangling().as_ptr(),
            len: 0,
            capacity: 0,
        }
    }

    fn push(&mut self, elem: T) {
        if self.len == self.capacity {
            self.grow();
        }

        unsafe {
            std::ptr::write(self.ptr.add(self.len), elem);
        }

        self.len += 1;
    }

    fn grow(&mut self) {
        let new_capacity = if self.capacity == 0 { 1 } else { self.capacity * 2 };
        let new_layout = std::alloc::Layout::array::<T>(new_capacity).unwrap();

        let new_ptr = unsafe {
            if self.capacity == 0 {
                std::alloc::alloc(new_layout)
            } else {
                let old_layout = std::alloc::Layout::array::<T>(self.capacity).unwrap();
                std::alloc::realloc(self.ptr as *mut u8, old_layout, new_layout.size())
            }
        };

        self.ptr = new_ptr as *mut T;
        self.capacity = new_capacity;
    }
}

impl<T> Drop for MyVec<T> {
    fn drop(&mut self) {
        if self.capacity != 0 {
            unsafe {
                let layout = std::alloc::Layout::array::<T>(self.capacity).unwrap();
                std::alloc::dealloc(self.ptr as *mut u8, layout);
            }
        }
    }
}
```

**Unsafe 使用原则**：
```
┌─────────────────────────────────────────────────────────┐
│                Unsafe Rust 最佳实践                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 最小化 unsafe 块                                     │
│     ├── 只在必要时使用 unsafe                            │
│     └── 尽可能缩小 unsafe 代码范围                        │
│                                                         │
│  2. 提供安全抽象                                         │
│     ├── 在 unsafe 代码外包装安全接口                      │
│     └── 确保不变量在边界处得到维护                        │
│                                                         │
│  3. 文档化假设                                           │
│     ├── 注释说明为什么是安全的                            │
│     └── 列出必须满足的前提条件                            │
│                                                         │
│  4. 严格测试                                             │
│     ├── 使用 Miri 检测未定义行为                         │
│     ├── 使用 AddressSanitizer                           │
│     └── 编写全面的单元测试                               │
│                                                         │
│  5. 审查检查                                             │
│     ├── Unsafe 代码需要特别的代码审查                     │
│     └── 考虑是否有安全的替代方案                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 9. 错误处理高级技巧

### 9.1 Result 和 Option 组合器

```rust
// 基本错误处理
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Division by zero".to_string())
    } else {
        Ok(a / b)
    }
}

// ? 操作符
fn calculate() -> Result<f64, String> {
    let x = divide(10.0, 2.0)?; // 自动传播错误
    let y = divide(x, 5.0)?;
    Ok(y)
}

// 自定义错误类型
use std::fmt;

#[derive(Debug)]
enum MathError {
    DivisionByZero,
    NegativeSquareRoot,
}

impl fmt::Display for MathError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            MathError::DivisionByZero => write!(f, "Division by zero"),
            MathError::NegativeSquareRoot => write!(f, "Square root of negative number"),
        }
    }
}

impl std::error::Error for MathError {}

fn safe_divide(a: f64, b: f64) -> Result<f64, MathError> {
    if b == 0.0 {
        Err(MathError::DivisionByZero)
    } else {
        Ok(a / b)
    }
}

fn safe_sqrt(x: f64) -> Result<f64, MathError> {
    if x < 0.0 {
        Err(MathError::NegativeSquareRoot)
    } else {
        Ok(x.sqrt())
    }
}
```

### 9.2 thiserror 和 anyhow

```rust
// 使用 thiserror 定义错误
use thiserror::Error;

#[derive(Error, Debug)]
enum DataStoreError {
    #[error("data store disconnected")]
    Disconnect(#[from] std::io::Error),

    #[error("the data for key `{0}` is not available")]
    Redaction(String),

    #[error("invalid header (expected {expected:?}, found {found:?})")]
    InvalidHeader {
        expected: String,
        found: String,
    },

    #[error("unknown data store error")]
    Unknown,
}

// 使用 anyhow 进行应用级错误处理
use anyhow::{Context, Result};

fn read_config() -> Result<String> {
    std::fs::read_to_string("config.toml")
        .context("Failed to read config file")?;
    Ok("config".to_string())
}

fn process_data() -> Result<()> {
    let config = read_config()
        .context("Failed to load configuration")?;

    // 更多处理...

    Ok(())
}
```

### 9.3 错误处理模式

```rust
// 模式 1: 组合多个 Result
fn combine_results() -> Result<i32, String> {
    let a = Some(5);
    let b = Some(10);

    let sum = a.and_then(|x| b.map(|y| x + y));

    sum.ok_or_else(|| "Missing values".to_string())
}

// 模式 2: 错误转换
fn convert_errors() -> Result<(), Box<dyn std::error::Error>> {
    let num: i32 = "42".parse()?; // ParseIntError 自动转换
    let _file = std::fs::read("file.txt")?; // io::Error 自动转换
    Ok(())
}

// 模式 3: 提供默认值
fn with_default() -> i32 {
    let result: Result<i32, _> = "not a number".parse();
    result.unwrap_or(0) // 失败时返回 0
}

// 模式 4: map 和 and_then
fn transform_result() -> Result<i32, std::num::ParseIntError> {
    "42".parse::<i32>()
        .map(|x| x * 2) // 成功时转换
        .and_then(|x| Ok(x + 1)) // 链式操作
}

// 模式 5: 早期返回
fn early_return(x: Option<i32>) -> Option<i32> {
    let value = x?; // 如果是 None，直接返回 None
    Some(value * 2)
}
```

---

## 10. 性能优化技巧

### 10.1 零成本抽象

```rust
// 迭代器：零成本抽象
fn sum_manual(v: &[i32]) -> i32 {
    let mut sum = 0;
    for i in 0..v.len() {
        sum += v[i];
    }
    sum
}

fn sum_iterator(v: &[i32]) -> i32 {
    v.iter().sum() // 编译后与手动循环相同
}

// 泛型单态化（Monomorphization）
fn generic_function<T: std::ops::Add<Output = T>>(a: T, b: T) -> T {
    a + b
}

fn monomorphization() {
    // 编译器为每种具体类型生成专门的函数
    let _ = generic_function(1i32, 2i32);    // 生成 i32 版本
    let _ = generic_function(1.0f64, 2.0f64); // 生成 f64 版本
    // 运行时没有动态派发开销
}
```

### 10.2 内存布局优化

```rust
// 结构体对齐和填充
#[derive(Debug)]
struct Unoptimized {
    a: u8,  // 1 byte
    b: u64, // 8 bytes (需要对齐到 8 字节边界)
    c: u8,  // 1 byte
    // 实际占用: 24 bytes (有填充)
}

#[derive(Debug)]
struct Optimized {
    b: u64, // 8 bytes
    a: u8,  // 1 byte
    c: u8,  // 1 byte
    // 实际占用: 16 bytes (更紧凑)
}

// repr 属性
#[repr(C)] // C 语言兼容布局
struct CCompatible {
    x: i32,
    y: i32,
}

#[repr(packed)] // 移除填充（可能影响性能）
struct Packed {
    a: u8,
    b: u64,
    c: u8,
}

#[repr(align(64))] // 指定对齐到 64 字节（缓存行）
struct CacheAligned {
    data: [u8; 64],
}

// 检查大小和对齐
fn check_layout() {
    use std::mem::{size_of, align_of};

    println!("Unoptimized: size={}, align={}",
        size_of::<Unoptimized>(), align_of::<Unoptimized>());
    println!("Optimized: size={}, align={}",
        size_of::<Optimized>(), align_of::<Optimized>());
}
```

### 10.3 内联优化

```rust
// 内联建议
#[inline]
fn small_function(x: i32) -> i32 { x * 2 }

#[inline(always)] // 总是内联
fn always_inline(x: i32) -> i32 { x + 1 }

#[inline(never)] // 从不内联
fn never_inline(x: i32) -> i32 { x - 1 }

// 编译器优化提示
#[cold] // 标记为不太可能执行的代码
fn error_path() { eprintln!("Error occurred"); }
```

---

## 11. 常见设计模式

### 11.1 Newtype 模式

```rust
// 为基本类型提供类型安全
struct UserId(u64);
struct ProductId(u64);

fn get_user(id: UserId) -> String {
    format!("User {}", id.0)
}

fn newtype_safety() {
    let user_id = UserId(42);
    let product_id = ProductId(100);

    get_user(user_id); // ✅
    // get_user(product_id); // ❌ 类型不匹配
}

// 实现 trait
impl std::fmt::Display for UserId {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "UserId({})", self.0)
    }
}
```

### 11.2 类型状态模式

```rust
// 使用类型系统编码状态机
struct Empty;
struct Ready;

struct Buffer<State = Empty> {
    data: Vec<u8>,
    _state: std::marker::PhantomData<State>,
}

impl Buffer<Empty> {
    fn new() -> Self {
        Buffer {
            data: Vec::new(),
            _state: std::marker::PhantomData,
        }
    }

    fn fill(mut self, data: Vec<u8>) -> Buffer<Ready> {
        self.data = data;
        Buffer {
            data: self.data,
            _state: std::marker::PhantomData,
        }
    }
}

impl Buffer<Ready> {
    fn process(&self) {
        println!("Processing {} bytes", self.data.len());
    }

    fn clear(self) -> Buffer<Empty> {
        Buffer {
            data: Vec::new(),
            _state: std::marker::PhantomData,
        }
    }
}

fn typestate_example() {
    let buffer = Buffer::new();
    // buffer.process(); // ❌ 编译错误：Empty 状态不能 process

    let buffer = buffer.fill(vec![1, 2, 3]);
    buffer.process(); // ✅ Ready 状态可以 process
}
```

### 11.3 Builder 模式

```rust
#[derive(Debug)]
struct User {
    username: String,
    email: String,
    age: Option<u32>,
    country: Option<String>,
}

struct UserBuilder {
    username: String,
    email: String,
    age: Option<u32>,
    country: Option<String>,
}

impl UserBuilder {
    fn new(username: String, email: String) -> Self {
        UserBuilder {
            username,
            email,
            age: None,
            country: None,
        }
    }

    fn age(mut self, age: u32) -> Self {
        self.age = Some(age);
        self
    }

    fn country(mut self, country: String) -> Self {
        self.country = Some(country);
        self
    }

    fn build(self) -> User {
        User {
            username: self.username,
            email: self.email,
            age: self.age,
            country: self.country,
        }
    }
}

fn builder_example() {
    let user = UserBuilder::new(
        "alice".to_string(),
        "alice@example.com".to_string()
    )
    .age(30)
    .country("USA".to_string())
    .build();

    println!("{:?}", user);
}
```

---

## 总结

### 核心要点

```
┌─────────────────────────────────────────────────────────┐
│                  Rust 核心概念总结                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  所有权：                                                │
│  ├── 每个值有唯一所有者                                    │
│  ├── Move vs Copy                                       │
│  └── Drop 自动清理                                       │
│                                                         │
│  借用：                                                  │
│  ├── 多个 &T 或一个 &mut T                                │
│  ├── 引用必须有效                                         │
│  └── 编译时检查，零运行时开销                               │
│                                                         │
│  生命周期：                                               │
│  ├── 防止悬垂引用                                         │
│  ├── 编译器推断 + 显式标注                                 │
│  └── 'static, 'a, HRTB                                  │
│                                                         │
│  并发：                                                  │
│  ├── Send + Sync trait                                  │
│  ├── Arc + Mutex 共享状态                                │
│  ├── Channel 消息传递                                    │
│  └── 编译时防止数据竞争                                    │
│                                                         │
│  异步：                                                  │
│  ├── Future + async/await                               │
│  ├── Zero-cost 抽象                                      │
│  ├── Tokio 运行时                                        │
│  └── 高效 I/O 多路复用                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 面试准备建议

1. **深入理解核心概念**
   - 所有权、借用、生命周期是基础
   - 理解编译器如何保证内存安全
   - 能够解释为什么 Rust 没有 GC 也能保证安全

2. **实践异步编程**
   - 理解 Future 的工作原理
   - 熟悉 Tokio 生态系统
   - 能够编写高性能异步代码

3. **掌握并发模型**
   - Send 和 Sync 的深入理解
   - 各种同步原语的使用场景
   - 避免数据竞争和死锁

4. **代码能力**
   - 能够快速定位编译错误
   - 理解错误信息并修复
   - 编写惯用的 Rust 代码

5. **系统设计**
   - Actor 模型
   - 事件驱动架构
   - 微服务设计

祝你面试顺利！🦀
