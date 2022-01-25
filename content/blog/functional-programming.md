---
title: "functional-programming"
date: 2015-09-14 12:02:16
categories: 
tags: 
comments: true
description: 函数式编程的阅读经验，体会
keywords: 函数式编程，functional programming

---


<!--more-->

函数式编程不是面向过程编程

早年学习pascal时，有produce和function两个很接近的概念,已经不记得有哪些区别了，笔记不知道还能不能找得到，去回顾了一下

在pascal中这样定义：
> Functions − these subprograms return a single value.
> Procedures − these subprograms do not return a value directly.

```pascal
procedure max(x, y: integer; var m: integer); 
begin
	if x > y then
		m := x
	else
		m := y;
end; 
```
```pascal
function max(x, y: integer): integer;
var
   m: integer;
begin
   if x > y then
      m := x
   else
      m := y;
   max := m;//将值赋给函数名，相当于return m
end;
```
实参，形参（var 关键字后的变量，只能是变量，而不允许是常数或带有运算的表达式。（因为堆内存，栈内存的原因））
对应着值传递，引用传递
quora上的回答：

> Object oriented solutions are interested in modelling a program as a result of the communication between each of its subsystems. A functional program is interested in modelling its solution as a single mathematical expression, such that you can understand it by using tools like equational reasoning.


 
## refer

[多范式编程语言－以Swift为例](http://www.infoq.com/cn/articles/multi-paradigm-programming-language-swift)

[quora_Is it time for us to dump the OOP paradigm? If yes, what can replace it?](http://www.quora.com/Is-it-time-for-us-to-dump-the-OOP-paradigm-If-yes-what-can-replace-it)

[pascal_produce](http://www.tutorialspoint.com/pascal/pascal_procedures.htm)
