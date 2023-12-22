---
title: 逻辑基础，真正理解逻辑式编程的第一步
description: 说的逻辑，我们都会想到这样的话：“这个人说话没有逻辑。”这通常暗示两种含义，一是这个人说话没有清晰的结构，咕咕哝哝不知说什么，没有一个清晰的类似于“因为…而且…所以…”的结构；另一种是这个人说的话传递了一个错误的信息，比如“美国的首都是纽约”。“逻辑”这个词的这一日常用法其实刚好反映了逻辑学的一些基本的研究内容。
categories:
 - programming
tags:
---

说的逻辑，我们都会想到这样的话：“这个人说话没有逻辑。”这通常暗示两种含义，一是这个人说话没有清晰的结构，咕咕哝哝不知说什么，没有一个清晰的类似于“因为…而且…所以…”的结构；另一种是这个人说的话传递了一个错误的信息，比如“美国的首都是纽约”。“逻辑”这个词的这一日常用法其实刚好反映了逻辑学的一些基本的研究内容。

# 从古典逻辑到新逻辑
在公元前四世纪，亚里士多德最早将逻辑建立为一门科学。我们最熟悉的当属他的三段论，即

- 所有人都是必死的。
- 苏格拉底是人。
- 苏格拉底是必死的。

是一种强大的推理工具。不过这个阶段的逻辑学属于古典逻辑，研究内容总的来说还比较随意，没有形成体系。随后便是相当长时间的沉寂。直到十七世纪，德国数学家、哲学家莱布尼茨进行了一些逻辑学方面的研究。然后莱氏的研究并没有给逻辑学带来太大的影响，甚至其逻辑学的著作也被掩埋。逻辑学真正迎来脱胎换骨发展的时间大致始于十九世纪中叶，其标志是布尔发表了其逻辑系统。这次发展使逻辑学彻底转变成一种具有与数学类似性质的学科。发展后的新逻辑学就是所说的“数理逻辑”、“演绎逻辑”或者“符号逻辑”。新的逻辑学相比于古典逻辑，不仅有坚实的基础和完善的方法，而且还建立了更为丰富的概念和定理。

说到新逻辑学内容丰富，在《逻辑与演绎科学方法论导论》一书介绍了命题逻辑、同一理论、类的理论、关系的理论以及演绎方法。在这些内容里，“命题逻辑”是最为大众熟知的一个领域，它也和Prolog关系最为密切。

# 命题逻辑

逻辑学的内容包罗万象，错综复杂。不过这不妨碍“命题逻辑”走向大众视野，因为它研究的对象就是人类如何“表达”。有了表达方法，人类才能记录知识，阐述真理。比如，

 - 张三和李四一起走来了。
 - x能被2整除，而且x能被3整除，那么x能被6整除。
 - 一个四边形即是矩形，又是菱形，那么它是正方形。

上面三个语句来自于不同的领域（学科），像“张三”、“整除”、“正方形”只在特定的领域里才有意义。不过，像“和”、“而且”、“即…又…”却能超越不同学科，表达了类似的概念。这样词汇还有很多，比如“不”、“或”、“如果…那么”等等。研究这些词语的含义就是“命题逻辑”的任务。命题逻辑把这些词称为逻辑运算符，它们包括：

 - NOT ¬
 - AND ∧
 - OR ∨
 - IMPLY (IF…THEN…) →
 - ……

基于上面的逻辑运算符，命题逻辑将它们与“原子命题”组成命题，并且包含了一套判断命题是否为定理的规则。不过，命题逻辑只是最简单的逻辑演算，它能表达和处理的语句非常有限，比如，

 - 苏格拉底是哲学家。
 - 拍拉图是哲学家。

这两句话在命题逻辑中只是两个独立的语句，可以分别用符号表示为P、Q。因此，想要揭示这些语句之间的关系，还需要对命题逻辑进行扩展。这就谈到了“一阶逻辑”。

# 一阶逻辑

对于上面的例子，一阶逻辑里可以表示为

```python
  Phil(a)
```
  
当a代表苏格拉底时，Phil(a)为真，当a表示柏拉图时，Phil(a)也为真。因此，可以看到一阶逻辑比命题逻辑更富于表达力。事实上，一阶逻辑保留了命题逻辑的所有内容，又加入项、公式、谓词、函数、量词等概念。

一阶逻辑最终表现为一系列的“符号”，这其中包含了逻辑符号与非逻辑符号两种。逻辑符号包括像逻辑运算符（¬、∧、∨、→）以及量词∃（存在量化）和∀（全称量化）。而非逻辑符号则更丰富一些，它包含两类，谓词符号（关系符号）与函数符号。

- 函数符号通常用小写字母f、g、h等表示，形如f、f(x)、f(x, y)。
    + 没有参数的函数代表常量，
    + f(x)可以表示“x的绝对值”、“x的父亲”，
    + f(x, y)可以表示“x与y的和”等等。
- 谓词符号通常用大写字母P、Q、R等表示，形如P，P(x)、P(x, y)。
    + 没有参数的形式表示命题变量，它可以代表任何命题，
    + P(x)可以表示“x是哲学家”，
    + P(x, y)可以表示“x大于y”或者“x是y的父亲”等等。

前面的Phil(a)就是一个包含一个参数的谓词。

一阶逻辑包含两种合法的表达式，即项和公式，直观上，“项”代表了事物，“公式”代表了谓词。谓词很像一个返回真、假值的函数。一阶逻辑的形成规则定义了项与公式应该长成什么样子，事实上该形成规则是上下文无关的形式文法。具体的文法会随着表述者不同而略有区别，在此只简单概述一下项与公式。

- 项包含两种，变量和函数。函数的定义是递归的，它是形如f(t1,t2,t3,…)的表达式，且t1、t2、t3也是一个项。
- 公式则更为丰富，它包括
    + 谓词符号，若P是n元谓词符号，t1、t2、…tn是项，那么P(t1, t2, …, tn)是公式
    + 等式，若t1、t2是项，那么t1＝t2是公式
    + 否定式
    + 二元关联词
    + 量化，若x是变量，t是公式，那么∀xt与∃xt都是公式

难以表达的“IF-THEN-ELSE”是一阶逻辑的一个缺陷。If..Then…Else（或者“ite(c, a, b)”，其中c是一个用公式表达的条件，当它为真时，返回a，为假时，返回b）结构大量地应用在数学公式表达和程序设计语言中，但是这种常用的结构却很难在一阶逻辑中给予表达，因为在这种结构中，c是公式，而一阶逻辑中的项和公式都只接受“项”作为参数。

ite固然有方法表达，比如(c→a) ∨ (¬c→b)，但是当c非常复杂时，这种表达方法也会很低效。

# 命题逻辑、一阶逻辑、高阶谓词

一阶逻辑对命题逻辑进行了扩展，引入了变量、项、量词、函数、谓词。不过一阶逻辑只允许量化变量，而在高级逻辑中，变量类型可以出现在量化中（二阶逻辑）。高阶谓词就是接受其他谓词作为参数的谓词。一般的，阶为 n 的高阶谓词接受一个或多个(n − 1)阶的谓词作为参数，这里的 n > 1。

一阶逻辑是数学基础的重要部分，它是很多公里系统的标准形式系统，比如皮亚诺公里就可以形式化为一阶逻辑。

最后，在《逻辑与演绎科学方法论导论》中没有提及霍恩子句，原因是霍恩子句的出现晚与该书的写作。它是只包含一个肯定文字的子句，这样使得子句在计算机上的表示更加高效。因此霍恩子句在逻辑编程中扮演基本角色，Prolog就是完全构造在霍恩子句之上的编程语言。