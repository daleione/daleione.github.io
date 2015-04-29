---
layout: page
title: 开始做毕业设计
---


### 题目
做一个语法类似于lisp的解释器
    (f 3 2)                                 -- result: 1
    (f :x 3 :y 2)                           -- result: 1
    (f :y 3 :x 2)                           -- result: -1

    (define fact
    (fun ([x Int] [-> Int])
    (if (= x 0) 1 (* x (fact (- x 1))))))
    
   (fact 5) 
