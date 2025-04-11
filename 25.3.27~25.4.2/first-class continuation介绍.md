# First-class Continuation介绍

时间：2025.4.2

## deepseek的解释

**一等延续（first-class continuation）** 是编程语言理论中的核心概念，指程序执行过程中的**完整控制状态**可以被捕获、存储、传递和恢复的机制。它赋予开发者对程序执行流程的底层控制权，具有以下关键特性：

---

### 核心定义
1. **程序状态的快照**  
   捕捉当前执行点的所有上下文信息，包括：
   - 调用栈（Call stack）
   - 变量绑定（Variable bindings）
   - 程序计数器（Program counter）

2. **一等公民（First-class）**  
   延续可像普通数据一样被操作：
   - 作为参数传递
   - 存储在变量中
   - 从函数返回
   - 动态生成和修改

---

### 技术实现范例
```scheme
; Scheme 语言通过 call/cc 实现
(define (example)
  (call/cc 
    (lambda (k)  ; k 是当前延续的具象化表示
      (display "A")
      (k #t)     ; 跳转回延续保存点
      (display "B")))) 
; 输出仅为 "A"，B 永远不会执行
```

---

### 与协程的关系
- **超集能力**  
  协程（Coroutine）本质上可通过延续实现：每次 `yield` 可视为保存当前延续，`resume` 则是恢复特定延续。
- **更强大的控制流**  
  延续允许任意跳转（如非局部退出、回溯等），而协程仅支持对称/非对称的协作式切换。

---

### 现实应用场景
| 场景              | 典型案例                  |
|-------------------|--------------------------|
| 异常处理          | 非局部退出（Non-local exit）|
| 回溯算法          | 搜索树中的状态回溯        |
| Web框架中间件     | Connect/Express.js 的 next() |
| 生成器模拟        | 惰性求值实现              |

---

### 代码可读性挑战
```javascript
// 函数式 JavaScript 中的延续传递风格（CPS）
function factorial(n, cont) {
  if (n === 0) cont(1);
  else factorial(n - 1, (res) => cont(n * res));
}
// 调用链的可读性显著下降
factorial(5, (result) => console.log(result)); // 输出 120
```

---

### 历史影响
- **学术研究转向**：20世纪90年代，由于延续能更统一地表达各类控制流（包括协程），导致协程作为独立课题的研究减少
- **语言设计取舍**：现代语言更倾向于有限制的延续（如C#的`async/await`），避免完全灵活的一等延续带来的认知负荷

**关键洞见**：延续提供了类似"时间旅行"的能力，但其强大特性与代码可维护性之间存在显著权衡。

## 查找资料

[Continuation](https://www.cnblogs.com/my_life/articles/7364237.html)

[Continuations Made Simple and Illustrated
Denys Duchier](https://www.ps.uni-saarland.de/~duchier/python/continuations.html)

call/cc在CPS下的定义：`call/cc(f, c) = f(c, c)`（即：将本语句的continuation传给f）

## 限定续体（Delimited Continuation）

[Delimited Continuations](https://ps-tuebingen-courses.github.io/pl1-lecture-notes/19-shift-reset/shift-reset.html)

Deepseek的解释：

**Delimited Continuation（限定延续）** 是编程语言理论中的一种控制流机制，允许捕获程序执行的某一部分（而非整个程序），并在之后恢复。它与普通 Continuation 的关键区别在于其作用范围被显式限定，从而提供更精细的控制。

---

### **核心概念**
1. **分隔符（Delimiter）**：
   - 使用如 `reset` 的关键字设定一个边界，限定 Continuation 的作用范围。
   - 例如：`reset { ... }` 内的代码是 Continuation 的捕获范围。

2. **捕获操作符**：
   - 如 `shift` 在 `reset` 块内捕获到分隔符的 Continuation。
   - `shift` 将捕获的 Continuation 包装成函数，供后续调用。

---

### **与普通 Continuation 的区别**
- **普通 Continuation**：捕获从当前点到程序结束的所有剩余计算（无限延续）。
- **限定 Continuation**：仅捕获到最近的分隔符（如 `reset`），作用域更小、更可控。

---

### **示例（伪代码）**
```scheme
(reset 
  (let ((result (shift k (k 10)))) 
    (+ result 5)))
```
- `reset` 设定范围，`shift` 捕获到 `reset` 的 Continuation `k`。
- `(k 10)` 会替换 `shift` 的位置，代入 `10`，最终计算 `10 + 5 = 15`。

---

### **主要应用场景**
1. **生成器/协程**：
   - 暂停执行并保存当前状态（如 `yield`），之后恢复。
   
2. **回溯算法**：
   - 在搜索失败时回退到之前的决策点（如 Prolog 的深度优先搜索）。

3. **异步编程**：
   - 管理非阻塞操作，将回调转换为线性代码（类似 async/await）。

4. **异常处理**：
   - 捕获到特定处理点的 Continuation，而非整个程序。

---

### **实现与语言支持**
- **Racket**：通过 `shift/reset` 或 `prompt/control` 提供原生支持。
- **Scala**：通过库（如 `scala-continuations`）模拟。
- **其他语言**：如 Haskell 的 `MonadCont` 提供类似抽象。

---

### **优势**
- **模块化**：避免全局状态污染，代码更易维护。
- **性能**：仅保存必要的上下文，资源消耗更低。
- **灵活性**：支持非局部控制流（如协程切换、自定义语法糖）。

---

### **总结**
Delimited Continuation 通过显式分隔符，将程序执行流切割为可控片段，使开发者能更安全、高效地管理复杂控制逻辑。它是实现高级抽象（如协程、生成器）的理论基础，广泛应用于函数式编程和语言设计中。

限定续体因为只捕获了执行流的一部分，因此是可返回的。而一般的续体不可返回。