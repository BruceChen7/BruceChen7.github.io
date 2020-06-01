---
title: 怎样用Hexo写文章
layout: post
date: 2014-12-04 21:20:40
tag: tools
---

`hexo new [layout] 'blog title`，其中layout的用三种格式

* post
* page
* draft

分别对应生成的文件在 `source/_posts`,`source/`,`source/_drafts` ,默认的格式是post 格式
举个例子

```
hexo new post 'title'
hexo new page 'page_title'
hexo new draft 'drafts_title'
```

## 头部格式(front-matter)
头部格式是指 在`_ _ _`前面的部分说明，比如说

```
title:  Hello World
date:  2013/7/13 20:46:25
---
```

可以给post 添加的属性有 `title` ,`tags`,`categories`,`date`,`comments`,`permalink`等，要注意的是这些属性后面的空格.  例子：

```
title:  hello world
tags:  example
categories: example
date： 2014/5/3
comments: true/false
---
```

tags 和 categories 多个可以这么写,

```
tags:
-  hello world
-  Example
categories:
-  SaySomething
-  Example

```

## 摘要

在博客的前后之间 加入 `<!--more-->`，注意前后的空格，例如

> Lorem ipsum dolor sit amet, consectetur adipiscing elit. Curabitur libero est, vulputate nec nibh sit amet, luctus placerat diam. Aliquam sit amet est arcu.
Aenean sit amet mi tristique, luctus diam sit amet, pharetra justo. Quisque ac faucibus tell



