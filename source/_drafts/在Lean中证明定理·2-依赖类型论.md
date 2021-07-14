---
title: 在Lean中证明定理·2.依赖类型论
urlname: 在Lean中证明定理·2.依赖类型论
categories:
tags:
---

[上篇](https://shedarshian.github.io/blog/%E6%95%B0%E5%AD%A6/Lean/proving_in_lean1.html)

<!-- toc -->

# 依赖类型论

依赖类型论是一个很强大的语言，我们可以借助它表达复杂的数学断言，书写复杂的硬件或软件规范，以及以一种自然和统一的方式对它们进行推理。Lean是基于一个叫做**Calculus of Constructions**版本的依赖类型论的，其中包括可数代的非累积宇宙和递归类型。阅读完这一章后，你将会对上面这些东西有更好的理解。

## 简单类型论

集合论作为数学的基础，有着简单又有魅力的本体。任何东西都是集合，无论是数字、函数、三角形、随机过程，还是黎曼流形。我们能从很少的几条与集合构建相关的基本公理出发就能建立这么复杂的数学宇宙，这是很不得了的。

但是出于多种目的（包括形式定理证明），有一个更基本的、能帮助我们管理和跟踪许多种不同类型的数学对象的基础设施更好。“类型论”的名字来自于这样一条事实：每一个表达式都有一个对应的**类型**。例如，在某一个上下文中，`x + 0`可能代表一个自然数，而`f`可能代表在自然数上的函数。

下面是在Lean中声明对象类型与检查对象类型的例子。

```lean
/- 声明一些常数 -/

constant m : nat        -- m是一个自然数
constant n : nat
constants b1 b2 : bool  -- 一次声明两个常数

/- 检查它们的类型 -/

#check m            -- 输出: nat
#check n
#check n + 0        -- nat
#check m * (n + 0)  -- nat
#check b1           -- bool
#check b1 && b2     -- “&&”是逻辑与
#check b1 || b2     -- 逻辑或
#check tt           -- 逻辑“真”

-- 自己动手试试吧！
```
[试试！](https://leanprover-community.github.io/lean-web-editor/#code=%2F-%20declare%20some%20constants%20-%2F%0A%0Aconstant%20m%20%3A%20nat%20%20%20%20%20%20%20%20--%20m%20is%20a%20natural%20number%0Aconstant%20n%20%3A%20nat%0Aconstants%20b1%20b2%20%3A%20bool%20%20--%20declare%20two%20constants%20at%20once%0A%0A%2F-%20check%20their%20types%20-%2F%0A%0A%23check%20m%20%20%20%20%20%20%20%20%20%20%20%20--%20output%3A%20nat%0A%23check%20n%0A%23check%20n%20%2B%200%20%20%20%20%20%20%20%20--%20nat%0A%23check%20m%20*%20(n%20%2B%200)%20%20--%20nat%0A%23check%20b1%20%20%20%20%20%20%20%20%20%20%20--%20bool%0A%23check%20b1%20%26%26%20b2%20%20%20%20%20--%20%22%26%26%22%20is%20boolean%20and%0A%23check%20b1%20%7C%7C%20b2%20%20%20%20%20--%20boolean%20or%0A%23check%20tt%20%20%20%20%20%20%20%20%20%20%20--%20boolean%20%22true%22%0A%0A--%20Try%20some%20examples%20of%20your%20own.)

在`/-`与`-/`之间的任何文本都会作为注释被Lean忽略。类似地，两个连字符表示它所在这一行后面的所有文本都是注释。注释块可以嵌套，也就是可以把很大一段代码一起注释掉，就像在很多编程语言中一样。

指令`constant`与`constants`给工作环境下引入了新的常量符号。`#check`指令要求Lean输出其类型；在Lean里，有额外输出的指令通常以井号开头。你应该试着自己声明一些常量，检查一些表达式的类型。以这种方式声明新对象是一个很好的试验系统的方法，但是这并不是需要的：Lean是一个基础的系统，它为我们提供了有力的方法去**定义**所有我们需要的数学对象，而不是只是把它们贴出来。我们会在后续章节介绍这些内容。

简单类型论是很强大的，这意味着我们可以用已有类型来构建新的类型。例如，如果`α`和`β`是两种类型，`α → β`就代表从`α`到`β`的函数类型，而`α × β`则代表笛卡尔积，也就是包含一个`α`类型的元素和一个`β`类型的元素的有序对。

```lean
constants m n : nat

constant f : nat → nat           -- 用“\to”或是“\r”来输入箭头
constant f' : nat -> nat         -- 同样可以使用ASCII记号
constant f'' : ℕ → ℕ             -- nat的另一种书写方式
constant p : nat × nat           -- 用“\times”来输入乘号
constant q : prod nat nat        -- 另一种方法
constant g : nat → nat → nat
constant g' : nat → (nat → nat)  -- 和g的类型相同！
constant h : nat × nat → nat

constant F : (nat → nat) → nat   -- 一个“泛函”

#check f               -- ℕ → ℕ
#check f n             -- ℕ
#check g m n           -- ℕ
#check g m             -- ℕ → ℕ
#check (m, n)          -- ℕ × ℕ
#check p.1             -- ℕ
#check p.2             -- ℕ
#check (m, n).1        -- ℕ
#check (p.1, n)        -- ℕ × ℕ
#check F f             -- ℕ
```
[试试！](https://leanprover-community.github.io/lean-web-editor/#code=constants%20m%20n%20%3A%20nat%0A%0Aconstant%20f%20%3A%20nat%20%E2%86%92%20nat%20%20%20%20%20%20%20%20%20%20%20--%20type%20the%20arrow%20as%20%22%5Cto%22%20or%20%22%5Cr%22%0Aconstant%20f'%20%3A%20nat%20-%3E%20nat%20%20%20%20%20%20%20%20%20--%20alternative%20ASCII%20notation%0Aconstant%20f''%20%3A%20%E2%84%95%20%E2%86%92%20%E2%84%95%20%20%20%20%20%20%20%20%20%20%20%20%20--%20alternative%20notation%20for%20nat%0Aconstant%20p%20%3A%20nat%20%C3%97%20nat%20%20%20%20%20%20%20%20%20%20%20--%20type%20the%20product%20as%20%22%5Ctimes%22%0Aconstant%20q%20%3A%20prod%20nat%20nat%20%20%20%20%20%20%20%20--%20alternative%20notation%0Aconstant%20g%20%3A%20nat%20%E2%86%92%20nat%20%E2%86%92%20nat%0Aconstant%20g'%20%3A%20nat%20%E2%86%92%20(nat%20%E2%86%92%20nat)%20%20--%20has%20the%20same%20type%20as%20g!%0Aconstant%20h%20%3A%20nat%20%C3%97%20nat%20%E2%86%92%20nat%0A%0Aconstant%20F%20%3A%20(nat%20%E2%86%92%20nat)%20%E2%86%92%20nat%20%20%20--%20a%20%22functional%22%0A%0A%23check%20f%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20--%20%E2%84%95%20%E2%86%92%20%E2%84%95%0A%23check%20f%20n%20%20%20%20%20%20%20%20%20%20%20%20%20--%20%E2%84%95%0A%23check%20g%20m%20n%20%20%20%20%20%20%20%20%20%20%20--%20%E2%84%95%0A%23check%20g%20m%20%20%20%20%20%20%20%20%20%20%20%20%20--%20%E2%84%95%20%E2%86%92%20%E2%84%95%0A%23check%20(m%2C%20n)%20%20%20%20%20%20%20%20%20%20--%20%E2%84%95%20%C3%97%20%E2%84%95%0A%23check%20p.1%20%20%20%20%20%20%20%20%20%20%20%20%20--%20%E2%84%95%0A%23check%20p.2%20%20%20%20%20%20%20%20%20%20%20%20%20--%20%E2%84%95%0A%23check%20(m%2C%20n).1%20%20%20%20%20%20%20%20--%20%E2%84%95%0A%23check%20(p.1%2C%20n)%20%20%20%20%20%20%20%20--%20%E2%84%95%20%C3%97%20%E2%84%95%0A%23check%20F%20f%20%20%20%20%20%20%20%20%20%20%20%20%20--%20%E2%84%95)

同样地，你应该自己试试。

