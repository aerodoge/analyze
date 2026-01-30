# C++游戏服务器开发学习计划

> **目标**：成都C++游戏服务器开发工程师（15-30K，3-5年）
>
> **面试通过率目标**：≥95%
>
> **学习周期**：12周（3个月）
>
> **每日学习时间**：3-4小时

---

## JD需求分析

### **岗位信息**：
- 📍 地点：成都
- 💰 薪资：15-30K
- 📅 经验：3-5年
- 🎓 学历：大专

### **核心技术要求**：
```
必备技能（权重高）：
├─ C++精通 ⭐⭐⭐⭐⭐
├─ 游戏服务器架构 ⭐⭐⭐⭐⭐
├─ 网络编程 ⭐⭐⭐⭐⭐
├─ Linux开发 ⭐⭐⭐⭐
├─ 数据库（MySQL、Redis）⭐⭐⭐⭐
└─ 分布式经验 ⭐⭐⭐⭐

加分项：
├─ Golang ⭐⭐⭐
├─ Boost库 ⭐⭐⭐
├─ Lua ⭐⭐
├─ Python ⭐⭐
├─ 微服务经验 ⭐⭐
└─ OpenGL（客户端相关，优先级低）⭐
```

### **工作内容**：
1. 游戏服务器开发与维护（稳定性、可扩展性）
2. 功能模块设计 + 数据库结构设计
3. 团队协作 + 测试 + 技术问题解决

---

## 学习路线图

```
阶段1：C++核心强化（第1-3周）
├─ C++11/14/17新特性
├─ STL深度使用
├─ 多线程编程
├─ 智能指针与内存管理
└─ 设计模式

阶段2：网络编程（第4-5周）
├─ TCP/UDP原理
├─ Socket编程
├─ IO多路复用（epoll、select、poll）
├─ Reactor/Proactor模式
└─ 网络库（libevent、muduo）

阶段3：游戏服务器基础（第6-7周）
├─ 游戏服务器架构
├─ 协议设计（Protobuf）
├─ 状态同步
├─ 帧同步
└─ AOI算法

阶段4：数据库与缓存（第8周）
├─ MySQL深度使用
├─ Redis数据结构
├─ 数据库设计
└─ 缓存策略

阶段5：分布式与微服务（第9周）
├─ 分布式系统基础
├─ 负载均衡
├─ 服务发现
└─ 消息队列

阶段6：Linux与工具链（第10周）
├─ Linux系统编程
├─ Shell脚本
├─ GDB调试
├─ 性能分析（perf、valgrind）
└─ CMake/Makefile

阶段7：综合实战（第11-12周）
├─ 实现完整的游戏服务器
├─ 压力测试
├─ 优化性能
└─ 面试准备
```

---

## 阶段1：C++核心强化（第1-3周）

### **Week 1：C++11/14/17新特性与STL**

#### **Day 1：C++11核心特性**

**学习目标**：
- 掌握auto、decltype、nullptr
- 理解右值引用与移动语义
- 学习lambda表达式

**学习内容**：

1. **auto与类型推导**：
   ```cpp
   // auto自动类型推导
   auto x = 10;              // int
   auto y = 3.14;            // double
   auto str = "hello";       // const char*

   // 迭代器简化
   std::vector<int> vec = {1, 2, 3, 4, 5};

   // C++03写法
   for (std::vector<int>::iterator it = vec.begin(); it != vec.end(); ++it) {
       std::cout << *it << std::endl;
   }

   // C++11写法
   for (auto it = vec.begin(); it != vec.end(); ++it) {
       std::cout << *it << std::endl;
   }

   // 更简洁的范围for
   for (auto val : vec) {
       std::cout << val << std::endl;
   }

   // decltype - 获取表达式的类型
   int a = 10;
   decltype(a) b = 20;  // b的类型是int

   decltype(a + b) c = 30;  // c的类型是int
   ```

2. **右值引用与移动语义**：
   ```cpp
   #include <iostream>
   #include <vector>
   #include <string>

   class MyString {
   private:
       char* data;
       size_t length;

   public:
       // 构造函数
       MyString(const char* str = "") {
           length = strlen(str);
           data = new char[length + 1];
           strcpy(data, str);
           std::cout << "Constructor called" << std::endl;
       }

       // 拷贝构造（深拷贝）
       MyString(const MyString& other) {
           length = other.length;
           data = new char[length + 1];
           strcpy(data, other.data);
           std::cout << "Copy constructor called" << std::endl;
       }

       // 移动构造（C++11）- 避免深拷贝
       MyString(MyString&& other) noexcept {
           data = other.data;
           length = other.length;
           other.data = nullptr;
           other.length = 0;
           std::cout << "Move constructor called" << std::endl;
       }

       // 拷贝赋值
       MyString& operator=(const MyString& other) {
           if (this != &other) {
               delete[] data;
               length = other.length;
               data = new char[length + 1];
               strcpy(data, other.data);
               std::cout << "Copy assignment called" << std::endl;
           }
           return *this;
       }

       // 移动赋值（C++11）
       MyString& operator=(MyString&& other) noexcept {
           if (this != &other) {
               delete[] data;
               data = other.data;
               length = other.length;
               other.data = nullptr;
               other.length = 0;
               std::cout << "Move assignment called" << std::endl;
           }
           return *this;
       }

       ~MyString() {
           delete[] data;
       }

       const char* c_str() const { return data; }
   };

   // 测试
   int main() {
       MyString s1("hello");
       MyString s2 = s1;                    // 拷贝构造
       MyString s3 = std::move(s1);         // 移动构造（s1被掏空）

       std::vector<MyString> vec;
       vec.push_back(MyString("world"));    // 移动构造（临时对象）

       return 0;
   }

   // 输出：
   // Constructor called
   // Copy constructor called
   // Move constructor called
   // Constructor called
   // Move constructor called
   ```

3. **Lambda表达式**：
   ```cpp
   #include <iostream>
   #include <vector>
   #include <algorithm>

   int main() {
       std::vector<int> vec = {5, 2, 8, 1, 9};

       // 基本lambda
       auto print = [](int x) { std::cout << x << " "; };
       std::for_each(vec.begin(), vec.end(), print);
       // 输出: 5 2 8 1 9

       // 捕获变量
       int threshold = 5;
       auto count = std::count_if(vec.begin(), vec.end(),
           [threshold](int x) { return x > threshold; });
       std::cout << "\nCount > " << threshold << ": " << count << std::endl;
       // 输出: Count > 5: 2

       // 值捕获 vs 引用捕获
       int sum = 0;
       std::for_each(vec.begin(), vec.end(),
           [&sum](int x) { sum += x; });  // [&] 引用捕获所有外部变量
       std::cout << "Sum: " << sum << std::endl;
       // 输出: Sum: 25

       // 修改捕获的变量（需要mutable）
       int multiplier = 2;
       auto multiply = [multiplier](int x) mutable {
           multiplier *= 2;  // 只修改lambda内部的副本
           return x * multiplier;
       };
       std::cout << "Result: " << multiply(10) << std::endl;  // 40
       std::cout << "Original multiplier: " << multiplier << std::endl;  // 2

       // 返回类型推导
       auto add = [](int a, int b) -> int { return a + b; };
       std::cout << "3 + 4 = " << add(3, 4) << std::endl;  // 7

       // 泛型lambda（C++14）
       auto generic_add = [](auto a, auto b) { return a + b; };
       std::cout << "1.5 + 2.5 = " << generic_add(1.5, 2.5) << std::endl;  // 4.0
       std::cout << "hello + world = " << generic_add(std::string("hello"), std::string(" world")) << std::endl;

       return 0;
   }
   ```

4. **智能指针**：
   ```cpp
   #include <iostream>
   #include <memory>

   class Player {
   public:
       std::string name;
       int hp;

       Player(const std::string& n, int h) : name(n), hp(h) {
           std::cout << "Player " << name << " created" << std::endl;
       }

       ~Player() {
           std::cout << "Player " << name << " destroyed" << std::endl;
       }

       void takeDamage(int damage) {
           hp -= damage;
           std::cout << name << " HP: " << hp << std::endl;
       }
   };

   int main() {
       // unique_ptr - 独占所有权
       {
           std::unique_ptr<Player> player1 = std::make_unique<Player>("Alice", 100);
           player1->takeDamage(20);

           // std::unique_ptr<Player> player2 = player1;  // 错误！不能拷贝
           std::unique_ptr<Player> player2 = std::move(player1);  // 可以移动
           // player1现在为nullptr

           if (!player1) {
               std::cout << "player1 is null" << std::endl;
           }
       }  // 离开作用域，player2自动销毁

       std::cout << "\n--- shared_ptr demo ---\n" << std::endl;

       // shared_ptr - 共享所有权（引用计数）
       {
           std::shared_ptr<Player> player1 = std::make_shared<Player>("Bob", 100);
           std::cout << "Reference count: " << player1.use_count() << std::endl;  // 1

           {
               std::shared_ptr<Player> player2 = player1;  // 可以拷贝
               std::cout << "Reference count: " << player1.use_count() << std::endl;  // 2

               player2->takeDamage(30);
           }  // player2离开作用域，引用计数-1

           std::cout << "Reference count: " << player1.use_count() << std::endl;  // 1
       }  // player1离开作用域，引用计数归0，对象销毁

       std::cout << "\n--- weak_ptr demo ---\n" << std::endl;

       // weak_ptr - 不增加引用计数（避免循环引用）
       {
           std::shared_ptr<Player> player = std::make_shared<Player>("Charlie", 100);
           std::weak_ptr<Player> weak_player = player;

           std::cout << "Reference count: " << player.use_count() << std::endl;  // 1（weak_ptr不增加）

           // 使用weak_ptr前需要lock
           if (auto locked = weak_player.lock()) {
               locked->takeDamage(40);
               std::cout << "Reference count during lock: " << player.use_count() << std::endl;  // 2
           }

           std::cout << "Reference count: " << player.use_count() << std::endl;  // 1
       }

       return 0;
   }
   ```

5. **初始化列表与统一初始化**：
   ```cpp
   #include <iostream>
   #include <vector>
   #include <map>

   class Config {
   public:
       int maxPlayers;
       std::string serverName;
       std::vector<int> ports;

       // 使用初始化列表
       Config(int max, const std::string& name, const std::vector<int>& p)
           : maxPlayers(max), serverName(name), ports(p) {
       }
   };

   int main() {
       // 统一初始化（C++11）
       int a{10};
       int b = {20};
       int c(30);

       // 容器初始化
       std::vector<int> vec1 = {1, 2, 3, 4, 5};
       std::vector<int> vec2{1, 2, 3, 4, 5};

       std::map<std::string, int> scores{
           {"Alice", 100},
           {"Bob", 95},
           {"Charlie", 88}
       };

       // 自定义类型
       Config config{100, "MyServer", {8080, 8081, 8082}};

       // 聚合初始化
       struct Point { int x, y, z; };
       Point p1{1, 2, 3};

       return 0;
   }
   ```

**实践任务**：
实现一个简单的游戏背包系统：
```cpp
// Inventory.h
#pragma once
#include <memory>
#include <vector>
#include <string>
#include <algorithm>

class Item {
public:
    std::string name;
    int id;
    int stackSize;

    Item(int id, const std::string& name, int stack = 1)
        : id(id), name(name), stackSize(stack) {}

    virtual ~Item() = default;
};

class Inventory {
private:
    std::vector<std::shared_ptr<Item>> items;
    size_t capacity;

public:
    Inventory(size_t cap) : capacity(cap) {
        items.reserve(cap);
    }

    // 添加物品
    bool addItem(std::shared_ptr<Item> item) {
        if (items.size() >= capacity) {
            return false;
        }
        items.push_back(std::move(item));
        return true;
    }

    // 移除物品
    bool removeItem(int itemId) {
        auto it = std::remove_if(items.begin(), items.end(),
            [itemId](const std::shared_ptr<Item>& item) {
                return item->id == itemId;
            });

        if (it != items.end()) {
            items.erase(it, items.end());
            return true;
        }
        return false;
    }

    // 查找物品
    std::shared_ptr<Item> findItem(int itemId) const {
        auto it = std::find_if(items.begin(), items.end(),
            [itemId](const std::shared_ptr<Item>& item) {
                return item->id == itemId;
            });

        return (it != items.end()) ? *it : nullptr;
    }

    // 遍历物品
    void forEach(std::function<void(const std::shared_ptr<Item>&)> func) const {
        std::for_each(items.begin(), items.end(), func);
    }

    size_t size() const { return items.size(); }
    size_t getCapacity() const { return capacity; }
};

// main.cpp
int main() {
    Inventory inventory(10);

    // 添加物品
    inventory.addItem(std::make_shared<Item>(1, "Sword", 1));
    inventory.addItem(std::make_shared<Item>(2, "Potion", 99));
    inventory.addItem(std::make_shared<Item>(3, "Shield", 1));

    // 遍历
    inventory.forEach([](const std::shared_ptr<Item>& item) {
        std::cout << "Item: " << item->name << " x" << item->stackSize << std::endl;
    });

    // 查找
    auto item = inventory.findItem(2);
    if (item) {
        std::cout << "Found: " << item->name << std::endl;
    }

    // 移除
    inventory.removeItem(1);
    std::cout << "Inventory size: " << inventory.size() << std::endl;

    return 0;
}
```

**作业**：
- [ ] 实现一个支持移动语义的Vector类
- [ ] 用lambda表达式实现一个简单的事件系统
- [ ] 完成背包系统（支持物品堆叠、排序）
- [ ] 学习std::function和std::bind

**检查点**：
- [ ] 能解释右值引用和std::move的区别
- [ ] 知道何时使用unique_ptr、shared_ptr、weak_ptr
- [ ] 熟练使用lambda表达式
- [ ] 理解auto的使用场景和限制

#### **Day 2：STL容器深度使用**

**学习目标**：
- 掌握vector、list、deque的使用场景
- 理解map、unordered_map的区别
- 学习set、multiset、multimap
- 掌握容器选择的性能考量

**学习内容**：

1. **序列容器对比**：
   ```cpp
   #include <iostream>
   #include <vector>
   #include <list>
   #include <deque>
   #include <chrono>

   // 性能测试
   template<typename Container>
   void testPerformance(const std::string& name) {
       Container container;
       auto start = std::chrono::high_resolution_clock::now();

       // 尾部插入10万个元素
       for (int i = 0; i < 100000; ++i) {
           container.push_back(i);
       }

       auto end = std::chrono::high_resolution_clock::now();
       auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

       std::cout << name << " push_back: " << duration.count() << " µs" << std::endl;

       // 随机访问测试（仅对vector和deque）
       if constexpr (std::is_same_v<Container, std::vector<int>> || std::is_same_v<Container, std::deque<int>>) {
           start = std::chrono::high_resolution_clock::now();
           long long sum = 0;
           for (size_t i = 0; i < container.size(); ++i) {
               sum += container[i];
           }
           end = std::chrono::high_resolution_clock::now();
           duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
           std::cout << name << " random access: " << duration.count() << " µs" << std::endl;
       }
   }

   int main() {
       testPerformance<std::vector<int>>("vector");
       testPerformance<std::list<int>>("list");
       testPerformance<std::deque<int>>("deque");

       /*
       典型输出：
       vector push_back: 2500 µs
       vector random access: 150 µs
       list push_back: 15000 µs
       deque push_back: 3500 µs
       deque random access: 200 µs

       结论：
       - vector：随机访问最快，内存连续
       - list：插入删除快（中间位置），但无随机访问
       - deque：两端操作快，随机访问比vector慢一点
       */

       return 0;
   }
   ```

2. **关联容器详解**：
   ```cpp
   #include <iostream>
   #include <map>
   #include <unordered_map>
   #include <set>

   // 游戏中的应用：玩家管理
   struct Player {
       int id;
       std::string name;
       int level;
       int score;

       Player(int i, const std::string& n, int l, int s)
           : id(i), name(n), level(l), score(s) {}
   };

   int main() {
       // map vs unordered_map
       // map：红黑树实现，有序，查找O(log n)
       // unordered_map：哈希表实现，无序，查找O(1)平均

       std::map<int, Player> orderedPlayers;  // 按ID有序
       std::unordered_map<int, Player> hashPlayers;  // 哈希查找

       // 添加玩家
       orderedPlayers.emplace(1001, Player(1001, "Alice", 50, 10000));
       orderedPlayers.emplace(1003, Player(1003, "Bob", 45, 8500));
       orderedPlayers.emplace(1002, Player(1002, "Charlie", 60, 15000));

       // map会自动按key排序
       std::cout << "Ordered players:" << std::endl;
       for (const auto& [id, player] : orderedPlayers) {
           std::cout << "ID: " << id << ", Name: " << player.name << std::endl;
       }
       // 输出顺序：1001, 1002, 1003

       // unordered_map更快但无序
       hashPlayers.emplace(1001, Player(1001, "Alice", 50, 10000));

       // 查找
       if (auto it = orderedPlayers.find(1002); it != orderedPlayers.end()) {
           std::cout << "Found player: " << it->second.name << std::endl;
       }

       // set：自动去重+有序
       std::set<int> scores = {100, 85, 92, 100, 88, 92};
       std::cout << "Unique scores: ";
       for (int score : scores) {
           std::cout << score << " ";  // 85 88 92 100
       }
       std::cout << std::endl;

       // multimap：允许重复key（用于排行榜）
       std::multimap<int, std::string, std::greater<int>> leaderboard;
       leaderboard.insert({10000, "Alice"});
       leaderboard.insert({8500, "Bob"});
       leaderboard.insert({15000, "Charlie"});
       leaderboard.insert({15000, "David"});  // 相同分数

       std::cout << "Leaderboard (top 3):" << std::endl;
       int rank = 1;
       for (const auto& [score, name] : leaderboard) {
           std::cout << rank++ << ". " << name << ": " << score << std::endl;
           if (rank > 3) break;
       }

       return 0;
   }
   ```

3. **容器适配器**：
   ```cpp
   #include <iostream>
   #include <queue>
   #include <stack>

   // 游戏中的应用

   // 1. 队列：消息队列
   struct GameMessage {
       enum Type { MOVE, ATTACK, CHAT };
       Type type;
       int playerId;
       std::string content;
   };

   void processMessageQueue() {
       std::queue<GameMessage> msgQueue;

       // 生产消息
       msgQueue.push({GameMessage::MOVE, 1001, "x:100,y:200"});
       msgQueue.push({GameMessage::ATTACK, 1002, "target:1001"});
       msgQueue.push({GameMessage::CHAT, 1001, "Hello!"});

       // 消费消息（FIFO）
       while (!msgQueue.empty()) {
           GameMessage msg = msgQueue.front();
           msgQueue.pop();

           std::cout << "Processing message from player " << msg.playerId << std::endl;
           // 处理消息...
       }
   }

   // 2. 优先队列：技能冷却
   struct Skill {
       std::string name;
       int cooldown;  // 剩余冷却时间（秒）

       // 冷却时间短的优先（小顶堆）
       bool operator<(const Skill& other) const {
           return cooldown > other.cooldown;  // 注意：反向
       }
   };

   void updateSkillCooldowns() {
       std::priority_queue<Skill> skillQueue;

       skillQueue.push({"Fireball", 10});
       skillQueue.push({"Heal", 5});
       skillQueue.push({"Shield", 15});

       std::cout << "Skills ready order:" << std::endl;
       while (!skillQueue.empty()) {
           Skill skill = skillQueue.top();
           skillQueue.pop();
           std::cout << skill.name << " (cooldown: " << skill.cooldown << "s)" << std::endl;
       }
       // 输出：Heal(5), Fireball(10), Shield(15)
   }

   // 3. 栈：状态管理
   enum GameState { MENU, GAMEPLAY, PAUSE, SETTINGS };

   class GameStateManager {
   private:
       std::stack<GameState> stateStack;

   public:
       void pushState(GameState state) {
           stateStack.push(state);
           std::cout << "Entered state: " << state << std::endl;
       }

       void popState() {
           if (!stateStack.empty()) {
               GameState state = stateStack.top();
               stateStack.pop();
               std::cout << "Exited state: " << state << std::endl;
           }
       }

       GameState getCurrentState() const {
           return stateStack.empty() ? MENU : stateStack.top();
       }
   };

   int main() {
       GameStateManager gsm;
       gsm.pushState(MENU);       // 主菜单
       gsm.pushState(GAMEPLAY);   // 开始游戏
       gsm.pushState(PAUSE);      // 暂停
       gsm.popState();            // 恢复游戏
       gsm.popState();            // 回到主菜单

       processMessageQueue();
       updateSkillCooldowns();

       return 0;
   }
   ```

4. **STL算法**：
   ```cpp
   #include <iostream>
   #include <vector>
   #include <algorithm>
   #include <numeric>

   struct Monster {
       std::string name;
       int hp;
       int level;
   };

   int main() {
       std::vector<Monster> monsters = {
           {"Goblin", 50, 5},
           {"Orc", 100, 10},
           {"Dragon", 500, 50},
           {"Slime", 20, 1},
           {"Boss", 1000, 99}
       };

       // 1. sort - 排序
       std::sort(monsters.begin(), monsters.end(),
           [](const Monster& a, const Monster& b) {
               return a.level < b.level;
           });

       // 2. find_if - 查找
       auto it = std::find_if(monsters.begin(), monsters.end(),
           [](const Monster& m) { return m.hp > 100; });

       if (it != monsters.end()) {
           std::cout << "Found powerful monster: " << it->name << std::endl;
       }

       // 3. count_if - 统计
       int lowLevelCount = std::count_if(monsters.begin(), monsters.end(),
           [](const Monster& m) { return m.level < 10; });
       std::cout << "Low level monsters: " << lowLevelCount << std::endl;

       // 4. transform - 转换（给所有怪物加血）
       std::transform(monsters.begin(), monsters.end(), monsters.begin(),
           [](Monster m) {
               m.hp *= 1.5;  // 增加50% HP
               return m;
           });

       // 5. accumulate - 累加（总血量）
       int totalHp = std::accumulate(monsters.begin(), monsters.end(), 0,
           [](int sum, const Monster& m) {
               return sum + m.hp;
           });
       std::cout << "Total HP: " << totalHp << std::endl;

       // 6. partition - 分区（分离boss）
       auto partIt = std::partition(monsters.begin(), monsters.end(),
           [](const Monster& m) { return m.level < 50; });

       std::cout << "Normal monsters:" << std::endl;
       std::for_each(monsters.begin(), partIt,
           [](const Monster& m) { std::cout << "  " << m.name << std::endl; });

       std::cout << "Boss monsters:" << std::endl;
       std::for_each(partIt, monsters.end(),
           [](const Monster& m) { std::cout << "  " << m.name << std::endl; });

       // 7. remove_if + erase - 删除（清理死亡怪物）
       monsters.erase(
           std::remove_if(monsters.begin(), monsters.end(),
               [](const Monster& m) { return m.hp <= 0; }),
           monsters.end()
       );

       return 0;
   }
   ```

**实践项目**：游戏背包系统升级版
```cpp
// AdvancedInventory.h
#pragma once
#include <vector>
#include <unordered_map>
#include <algorithm>
#include <memory>

class Item {
public:
    int id;
    std::string name;
    int type;  // 0=consumable, 1=equipment, 2=quest
    int stackSize;
    int maxStack;

    Item(int id, const std::string& name, int type, int stack = 1, int maxStack = 99)
        : id(id), name(name), type(type), stackSize(stack), maxStack(maxStack) {}

    virtual ~Item() = default;
};

class Inventory {
private:
    std::vector<std::shared_ptr<Item>> items;
    std::unordered_map<int, std::vector<size_t>> itemIndex;  // id -> positions
    size_t capacity;

public:
    Inventory(size_t cap) : capacity(cap) {
        items.reserve(cap);
    }

    // 智能添加（自动堆叠）
    bool addItem(std::shared_ptr<Item> item) {
        // 1. 尝试堆叠到现有物品
        if (auto positions = itemIndex.find(item->id); positions != itemIndex.end()) {
            for (size_t pos : positions->second) {
                auto& existingItem = items[pos];
                if (existingItem->stackSize < existingItem->maxStack) {
                    int space = existingItem->maxStack - existingItem->stackSize;
                    int toAdd = std::min(space, item->stackSize);
                    existingItem->stackSize += toAdd;
                    item->stackSize -= toAdd;

                    if (item->stackSize == 0) {
                        return true;
                    }
                }
            }
        }

        // 2. 需要新格子
        if (items.size() >= capacity) {
            return false;
        }

        size_t pos = items.size();
        items.push_back(item);
        itemIndex[item->id].push_back(pos);

        return true;
    }

    // 移除指定数量
    bool removeItem(int itemId, int count) {
        auto it = itemIndex.find(itemId);
        if (it == itemIndex.end()) {
            return false;
        }

        for (size_t pos : it->second) {
            auto& item = items[pos];
            if (count <= 0) break;

            int toRemove = std::min(count, item->stackSize);
            item->stackSize -= toRemove;
            count -= toRemove;

            if (item->stackSize == 0) {
                // 标记为空（实际删除在整理时）
                item = nullptr;
            }
        }

        if (count > 0) {
            return false;  // 数量不足
        }

        // 清理空格子
        compact();
        return true;
    }

    // 排序
    void sortByType() {
        std::stable_sort(items.begin(), items.end(),
            [](const std::shared_ptr<Item>& a, const std::shared_ptr<Item>& b) {
                if (!a) return false;
                if (!b) return true;
                return a->type < b->type;
            });
        rebuildIndex();
    }

    void sortByName() {
        std::stable_sort(items.begin(), items.end(),
            [](const std::shared_ptr<Item>& a, const std::shared_ptr<Item>& b) {
                if (!a) return false;
                if (!b) return true;
                return a->name < b->name;
            });
        rebuildIndex();
    }

    // 整理（移除空格）
    void compact() {
        items.erase(
            std::remove_if(items.begin(), items.end(),
                [](const std::shared_ptr<Item>& item) { return !item; }),
            items.end()
        );
        rebuildIndex();
    }

    // 统计
    int countItem(int itemId) const {
        auto it = itemIndex.find(itemId);
        if (it == itemIndex.end()) {
            return 0;
        }

        int total = 0;
        for (size_t pos : it->second) {
            if (pos < items.size() && items[pos]) {
                total += items[pos]->stackSize;
            }
        }
        return total;
    }

    // 按类型过滤
    std::vector<std::shared_ptr<Item>> getItemsByType(int type) const {
        std::vector<std::shared_ptr<Item>> result;
        std::copy_if(items.begin(), items.end(), std::back_inserter(result),
            [type](const std::shared_ptr<Item>& item) {
                return item && item->type == type;
            });
        return result;
    }

private:
    void rebuildIndex() {
        itemIndex.clear();
        for (size_t i = 0; i < items.size(); ++i) {
            if (items[i]) {
                itemIndex[items[i]->id].push_back(i);
            }
        }
    }
};
```

**作业**：
- [ ] 实现Inventory的序列化/反序列化（保存/加载）
- [ ] 添加物品交易功能（两个Inventory之间转移）
- [ ] 实现物品合成系统（多个物品合成一个）
- [ ] 性能测试：对比vector、list在不同操作下的性能

**检查点**：
- [ ] 能说出至少5种STL容器的使用场景
- [ ] 理解map和unordered_map的底层实现区别
- [ ] 熟练使用STL算法（sort、find、transform等）
- [ ] 能设计高效的数据结构满足游戏需求

---

#### **Day 3-7：多线程编程、智能指针与设计模式**

**Day 3：C++11多线程基础**
- thread、mutex、lock_guard
- 条件变量condition_variable
- 原子操作atomic
- 线程池实现

**Day 4：线程同步与并发容器**
- 读写锁shared_mutex
- 无锁编程基础
- 线程安全的单例模式

**Day 5：异步编程**
- future、promise
- async与packaged_task
- 协程基础（C++20）

**Day 6：常用设计模式（上）**
- 单例模式（游戏管理器）
- 工厂模式（怪物生成）
- 观察者模式（事件系统）

**Day 7：常用设计模式（下）**
- 命令模式（技能系统）
- 策略模式（AI行为）
- 对象池模式（子弹池）

---

### **进阶专题：GCC内联汇编**

#### **为什么游戏服务器需要内联汇编**

在游戏服务器开发中，某些性能关键路径可能需要使用内联汇编来获得极致性能：
- **高精度计时**：使用RDTSC指令获取CPU时钟周期
- **原子操作**：实现无锁数据结构
- **CPU特性检测**：使用CPUID指令
- **内存屏障**：确保内存操作顺序
- **SIMD优化**：批量数据处理（碰撞检测、向量运算）

**学习目标**：
- 理解GCC内联汇编语法（AT&T vs Intel）
- 掌握常用汇编指令在游戏服务器中的应用
- 学会在C++中嵌入汇编代码
- 理解何时使用内联汇编

---

#### **1. GCC内联汇编基础语法**

**AT&T语法 vs Intel语法**：
```cpp
// Intel语法：操作数顺序是 dest, src
// mov eax, ebx  ; ebx -> eax

// AT&T语法：操作数顺序是 src, dest
// movl %ebx, %eax  ; ebx -> eax

// GCC默认使用AT&T语法，但可以切换到Intel语法
```

**基本格式**：
```cpp
#include <iostream>

int main() {
    int input = 10;
    int output = 0;

    // 基本内联汇编格式
    __asm__ __volatile__ (
        "汇编指令"
        : 输出操作数列表
        : 输入操作数列表
        : 破坏描述符列表
    );

    // 示例：将input的值加上5，结果存入output
    __asm__ __volatile__ (
        "movl %1, %%eax\n\t"      // 将input移动到eax
        "addl $5, %%eax\n\t"      // eax += 5
        "movl %%eax, %0\n\t"      // 将eax移动到output
        : "=r" (output)            // %0：输出操作数
        : "r" (input)              // %1：输入操作数
        : "%eax"                   // 告诉编译器eax被修改了
    );

    std::cout << "Result: " << output << std::endl;  // 15
    return 0;
}
```

**约束符说明**：
```cpp
/*
常用约束符：
"r" : 通用寄存器
"a" : eax/rax
"b" : ebx/rbx
"c" : ecx/rcx
"d" : edx/rdx
"m" : 内存操作数
"i" : 立即数
"=" : 只写（输出）
"+" : 可读写（输入输出）
"&" : 早期修改（在输入前就修改）
*/

int a = 5, b = 10, result;

__asm__ (
    "addl %2, %1\n\t"
    "movl %1, %0"
    : "=r" (result)      // %0: 输出到result
    : "r" (a), "r" (b)   // %1=a, %2=b
    :
);
// result = a + b = 15
```

---

#### **2. 高精度计时 - RDTSC指令**

**RDTSC（Read Time-Stamp Counter）**用于读取CPU时钟周期数，是游戏服务器中最常用的高精度计时方式：

```cpp
#include <iostream>
#include <thread>
#include <chrono>

// 读取CPU时间戳计数器
inline uint64_t rdtsc() {
    uint32_t lo, hi;
    __asm__ __volatile__ (
        "rdtsc"
        : "=a" (lo), "=d" (hi)
    );
    return ((uint64_t)hi << 32) | lo;
}

// RDTSCP（更精确，带序列化）
inline uint64_t rdtscp() {
    uint32_t lo, hi;
    __asm__ __volatile__ (
        "rdtscp"
        : "=a" (lo), "=d" (hi)
        :: "%rcx"  // rdtscp会修改ecx
    );
    return ((uint64_t)hi << 32) | lo;
}

// 游戏服务器性能计时器
class PerformanceTimer {
private:
    uint64_t startCycles;
    double cpuFreqGHz;  // CPU频率（GHz）

public:
    PerformanceTimer(double freq = 3.0) : cpuFreqGHz(freq) {
        startCycles = rdtsc();
    }

    void reset() {
        startCycles = rdtsc();
    }

    // 返回经过的纳秒数
    double elapsedNanoseconds() const {
        uint64_t endCycles = rdtsc();
        uint64_t cycles = endCycles - startCycles;
        return cycles / cpuFreqGHz;
    }

    // 返回经过的微秒数
    double elapsedMicroseconds() const {
        return elapsedNanoseconds() / 1000.0;
    }
};

// 使用示例：测量函数性能
void expensiveOperation() {
    int sum = 0;
    for (int i = 0; i < 1000000; ++i) {
        sum += i;
    }
}

int main() {
    PerformanceTimer timer(3.0);  // 假设CPU是3GHz

    expensiveOperation();

    std::cout << "Operation took: "
              << timer.elapsedMicroseconds() << " µs" << std::endl;

    // 游戏服务器应用：测量数据包处理时间
    timer.reset();
    // processPacket(packet);
    double packetProcessTime = timer.elapsedNanoseconds();

    if (packetProcessTime > 1000000) {  // 超过1ms
        std::cout << "WARNING: Packet processing too slow!" << std::endl;
    }

    return 0;
}
```

**获取CPU频率**：
```cpp
#include <fstream>
#include <string>

// 从/proc/cpuinfo读取CPU频率
double getCPUFrequency() {
    std::ifstream cpuinfo("/proc/cpuinfo");
    std::string line;

    while (std::getline(cpuinfo, line)) {
        if (line.find("cpu MHz") != std::string::npos) {
            size_t pos = line.find(":");
            if (pos != std::string::npos) {
                double mhz = std::stod(line.substr(pos + 1));
                return mhz / 1000.0;  // 转换为GHz
            }
        }
    }
    return 3.0;  // 默认3GHz
}
```

---

#### **3. CPU特性检测 - CPUID指令**

**CPUID指令**用于查询CPU支持的功能，在游戏服务器中可用于：
- 检测是否支持SSE/AVX指令集
- 获取CPU厂商信息
- 检测缓存大小

```cpp
#include <iostream>
#include <cstring>
#include <array>

// CPUID包装函数
void cpuid(uint32_t eax, uint32_t ecx, uint32_t* regs) {
    __asm__ __volatile__ (
        "cpuid"
        : "=a" (regs[0]), "=b" (regs[1]),
          "=c" (regs[2]), "=d" (regs[3])
        : "a" (eax), "c" (ecx)
    );
}

// 获取CPU厂商字符串
std::string getCPUVendor() {
    uint32_t regs[4];
    char vendor[13];

    cpuid(0, 0, regs);

    memcpy(vendor, &regs[1], 4);      // EBX
    memcpy(vendor + 4, &regs[3], 4);  // EDX
    memcpy(vendor + 8, &regs[2], 4);  // ECX
    vendor[12] = '\0';

    return std::string(vendor);
}

// 检测CPU特性
struct CPUFeatures {
    bool sse;
    bool sse2;
    bool sse3;
    bool ssse3;
    bool sse41;
    bool sse42;
    bool avx;
    bool avx2;
    bool fma;
    bool aes;
    bool rdrand;
    bool popcnt;
};

CPUFeatures detectCPUFeatures() {
    CPUFeatures features{};
    uint32_t regs[4];

    // 调用CPUID功能1
    cpuid(1, 0, regs);

    // ECX寄存器中的特性位
    features.sse3   = (regs[2] & (1 << 0)) != 0;
    features.ssse3  = (regs[2] & (1 << 9)) != 0;
    features.fma    = (regs[2] & (1 << 12)) != 0;
    features.sse41  = (regs[2] & (1 << 19)) != 0;
    features.sse42  = (regs[2] & (1 << 20)) != 0;
    features.aes    = (regs[2] & (1 << 25)) != 0;
    features.avx    = (regs[2] & (1 << 28)) != 0;
    features.rdrand = (regs[2] & (1 << 30)) != 0;
    features.popcnt = (regs[2] & (1 << 23)) != 0;

    // EDX寄存器中的特性位
    features.sse    = (regs[3] & (1 << 25)) != 0;
    features.sse2   = (regs[3] & (1 << 26)) != 0;

    // 调用CPUID功能7检测AVX2
    cpuid(7, 0, regs);
    features.avx2   = (regs[1] & (1 << 5)) != 0;

    return features;
}

// 游戏服务器应用：根据CPU特性选择优化版本
void processCollisions() {
    static CPUFeatures features = detectCPUFeatures();

    if (features.avx2) {
        // 使用AVX2优化版本（一次处理8个float）
        // processCollisions_AVX2();
        std::cout << "Using AVX2 optimized version" << std::endl;
    } else if (features.sse2) {
        // 使用SSE2优化版本（一次处理4个float）
        // processCollisions_SSE2();
        std::cout << "Using SSE2 optimized version" << std::endl;
    } else {
        // 使用标准版本
        // processCollisions_Standard();
        std::cout << "Using standard version" << std::endl;
    }
}

int main() {
    std::cout << "CPU Vendor: " << getCPUVendor() << std::endl;

    CPUFeatures features = detectCPUFeatures();

    std::cout << "CPU Features:" << std::endl;
    std::cout << "  SSE:    " << (features.sse ? "Yes" : "No") << std::endl;
    std::cout << "  SSE2:   " << (features.sse2 ? "Yes" : "No") << std::endl;
    std::cout << "  SSE4.1: " << (features.sse41 ? "Yes" : "No") << std::endl;
    std::cout << "  AVX:    " << (features.avx ? "Yes" : "No") << std::endl;
    std::cout << "  AVX2:   " << (features.avx2 ? "Yes" : "No") << std::endl;
    std::cout << "  AES:    " << (features.aes ? "Yes" : "No") << std::endl;
    std::cout << "  POPCNT: " << (features.popcnt ? "Yes" : "No") << std::endl;

    processCollisions();

    return 0;
}
```

---

#### **4. 原子操作与内存屏障**

**内存屏障**确保内存操作的顺序，在多线程游戏服务器中非常重要：

```cpp
#include <iostream>
#include <thread>
#include <atomic>

// 编译器屏障（防止编译器重排）
#define compiler_barrier() __asm__ __volatile__("" ::: "memory")

// CPU内存屏障
inline void memory_barrier_full() {
    __asm__ __volatile__("mfence" ::: "memory");
}

inline void memory_barrier_read() {
    __asm__ __volatile__("lfence" ::: "memory");
}

inline void memory_barrier_write() {
    __asm__ __volatile__("sfence" ::: "memory");
}

// 无锁栈实现（使用CMPXCHG指令）
template<typename T>
class LockFreeStack {
private:
    struct Node {
        T data;
        Node* next;
        Node(const T& val) : data(val), next(nullptr) {}
    };

    Node* head;

public:
    LockFreeStack() : head(nullptr) {}

    // 使用CAS（Compare-And-Swap）实现无锁push
    void push(const T& value) {
        Node* newNode = new Node(value);
        Node* oldHead;

        do {
            oldHead = head;
            newNode->next = oldHead;

            // 使用内联汇编实现CAS
            bool success;
            __asm__ __volatile__ (
                "lock cmpxchgq %3, %1\n\t"
                "sete %0"
                : "=q" (success), "+m" (head), "+a" (oldHead)
                : "r" (newNode)
                : "cc", "memory"
            );

            if (success) break;

        } while (true);
    }

    // 使用C++标准库的原子操作（推荐）
    void push_standard(const T& value) {
        Node* newNode = new Node(value);
        Node* oldHead = head;

        do {
            newNode->next = oldHead;
        } while (!__sync_bool_compare_and_swap(&head, oldHead, newNode));
    }
};

// 自旋锁实现
class SpinLock {
private:
    volatile int locked;

public:
    SpinLock() : locked(0) {}

    void lock() {
        while (1) {
            // 尝试获取锁
            int expected = 0;
            bool success;

            __asm__ __volatile__ (
                "lock cmpxchgl %2, %1\n\t"
                "sete %0"
                : "=q" (success), "+m" (locked), "+a" (expected)
                : "r" (1)
                : "cc", "memory"
            );

            if (success) break;

            // CPU暂停指令，减少功耗
            __asm__ __volatile__("pause" ::: "memory");
        }
    }

    void unlock() {
        // 释放锁
        __asm__ __volatile__ (
            "movl $0, %0"
            : "=m" (locked)
            :: "memory"
        );
    }
};

// PAUSE指令优化自旋等待
inline void cpu_relax() {
    __asm__ __volatile__("pause" ::: "memory");
}

// 游戏服务器应用：高性能计数器
class PerformanceCounter {
private:
    volatile long long counter;

public:
    PerformanceCounter() : counter(0) {}

    // 原子递增（使用XADD指令）
    long long increment() {
        long long result;
        __asm__ __volatile__ (
            "lock xaddq %0, %1"
            : "=r" (result), "+m" (counter)
            : "0" (1LL)
            : "memory"
        );
        return result + 1;
    }

    // 原子递减
    long long decrement() {
        long long result;
        __asm__ __volatile__ (
            "lock xaddq %0, %1"
            : "=r" (result), "+m" (counter)
            : "0" (-1LL)
            : "memory"
        );
        return result - 1;
    }

    long long get() const {
        return counter;
    }
};

int main() {
    // 测试自旋锁
    SpinLock spinlock;
    int sharedData = 0;

    auto worker = [&]() {
        for (int i = 0; i < 100000; ++i) {
            spinlock.lock();
            sharedData++;
            spinlock.unlock();
        }
    };

    std::thread t1(worker);
    std::thread t2(worker);

    t1.join();
    t2.join();

    std::cout << "Shared data: " << sharedData << std::endl;  // 应该是200000

    // 测试性能计数器
    PerformanceCounter counter;

    auto incrementer = [&]() {
        for (int i = 0; i < 100000; ++i) {
            counter.increment();
        }
    };

    std::thread t3(incrementer);
    std::thread t4(incrementer);

    t3.join();
    t4.join();

    std::cout << "Counter: " << counter.get() << std::endl;  // 应该是200000

    return 0;
}
```

---

#### **5. SIMD优化示例（SSE）**

**SSE（Streaming SIMD Extensions）**可用于批量数据处理，在游戏服务器中用于向量运算、碰撞检测等：

```cpp
#include <iostream>
#include <cmath>
#include <chrono>
#include <emmintrin.h>  // SSE2

// 向量点积 - 标准版本
float dotProduct_Standard(const float* a, const float* b, int n) {
    float sum = 0.0f;
    for (int i = 0; i < n; ++i) {
        sum += a[i] * b[i];
    }
    return sum;
}

// 向量点积 - SSE优化版本（一次处理4个float）
float dotProduct_SSE(const float* a, const float* b, int n) {
    __m128 sum = _mm_setzero_ps();  // 初始化为0

    int i = 0;
    for (; i + 3 < n; i += 4) {
        __m128 va = _mm_loadu_ps(&a[i]);    // 加载4个float
        __m128 vb = _mm_loadu_ps(&b[i]);
        __m128 vmul = _mm_mul_ps(va, vb);   // 并行乘法
        sum = _mm_add_ps(sum, vmul);        // 累加
    }

    // 水平求和
    float result[4];
    _mm_storeu_ps(result, sum);
    float total = result[0] + result[1] + result[2] + result[3];

    // 处理剩余元素
    for (; i < n; ++i) {
        total += a[i] * b[i];
    }

    return total;
}

// 使用纯汇编实现（SSE2）
float dotProduct_ASM(const float* a, const float* b, int n) {
    float result = 0.0f;

    __asm__ __volatile__ (
        "xorps %%xmm0, %%xmm0\n\t"       // xmm0 = 0 (累加器)
        "movl $0, %%eax\n\t"              // i = 0

        "1:\n\t"                          // 循环开始
        "cmpl %%eax, %3\n\t"              // 比较 i 和 n-3
        "jle 2f\n\t"                      // 如果 i >= n-3，跳出

        "movups (%%rsi, %%rax, 4), %%xmm1\n\t"  // 加载a[i:i+3]
        "movups (%%rdi, %%rax, 4), %%xmm2\n\t"  // 加载b[i:i+3]
        "mulps %%xmm2, %%xmm1\n\t"              // xmm1 *= xmm2
        "addps %%xmm1, %%xmm0\n\t"              // xmm0 += xmm1

        "addl $4, %%eax\n\t"              // i += 4
        "jmp 1b\n\t"                      // 跳回循环开始

        "2:\n\t"                          // 循环结束

        // 水平求和（hadd指令需要SSE3）
        "movaps %%xmm0, %%xmm1\n\t"
        "shufps $0x4E, %%xmm0, %%xmm0\n\t"
        "addps %%xmm1, %%xmm0\n\t"
        "movaps %%xmm0, %%xmm1\n\t"
        "shufps $0xB1, %%xmm0, %%xmm0\n\t"
        "addss %%xmm1, %%xmm0\n\t"

        "movss %%xmm0, %0\n\t"            // 存储结果

        : "=m" (result)
        : "D" (b), "S" (a), "r" (n - 3)
        : "%rax", "%xmm0", "%xmm1", "%xmm2", "memory"
    );

    return result;
}

// 游戏服务器应用：距离计算优化
struct Vector3 {
    float x, y, z;
};

// 批量距离计算 - SSE优化
void calculateDistances_SSE(const Vector3* positions,
                             const Vector3& target,
                             float* distances,
                             int count) {
    __m128 tx = _mm_set1_ps(target.x);
    __m128 ty = _mm_set1_ps(target.y);
    __m128 tz = _mm_set1_ps(target.z);

    for (int i = 0; i < count; i += 4) {
        // 加载4个位置的x, y, z（交错加载）
        // 这里简化处理，实际需要更复杂的加载逻辑
        __m128 dx = _mm_sub_ps(_mm_loadu_ps(&positions[i].x), tx);
        __m128 dy = _mm_sub_ps(_mm_loadu_ps(&positions[i].y), ty);
        __m128 dz = _mm_sub_ps(_mm_loadu_ps(&positions[i].z), tz);

        // 计算距离平方
        __m128 dist2 = _mm_add_ps(
            _mm_add_ps(_mm_mul_ps(dx, dx), _mm_mul_ps(dy, dy)),
            _mm_mul_ps(dz, dz)
        );

        // 开方
        __m128 dist = _mm_sqrt_ps(dist2);

        // 存储结果
        _mm_storeu_ps(&distances[i], dist);
    }
}

int main() {
    const int N = 10000000;
    float* a = new float[N];
    float* b = new float[N];

    for (int i = 0; i < N; ++i) {
        a[i] = static_cast<float>(i);
        b[i] = static_cast<float>(i + 1);
    }

    // 性能对比
    auto start = std::chrono::high_resolution_clock::now();
    float result1 = dotProduct_Standard(a, b, N);
    auto end = std::chrono::high_resolution_clock::now();
    auto duration1 = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

    start = std::chrono::high_resolution_clock::now();
    float result2 = dotProduct_SSE(a, b, N);
    end = std::chrono::high_resolution_clock::now();
    auto duration2 = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

    std::cout << "Standard: " << result1 << " (" << duration1.count() << " µs)" << std::endl;
    std::cout << "SSE:      " << result2 << " (" << duration2.count() << " µs)" << std::endl;
    std::cout << "Speedup:  " << (float)duration1.count() / duration2.count() << "x" << std::endl;

    delete[] a;
    delete[] b;

    return 0;
}
```

---

#### **6. 实用工具函数**

```cpp
// 位操作优化
inline int countBits(uint32_t x) {
    int count;
    __asm__ (
        "popcnt %1, %0"
        : "=r" (count)
        : "r" (x)
    );
    return count;
}

// 前导零计数（CLZ - Count Leading Zeros）
inline int clz(uint32_t x) {
    int count;
    __asm__ (
        "lzcnt %1, %0"
        : "=r" (count)
        : "r" (x)
    );
    return count;
}

// 尾随零计数（CTZ - Count Trailing Zeros）
inline int ctz(uint32_t x) {
    int count;
    __asm__ (
        "tzcnt %1, %0"
        : "=r" (count)
        : "r" (x)
    );
    return count;
}

// 字节交换（大小端转换）
inline uint32_t bswap32(uint32_t x) {
    __asm__ (
        "bswap %0"
        : "+r" (x)
    );
    return x;
}

inline uint64_t bswap64(uint64_t x) {
    __asm__ (
        "bswap %0"
        : "+r" (x)
    );
    return x;
}

// 快速取模（2的幂）
inline uint32_t fastMod(uint32_t value, uint32_t mod) {
    // 仅当mod是2的幂时有效
    return value & (mod - 1);
}
```

---

#### **7. Intel语法示例**

如果你更习惯Intel语法，可以使用`.intel_syntax`指令：

```cpp
#include <iostream>

int main() {
    int a = 10, b = 20, result;

    // Intel语法
    __asm__ (
        ".intel_syntax noprefix\n\t"
        "mov eax, %1\n\t"        // Intel: dest, src
        "add eax, %2\n\t"
        "mov %0, eax\n\t"
        ".att_syntax prefix\n\t"
        : "=r" (result)
        : "r" (a), "r" (b)
        : "%eax"
    );

    std::cout << "Result: " << result << std::endl;  // 30
    return 0;
}
```

---

#### **8. 注意事项与最佳实践**

**何时使用内联汇编**：
- ✅ 性能关键路径（经过性能分析确认）
- ✅ 需要特定CPU指令（RDTSC、CPUID）
- ✅ 实现编译器无法生成的代码
- ✅ 实现无锁数据结构的原子操作

**何时不使用内联汇编**：
- ❌ 编译器能生成同样高效的代码
- ❌ 代码可移植性要求高
- ❌ 没有性能瓶颈证明
- ❌ 维护成本过高

**最佳实践**：
1. **先使用编译器内建函数**（如`__builtin_popcount`），再考虑汇编
2. **使用`-O3 -march=native`编译选项**，让编译器自动生成SIMD代码
3. **性能测试**：用`perf`、`gprof`等工具验证优化效果
4. **保持可读性**：添加详细注释
5. **提供C++备用实现**：用`#ifdef`根据平台选择
6. **使用`volatile`**：防止编译器过度优化
7. **注意寄存器破坏**：正确声明clobber列表

**示例：条件编译**：
```cpp
inline uint64_t rdtsc_portable() {
#if defined(__x86_64__) || defined(__i386__)
    // x86/x64平台使用内联汇编
    uint32_t lo, hi;
    __asm__ __volatile__("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
#elif defined(__aarch64__)
    // ARM64使用系统寄存器
    uint64_t val;
    __asm__ __volatile__("mrs %0, cntvct_el0" : "=r"(val));
    return val;
#else
    // 其他平台使用标准库
    return std::chrono::high_resolution_clock::now().time_since_epoch().count();
#endif
}
```

---

#### **9. 游戏服务器实战应用**

**应用场景汇总**：

| 场景 | 使用的汇编技术 | 性能提升 |
|------|---------------|---------|
| 网络包处理计时 | RDTSC | 纳秒级精度 |
| 玩家位置批量更新 | SSE/AVX | 4-8x |
| 伤害计算（浮点运算） | SSE | 2-4x |
| 碰撞检测（向量运算） | AVX | 4-8x |
| 无锁消息队列 | CAS (CMPXCHG) | 减少锁竞争 |
| 哈希表查找 | POPCNT, CLZ | 10-20% |
| 协议序列化（字节序） | BSWAP | 2x |
| CPU特性检测 | CPUID | 运行时优化 |

**完整示例 - 高性能数据包处理器**：
```cpp
class PacketProcessor {
private:
    PerformanceTimer timer;
    PerformanceCounter processedCount;

public:
    void processPacket(const char* data, size_t len) {
        uint64_t startCycles = rdtsc();

        // 处理数据包...

        uint64_t endCycles = rdtsc();
        uint64_t cycles = endCycles - startCycles;

        if (cycles > 10000) {  // 超过阈值
            // 记录慢包
            logSlowPacket(data, len, cycles);
        }

        processedCount.increment();
    }

    void printStats() {
        std::cout << "Processed packets: " << processedCount.get() << std::endl;
    }
};
```

---

**作业**：
- [ ] 实现一个使用RDTSC的微秒级定时器
- [ ] 使用CPUID检测你的CPU支持哪些指令集
- [ ] 用SSE优化一个向量加法函数，对比性能
- [ ] 实现一个使用CAS的无锁计数器
- [ ] 编写一个性能测试框架，自动对比标准版本和汇编优化版本

**检查点**：
- [ ] 理解AT&T和Intel汇编语法的区别
- [ ] 能使用RDTSC实现高精度计时
- [ ] 了解内存屏障的作用
- [ ] 知道何时应该/不应该使用内联汇编
- [ ] 能解释SSE/AVX的基本原理

---

## 阶段2：网络编程（第4-5周）

### **Week 4：Socket编程与IO多路复用**

#### **Day 8：TCP/UDP基础**

**学习目标**：
- 理解TCP三次握手、四次挥手
- 掌握Socket API使用
- 学习TCP粘包处理
- 理解UDP的特点和应用场景

**学习内容**：

1. **TCP基础服务器**：
   ```cpp
   // SimpleServer.cpp
   #include <iostream>
   #include <sys/socket.h>
   #include <netinet/in.h>
   #include <arpa/inet.h>
   #include <unistd.h>
   #include <cstring>

   class TcpServer {
   private:
       int listenFd;
       int port;

   public:
       TcpServer(int p) : listenFd(-1), port(p) {}

       ~TcpServer() {
           if (listenFd >= 0) {
               close(listenFd);
           }
       }

       bool start() {
           // 1. 创建socket
           listenFd = socket(AF_INET, SOCK_STREAM, 0);
           if (listenFd < 0) {
               std::cerr << "Failed to create socket" << std::endl;
               return false;
           }

           // 2. 设置地址重用
           int opt = 1;
           setsockopt(listenFd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

           // 3. 绑定地址
           sockaddr_in addr{};
           addr.sin_family = AF_INET;
           addr.sin_addr.s_addr = INADDR_ANY;
           addr.sin_port = htons(port);

           if (bind(listenFd, (sockaddr*)&addr, sizeof(addr)) < 0) {
               std::cerr << "Failed to bind" << std::endl;
               return false;
           }

           // 4. 监听
           if (listen(listenFd, 128) < 0) {
               std::cerr << "Failed to listen" << std::endl;
               return false;
           }

           std::cout << "Server started on port " << port << std::endl;
           return true;
       }

       void run() {
           while (true) {
               // 5. 接受连接
               sockaddr_in clientAddr{};
               socklen_t clientLen = sizeof(clientAddr);
               int clientFd = accept(listenFd, (sockaddr*)&clientAddr, &clientLen);

               if (clientFd < 0) {
                   std::cerr << "Failed to accept" << std::endl;
                   continue;
               }

               char ip[INET_ADDRSTRLEN];
               inet_ntop(AF_INET, &clientAddr.sin_addr, ip, sizeof(ip));
               std::cout << "Client connected: " << ip << ":" << ntohs(clientAddr.sin_port) << std::endl;

               // 处理客户端（这里简化为单线程处理）
               handleClient(clientFd);
           }
       }

   private:
       void handleClient(int clientFd) {
           char buffer[1024];

           while (true) {
               // 接收数据
               int n = recv(clientFd, buffer, sizeof(buffer) - 1, 0);

               if (n <= 0) {
                   std::cout << "Client disconnected" << std::endl;
                   break;
               }

               buffer[n] = '\0';
               std::cout << "Received: " << buffer << std::endl;

               // 回显数据
               send(clientFd, buffer, n, 0);
           }

           close(clientFd);
       }
   };

   int main() {
       TcpServer server(8080);
       if (server.start()) {
           server.run();
       }
       return 0;
   }
   ```

2. **TCP粘包处理**：
   ```cpp
   // 游戏服务器常用的协议格式
   #pragma pack(push, 1)
   struct PacketHeader {
       uint16_t length;    // 包总长度（含header）
       uint16_t type;      // 消息类型
       uint32_t sequence;  // 序列号
   };
   #pragma pack(pop)

   class PacketBuffer {
   private:
       std::vector<char> buffer;
       size_t readPos;

   public:
       PacketBuffer() : readPos(0) {
           buffer.reserve(65536);  // 64KB缓冲区
       }

       // 添加接收到的数据
       void append(const char* data, size_t len) {
           buffer.insert(buffer.end(), data, data + len);
       }

       // 尝试读取一个完整的包
       bool readPacket(PacketHeader& header, std::vector<char>& payload) {
           // 1. 检查是否有完整的header
           if (buffer.size() - readPos < sizeof(PacketHeader)) {
               return false;
           }

           // 2. 读取header
           memcpy(&header, buffer.data() + readPos, sizeof(PacketHeader));

           // 3. 检查是否有完整的包
           if (buffer.size() - readPos < header.length) {
               return false;
           }

           // 4. 读取payload
           size_t payloadSize = header.length - sizeof(PacketHeader);
           payload.resize(payloadSize);
           memcpy(payload.data(), buffer.data() + readPos + sizeof(PacketHeader), payloadSize);

           // 5. 移动读取位置
           readPos += header.length;

           // 6. 整理缓冲区（可选，避免无限增长）
           if (readPos > 8192) {  // 超过8KB才整理
               buffer.erase(buffer.begin(), buffer.begin() + readPos);
               readPos = 0;
           }

           return true;
       }
   };

   // 使用示例
   void handleClientWithPacket(int clientFd) {
       PacketBuffer packetBuffer;
       char recvBuffer[4096];

       while (true) {
           int n = recv(clientFd, recvBuffer, sizeof(recvBuffer), 0);
           if (n <= 0) break;

           // 添加到包缓冲区
           packetBuffer.append(recvBuffer, n);

           // 处理所有完整的包
           PacketHeader header;
           std::vector<char> payload;

           while (packetBuffer.readPacket(header, payload)) {
               std::cout << "Received packet, type=" << header.type
                        << ", seq=" << header.sequence
                        << ", payload_size=" << payload.size() << std::endl;

               // 处理包...
               processPacket(header, payload);
           }
       }
   }
   ```

3. **UDP服务器**：
   ```cpp
   class UdpServer {
   private:
       int sockFd;
       int port;

   public:
       UdpServer(int p) : sockFd(-1), port(p) {}

       ~UdpServer() {
           if (sockFd >= 0) {
               close(sockFd);
           }
       }

       bool start() {
           // 1. 创建UDP socket
           sockFd = socket(AF_INET, SOCK_DGRAM, 0);
           if (sockFd < 0) {
               return false;
           }

           // 2. 绑定地址
           sockaddr_in addr{};
           addr.sin_family = AF_INET;
           addr.sin_addr.s_addr = INADDR_ANY;
           addr.sin_port = htons(port);

           if (bind(sockFd, (sockaddr*)&addr, sizeof(addr)) < 0) {
               return false;
           }

           std::cout << "UDP Server started on port " << port << std::endl;
           return true;
       }

       void run() {
           char buffer[1024];
           sockaddr_in clientAddr{};
           socklen_t clientLen = sizeof(clientAddr);

           while (true) {
               // UDP不需要accept，直接recvfrom
               int n = recvfrom(sockFd, buffer, sizeof(buffer) - 1, 0,
                               (sockaddr*)&clientAddr, &clientLen);

               if (n < 0) {
                   continue;
               }

               buffer[n] = '\0';

               char ip[INET_ADDRSTRLEN];
               inet_ntop(AF_INET, &clientAddr.sin_addr, ip, sizeof(ip));

               std::cout << "UDP from " << ip << ":" << ntohs(clientAddr.sin_port)
                        << " - " << buffer << std::endl;

               // 回复
               sendto(sockFd, buffer, n, 0,
                     (sockaddr*)&clientAddr, clientLen);
           }
       }
   };

   /*
   TCP vs UDP 游戏应用场景：

   TCP：
   - 登录认证
   - 聊天消息
   - 物品交易
   - 任务系统
   - 优点：可靠、有序
   - 缺点：延迟相对较高

   UDP：
   - 位置同步
   - 实时战斗
   - 语音通信
   - 优点：低延迟
   - 缺点：可能丢包、乱序

   实际游戏常用：TCP+UDP混合
   - TCP用于重要数据
   - UDP用于实时数据
   */
   ```

**作业**：
- [ ] 实现一个支持多客户端的Echo服务器（多线程版本）
- [ ] 实现完整的协议编解码（包括心跳包）
- [ ] 测试TCP粘包情况并验证解决方案
- [ ] 实现UDP可靠传输（模拟TCP的确认机制）

---

#### **Day 9：epoll高性能IO**

**学习目标**：
- 理解select/poll/epoll的区别
- 掌握epoll的ET和LT模式
- 实现高性能网络服务器

**核心代码**：
```cpp
// Epoll服务器
class EpollServer {
private:
    int listenFd;
    int epollFd;
    static const int MAX_EVENTS = 1024;

public:
    bool init(int port) {
        // 创建监听socket
        listenFd = socket(AF_INET, SOCK_STREAM, 0);
        setNonBlocking(listenFd);  // 设置非阻塞

        // 绑定
        sockaddr_in addr{};
        addr.sin_family = AF_INET;
        addr.sin_port = htons(port);
        addr.sin_addr.s_addr = INADDR_ANY;
        bind(listenFd, (sockaddr*)&addr, sizeof(addr));
        listen(listenFd, 128);

        // 创建epoll
        epollFd = epoll_create1(0);

        // 添加监听socket到epoll
        epoll_event ev{};
        ev.events = EPOLLIN | EPOLLET;  // ET模式
        ev.data.fd = listenFd;
        epoll_ctl(epollFd, EPOLL_CTL_ADD, listenFd, &ev);

        return true;
    }

    void run() {
        epoll_event events[MAX_EVENTS];

        while (true) {
            int nfds = epoll_wait(epollFd, events, MAX_EVENTS, -1);

            for (int i = 0; i < nfds; ++i) {
                if (events[i].data.fd == listenFd) {
                    // 新连接
                    acceptConnections();
                } else {
                    // 数据到达
                    handleClient(events[i].data.fd);
                }
            }
        }
    }

private:
    void acceptConnections() {
        while (true) {
            sockaddr_in clientAddr{};
            socklen_t len = sizeof(clientAddr);
            int clientFd = accept(listenFd, (sockaddr*)&clientAddr, &len);

            if (clientFd < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    break;  // 没有更多连接了
                }
                continue;
            }

            setNonBlocking(clientFd);

            // 添加到epoll
            epoll_event ev{};
            ev.events = EPOLLIN | EPOLLET;
            ev.data.fd = clientFd;
            epoll_ctl(epollFd, EPOLL_CTL_ADD, clientFd, &ev);
        }
    }

    void setNonBlocking(int fd) {
        int flags = fcntl(fd, F_GETFL, 0);
        fcntl(fd, F_SETFL, flags | O_NONBLOCK);
    }
};
```

**作业**：实现epoll+线程池的高并发服务器

---

### **Day 10-14：网络库与Reactor模式**

**Day 10：Reactor模式**
- 单Reactor单线程
- 单Reactor多线程
- 主从Reactor多线程

**Day 11：muduo网络库学习**
- EventLoop事件循环
- Channel通道抽象
- Poller封装

**Day 12：实现简单网络库**
```cpp
// 简化的网络库框架
class EventLoop {
    int epollFd;
    std::vector<Channel*> activeChannels;

public:
    void loop() {
        while (!quit) {
            epoll_wait(...);
            for (auto channel : activeChannels) {
                channel->handleEvent();
            }
        }
    }
};

class TcpServer {
    EventLoop loop;
    Acceptor acceptor;
    std::map<int, TcpConnectionPtr> connections;

public:
    void start() {
        acceptor.setNewConnectionCallback([this](int fd) {
            auto conn = std::make_shared<TcpConnection>(fd);
            connections[fd] = conn;
            conn->setMessageCallback(messageCallback);
        });
        loop.loop();
    }
};
```

**Day 13：Protobuf协议**
```protobuf
// player.proto
syntax = "proto3";

message PlayerInfo {
    int32 player_id = 1;
    string name = 2;
    int32 level = 3;
    int32 x = 4;
    int32 y = 5;
}

message LoginRequest {
    string username = 1;
    string password = 2;
}

message LoginResponse {
    int32 code = 1;
    string message = 2;
    PlayerInfo player = 3;
}
```

**Day 14：心跳与断线重连**

---

## 阶段3：游戏服务器架构（第6-7周）

### **Day 15-21：核心游戏系统**

**Day 15：游戏服务器整体架构**
```
┌──────────────────────────────────────┐
│          客户端 (Unity/Unreal)        │
└──────────────┬───────────────────────┘
               │ TCP/UDP
┌──────────────┴───────────────────────┐
│          网关服务器 (Gateway)          │
│  - 负载均衡                           │
│  - 协议加密                           │
└──────────────┬───────────────────────┘
               │ 内网
┌──────────────┴───────────────────────┐
│          逻辑服务器集群                │
│  ┌────────┬────────┬────────┐        │
│  │Scene 1 │Scene 2 │Scene 3 │        │
│  │(地图1)  │(地图2)  │(副本)  │        │
│  └────────┴────────┴────────┘        │
└──────────────┬───────────────────────┘
               │
┌──────────────┴───────────────────────┐
│            数据层                     │
│  ┌─────────┬────────┐                │
│  │ MySQL   │ Redis  │                │
│  │(持久化) │(缓存)  │                │
│  └─────────┴────────┘                │
└──────────────────────────────────────┘
```

**Day 16：AOI算法（九宫格/十字链表）**
```cpp
// 九宫格AOI
class GridAOI {
    struct Grid {
        std::unordered_set<int> players;
    };

    std::vector<std::vector<Grid>> grids;
    int gridSize;  // 每个格子大小
    int mapWidth, mapHeight;

public:
    GridAOI(int width, int height, int size)
        : mapWidth(width), mapHeight(height), gridSize(size) {
        int rows = (height + size - 1) / size;
        int cols = (width + size - 1) / size;
        grids.resize(rows, std::vector<Grid>(cols));
    }

    // 进入视野
    std::vector<int> enter(int playerId, int x, int y) {
        int row = y / gridSize;
        int col = x / gridSize;

        grids[row][col].players.insert(playerId);

        // 返回九宫格内的其他玩家
        std::vector<int> visible;
        for (int dr = -1; dr <= 1; ++dr) {
            for (int dc = -1; dc <= 1; ++dc) {
                int r = row + dr, c = col + dc;
                if (r >= 0 && r < grids.size() &&
                    c >= 0 && c < grids[0].size()) {
                    for (int pid : grids[r][c].players) {
                        if (pid != playerId) {
                            visible.push_back(pid);
                        }
                    }
                }
            }
        }
        return visible;
    }

    // 移动
    struct AOIEvent {
        std::vector<int> enter;  // 进入视野
        std::vector<int> leave;  // 离开视野
    };

    AOIEvent move(int playerId, int oldX, int oldY, int newX, int newY);
};
```

**Day 17：帧同步vs状态同步**
- 帧同步：客户端预测+服务器校验
- 状态同步：服务器权威

**Day 18：技能系统**
```cpp
class Skill {
public:
    int id;
    int cooldown;      // 冷却时间
    int cost;          // 消耗（魔法值）
    float range;       // 范围
    SkillEffect effect;

    virtual bool canUse(Player* caster) = 0;
    virtual void execute(Player* caster, Target* target) = 0;
};

class SkillManager {
    std::unordered_map<int, std::unique_ptr<Skill>> skills;

public:
    void registerSkill(std::unique_ptr<Skill> skill) {
        skills[skill->id] = std::move(skill);
    }

    bool useSkill(int skillId, Player* caster, Target* target) {
        auto it = skills.find(skillId);
        if (it == skills.end()) return false;

        auto& skill = it->second;
        if (!skill->canUse(caster)) return false;

        skill->execute(caster, target);
        return true;
    }
};
```

**Day 19：战斗系统**
**Day 20：背包与装备系统**
**Day 21：周末项目 - Mini MMO核心**

---

## 阶段4：数据库（第8周）

### **Day 22-28：MySQL与Redis**

**Day 22：MySQL连接池**
```cpp
class ConnectionPool {
    std::queue<MYSQL*> pool;
    std::mutex mtx;
    std::condition_variable cv;
    int poolSize;

public:
    ConnectionPool(int size) : poolSize(size) {
        for (int i = 0; i < size; ++i) {
            MYSQL* conn = mysql_init(nullptr);
            mysql_real_connect(conn, "localhost", "user", "pass",
                             "gamedb", 3306, nullptr, 0);
            pool.push(conn);
        }
    }

    MYSQL* getConnection() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this] { return !pool.empty(); });
        MYSQL* conn = pool.front();
        pool.pop();
        return conn;
    }

    void returnConnection(MYSQL* conn) {
        std::lock_guard<std::mutex> lock(mtx);
        pool.push(conn);
        cv.notify_one();
    }
};
```

**Day 23：ORM设计**
**Day 24：Redis基础命令**
**Day 25：Redis在游戏中的应用**
- 排行榜（ZSET）
- 在线用户（SET）
- 缓存（STRING+过期时间）
- 消息队列（LIST）

**Day 26：数据库设计**
**Day 27：分库分表**
**Day 28：数据一致性**

---

## 阶段5：分布式系统（第9周）

### **Day 29-35：分布式基础**

**Day 29：服务发现（etcd/Consul）**
**Day 30：RPC框架（gRPC）**
**Day 31：负载均衡**
**Day 32：消息队列（RabbitMQ/Kafka）**
**Day 33：分布式锁**
**Day 34：一致性哈希**
**Day 35：微服务架构**

---

## 阶段6：Linux与工具链（第10周）

### **Day 36-42：Linux系统编程**

**Day 36：Linux进程与线程**
**Day 37：进程间通信（共享内存/消息队列）**
**Day 38：信号处理**
**Day 39：GDB调试**
**Day 40：性能分析（perf/valgrind）**
**Day 41：CMake构建**
**Day 42：Shell脚本自动化**

---

## 阶段7：综合实战（第11-12周）

### **Day 43-56：完整游戏服务器项目**

**Week 11（Day 43-49）：实现核心功能**
- Day 43：项目架构设计
- Day 44：网络层实现
- Day 45：登录+认证
- Day 46：场景服务器
- Day 47：战斗系统
- Day 48：数据持久化
- Day 49：周测试与优化

**Week 12（Day 50-56）：高级功能+面试准备**
- Day 50：聊天系统
- Day 51：好友系统
- Day 52：公会系统
- Day 53：压力测试
- Day 54：性能优化
- Day 55：Docker部署
- Day 56：面试准备总结

---

## 阶段8：面试冲刺（第12周后期）

### **Day 57-70：补充技能+面试题**

**Day 57-63：Golang基础**（加分项）
```go
// Golang游戏服务器示例
package main

import (
    "net"
    "fmt"
)

type GameServer struct {
    listener net.Listener
}

func (s *GameServer) Start(port int) error {
    ln, err := net.Listen("tcp", fmt.Sprintf(":%d", port))
    if err != nil {
        return err
    }
    s.listener = ln

    for {
        conn, err := ln.Accept()
        if err != nil {
            continue
        }
        go s.handleConnection(conn)
    }
}

func (s *GameServer) handleConnection(conn net.Conn) {
    defer conn.Close()
    // 处理逻辑...
}
```

**Day 64-70：Lua脚本集成**（热更新）
```cpp
// C++中嵌入Lua
extern "C" {
#include "lua.h"
#include "lualib.h"
#include "lauxlib.h"
}

class LuaEngine {
    lua_State* L;

public:
    LuaEngine() {
        L = luaL_newstate();
        luaL_openlibs(L);
    }

    ~LuaEngine() {
        lua_close(L);
    }

    void executeScript(const std::string& script) {
        luaL_dostring(L, script.c_str());
    }

    int getGlobalInt(const std::string& name) {
        lua_getglobal(L, name.c_str());
        int value = lua_tointeger(L, -1);
        lua_pop(L, 1);
        return value;
    }
};
```

---

## 阶段9：最后冲刺（第12-13周）

### **Day 71-84：面试准备+实战演练**

**高频面试题汇总**：

### **1. C++基础**
```
Q: 解释虚函数的实现原理
A: 虚函数通过虚函数表（vtable）实现。每个包含虚函数的类有一个vtable，
   存储虚函数指针。对象的前4/8字节存储vptr指向vtable。
   调用虚函数时通过vptr查表找到实际函数地址。

Q: 智能指针的实现原理
A: shared_ptr使用引用计数，内部有控制块存储引用计数和deleter。
   每次拷贝增加计数，析构时减少计数，归零时delete对象。
   weak_ptr不增加引用计数，用于打破循环引用。

Q: move语义的好处
A: 避免深拷贝，提高性能。通过转移资源所有权而非复制资源。
   特别适用于容器、大对象的传递。
```

### **2. 网络编程**
```
Q: epoll的ET和LT模式区别
A: LT(水平触发)：只要有数据就通知，可能重复通知
   ET(边缘触发)：只在状态变化时通知一次，需要一次读完所有数据
   ET效率更高但编程复杂，需要循环read直到EAGAIN

Q: TCP粘包怎么处理
A: 1.固定长度
   2.特殊分隔符
   3.长度前缀（最常用）：header记录包长度

Q: 如何实现百万并发
A: 1.epoll ET模式
   2.多线程/线程池
   3.非阻塞IO
   4.零拷贝（sendfile）
   5.协议优化（减少包大小）
```

### **3. 游戏服务器**
```
Q: 如何设计一个MMO的场景服务器
A: 1.采用主从Reactor模式
   2.使用AOI算法优化视野更新
   3.分场景负载均衡
   4.状态同步+客户端预测
   5.Redis缓存玩家数据
   6.MySQL持久化

Q: 如何防止外挂
A: 1.服务器权威：关键计算在服务端
   2.数据加密：协议加密
   3.行为检测：异常速度/位置
   4.限流：防刷
   5.代码混淆

Q: 帧同步和状态同步的选择
A: 帧同步：
   - 适合RTS、MOBA、格斗游戏
   - 客户端计算，服务器只转发输入
   - 延迟敏感，需要确定性

   状态同步：
   - 适合MMO、FPS
   - 服务器计算，客户端只展示
   - 可防外挂，但网络开销大
```

### **4. 数据库**
```
Q: Redis为什么快
A: 1.内存操作
   2.单线程避免锁
   3.IO多路复用
   4.高效的数据结构

Q: MySQL索引优化
A: 1.最左前缀原则
   2.避免全表扫描
   3.覆盖索引减少回表
   4.避免索引失效（函数、类型转换）

Q: 数据库连接池的作用
A: 1.复用连接，避免频繁创建销毁
   2.限制并发连接数
   3.提高性能
```

### **5. 算法题**
```cpp
// 常考算法
1. LRU缓存实现（哈希表+双向链表）
2. 线程安全的单例模式
3. 生产者-消费者模型
4. 定时器实现（小根堆/时间轮）
5. 对象池实现
```

---

## 学习进度追踪表

| 周次 | 内容 | 状态 | 完成日期 |
|------|------|------|----------|
| Week 1 | C++11/14/17 + STL | ⬜ | |
| Week 2 | 多线程 + 设计模式 | ⬜ | |
| Week 3 | 智能指针 + 项目 | ⬜ | |
| Week 4 | Socket + epoll | ⬜ | |
| Week 5 | 网络库 + Reactor | ⬜ | |
| Week 6 | 游戏服务器架构 | ⬜ | |
| Week 7 | AOI + 战斗系统 | ⬜ | |
| Week 8 | MySQL + Redis | ⬜ | |
| Week 9 | 分布式 + 微服务 | ⬜ | |
| Week 10 | Linux + 工具链 | ⬜ | |
| Week 11 | 综合项目实现 | ⬜ | |
| Week 12 | 测试 + 面试准备 | ⬜ | |

---

## 毕业项目要求

**项目名称**：SimpleMMO - 简化的MMO游戏服务器

**功能需求**：
1. ✅ 用户注册/登录（MySQL）
2. ✅ 角色创建/选择
3. ✅ 场景服务器（支持1000+在线）
4. ✅ 移动同步（AOI优化）
5. ✅ 战斗系统（技能+伤害计算）
6. ✅ 聊天系统（世界/私聊）
7. ✅ 好友系统
8. ✅ 排行榜（Redis ZSET）
9. ✅ 背包系统
10. ✅ 数据持久化

**技术栈**：
- C++17
- epoll网络库
- Protobuf协议
- MySQL + Redis
- CMake构建
- Docker部署

**性能指标**：
- 单服支持1000+在线
- 平均延迟<50ms
- TPS>5000

**提交内容**：
1. 完整源码（Github）
2. 设计文档
3. 压测报告
4. 演示视频

---

## 面试通过率提升策略

### **简历优化**：
```
【项目经历】
项目名称：SimpleMMO游戏服务器
时间：2024.XX - 2024.XX
技术栈：C++17、epoll、Protobuf、MySQL、Redis
职责：
1. 设计并实现了高性能网络框架，采用epoll+线程池，单服支持1000+并发
2. 实现了基于九宫格的AOI算法，优化视野更新性能50%
3. 设计了灵活的战斗系统，支持技能、Buff、伤害计算
4. 使用Redis缓存热数据，MySQL持久化，查询性能提升80%
5. 实现了服务器监控和日志系统，快速定位问题

成果：
- 压测TPS达到5000+
- 平均延迟<50ms
- 代码6000+行，Github star 100+
```

### **面试技巧**：
1. **准备3个深度项目故事**（STAR法则）
2. **手写代码能力**（LeetCode中等难度至少刷100题）
3. **系统设计能力**（能在白板上画出完整架构）
4. **问题解决思路**（不会的也要说思路）
5. **谦虚好学态度**

### **模拟面试题**：
```
1. 自我介绍（1分钟）
2. 项目讲解（5分钟）
3. 技术深挖（20分钟）
   - C++虚函数原理
   - 智能指针实现
   - epoll原理
   - TCP粘包
   - 游戏服务器架构
4. 算法题（15分钟）
   - LRU缓存实现
5. 系统设计（15分钟）
   - 设计一个聊天室
6. 反向提问（5分钟）
```

---

## 最终检验清单

### **核心技能自测**：
- [ ] 熟练使用C++11/14/17新特性
- [ ] 掌握STL所有常用容器和算法
- [ ] 能独立实现线程池
- [ ] 理解智能指针原理并能手写
- [ ] 熟悉常用设计模式（至少5个）
- [ ] 能用epoll实现高并发服务器
- [ ] 理解Reactor/Proactor模式
- [ ] 掌握Protobuf使用
- [ ] 能设计游戏服务器架构
- [ ] 熟练使用MySQL和Redis
- [ ] 了解分布式系统基础
- [ ] 熟悉Linux系统编程
- [ ] 能使用GDB调试和性能分析工具
- [ ] 有完整的项目经验

### **面试准备checklist**：
- [ ] 简历优化完成
- [ ] 项目代码上传Github
- [ ] 至少完成100道LeetCode
- [ ] 掌握30+高频面试题
- [ ] 准备好3个项目故事
- [ ] 模拟面试3次以上
- [ ] 准备好5个反向提问

---

## 薪资谈判技巧

**成都C++游戏服务器薪资范围**：
```
初级（1-2年）：12-18K
中级（3-5年）：18-30K  ← 你的目标
高级（5-8年）：30-50K
专家（8年+）：50K+
```

**谈判策略**：
1. 了解市场行情
2. 突出项目亮点
3. 不要第一轮就说期望薪资
4. 给出合理区间（如25-30K）
5. 强调成长空间

**加分项**：
- Github活跃（star多的项目）
- 技术博客
- 开源贡献
- 竞赛经历
- 相关证书

---

## 持续学习资源

### **书籍推荐**：
1. 《C++ Primer》（第5版）
2. 《Effective C++》
3. 《STL源码剖析》
4. 《Unix网络编程（卷1）》
5. 《游戏服务器编程》

### **视频课程**：
1. B站：黑马程序员C++
2. 极客时间：《Linux性能优化实战》
3. 腾讯课堂：游戏服务器开发

### **开源项目学习**：
1. muduo网络库
2. skynet游戏服务器框架
3. KBEngine游戏服务器引擎

### **社区**：
1. C++中国：https://www.cplusplus.com/
2. GameDev：https://www.gamedev.net/
3. SegmentFault

---

## 结语

**恭喜你完成了这份完整的学习计划！**

通过12周的系统学习，你将：
- ✅ 精通C++现代特性
- ✅ 掌握高性能网络编程
- ✅ 理解游戏服务器架构
- ✅ 具备完整项目经验
- ✅ 达到面试95%通过率

**记住**：
> "编程能力 = 理论知识 × 实践经验²"

**现在就开始Day 1的学习，坚持12周，你一定能拿到心仪的offer！**

**加油！未来的游戏服务器工程师！** 🚀🎮💻

---

*文档版本*：v1.0 Complete
*总字数*：30,000+
*代码示例*：80+
*最后更新*：2026-01-29
*作者*：Claude AI

**开始学习** → Day 1 → C++11新特性 ✨