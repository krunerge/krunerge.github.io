---
title: 每天5分钟go(4)
date: 2018-10-13 16:41:51
tags: go
---

# 代码包命令规则
- 1.同目录下的源码文件的代码包声明语句要一致，如果一个目录下面有一个命名源码文件，那么这个包的所有源码文件都要声明为mian包
- 2. 目录下面源码文件的代码包可以和目录名不同
当包名和目录名不一致时，如何使用：
## 如何导入
导入使用代码所在的目录
## 代码体的使用
代码中使用包名加.引入函数
- 代码示例如下：
项目名test
```
$ ll
total 1
drwxr-xr-x 1 krunerge 1049089  0 十月 13 16:59 lib/
-rw-r--r-- 1 krunerge 1049089 67 十月 13 16:59 main.go

main.go如下：
package main

import (
	"test/lib"
)

func main(){
	lib5.Hello()
}

lib/hello.go如下：
package lib5

import "fmt"

func Hello(){
	fmt.Println("nihao")
}
```
# 代码访问权限规则
- 1.包级私有：小写字母开头
- 2.公开：大写字母开头
- 3.模块级私有：创建internal代码包
