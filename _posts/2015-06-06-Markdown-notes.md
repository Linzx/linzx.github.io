---
layout: post
title:  "Markdown个人笔记"
date:   2015-06-06 15:14:54
catalog:  true
tags:
    - else

---

## 基本语法
语法方面的文章有很多，这里就不详细说明


- <http://www.appinn.com/markdown/>
- <https://help.github.com/articles/markdown-basics>
- <https://help.github.com/articles/github-flavored-markdown>
- <http://wowubuntu.com/markdown>


## Markdown工具

- [StackEdit](https://stackedit.io) 非常适合github用户，提供直接publish到github的功能;
尤其是windows再也不用担心中文乱码的问题.


## 遇到的问题
-  基于Jekyll的github个人博客，默认是采用kramdown来解析markdown语法，但是无法识别 ```标记符号，围起来的代码块，建议采用4个空格的方式。

- 在所有标记符号之后，要记得**加空格**，否则在markdown的编辑器显示可能正常，当发布到blog时会出现非预期的格式。

- 一行的标记符号结束后，结果加**回车**，避免出现格式混乱。之前我用用表格的markdown语法，在编辑器格式正常，发布博文到github.io上时，表格不显示，经过一段时间折腾尝试，原来是在上一行的格式增加一个回车就解决了。
