title: QXlsx快速使用
abbrlink: d87f7e0c
tags:
  - mess
categories: []
author: upstairs
date: 2024-11-12 11:15:00
---
# QXlsx快速使用

## 0X00 导入项目属性

将include,lib文件夹放入项目include,lib内,（注：这两个文件夹可以自己创建）

在VS项目中找到项目右键点击**属性**

文件夹加入项目属性,lib名称要写到依赖项中,dll文件加到生成exe所在目录.

## 0x01 快速使用

引入头文件:`#include <QtXlsx>`

创建xlsx:
`QXlsx::Document xlsx;`
导出为excel文件