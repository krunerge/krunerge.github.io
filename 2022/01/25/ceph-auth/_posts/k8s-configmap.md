---
title: 'k8s:configmap'
date: 2018-12-16 23:21:00
tags: k8s configmap
---

# 1.configmap创建方式
- 从字符串创建
- 从env文件创建
- 从目录创建
- 从json/yaml文件创建

# 2.configmap使用
- 用作pod的环境变量
- 通过环境变量作为pod命令参数
- 使用volume将configmap作为文件或者目录直接挂载
