---
title: 从scala-repl到spark-shell
tags:
- spark
categories:
- spark, scala, spark-repl, scala-repl, spark-shell
---
# 前言
最近在思考，如何将spark-shell的交互方式变为spark-web。即用户在页面输入scala代码，spark实时运行，并将结果展示在web页面上。这样可以让有编程能力的数据处理员更方便的使用spark。为了实现这个目的就需要了解spark-shell的实现原理，想办法控制其数据入输出流。  
本文主要通过源码介绍了spark-shell是如何借助scala-repl实现的。

# spark-shell
spark-shell
