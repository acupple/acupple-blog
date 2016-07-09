---
title: Go基于Sublime的开发环境搭建
date: 2016-06-03 11:29:05
tags: Go
---
作为一个Go语言新手，趁手的开发工具是必须的 :) 本文简单记录一下自己搭建Go开发环境的过程
## 安装sublime text2/3
## 配置Package Control
## 安装gosublime包
* 通过Package Control安装
* 手动安装
## 安装gocode
用于实现代码自动提示
```go
go get github.com/nsf/gocode
go install github.com/nsf/gocode
```
## 常用快捷键:
1. ctrl(command) + b : 类似于terminal的控制面板
2. ctrl(command) + p:  搜索文件，例如test.go, 前面加上#在当前文件中搜索
