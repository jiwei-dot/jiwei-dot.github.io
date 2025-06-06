---
title:      "git bisect用法"
date:       2025-06-07 00:40:00 +0800
categories: [Git]
tag: []
math: true
description: 使用git bisect查找某一笔出错的commit.
comments: false
mermaid: true
---

## 前言

项目上经常会存在这样一个问题，明明基于主线构建出来的上个版本的包功能测试全都正常，但是当前版本却存在BUG。由于参与开发的人数众多，两个版本之间可能存在非常多笔commit，想要找到究竟是哪笔commit引入了问题非常重要。除了基于每一笔commit提交去验证版本之外，还可以利用git bisect功能进行二分查找，这样速度更快且效率更高。

## 示例

我们可以任意写一些代码，然后故意在中间引入一个带有bug的commit。下面的代码就是我随便写的一个示例， 可以参考git log记录，我是在5a21ae5故意引入一个bug将第8行的返回值从true改为false，这样就会导致除数为0引发异常。
```cpp
// test.cpp
#include <iostream>
#include <stdexcept>

bool always_return_true()
{
    // this is a bug commit
    return false;
}

int denominator()
{
    if (always_return_true())
        return 1;
    return 0;
}

int main()
{
    int num = 20;
    try {
        std::cout << num / denominator() << std::endl;
    } catch (const std::exception &e) {
        std::cerr << e.what() << std::endl;
    }
    
    return 0;
}
```

```sh
git log --oneline

e879feb (HEAD -> master) change num from 10 to 20
f6b08de add 'this is a bug commit' comment
5a21ae5 this is a bug commit
cf8acd5 ignore .vscode
6658221 add try catch
3e3ac9f first commit
```

我们在当前版本发现程序出现了问题，于是乎可以往前找一个功能正常的commit，然后使用git bisect二分方法。

```sh
git bisect start HEAD 3e3ac9f
```
HEAD代表当前存在问题的commit，3e3ac9f代表上一次发版功能验证正常的commit，此时git会给我们自动切换到cf8acd5，我们此时可以基于该commit进行版本验证，如果验证正常就运行：
```sh
git bisect good
```
如果验证异常就运行：
```sh
git bisect bad
```
一直重复下去，直到运行到最后git就会输出一句话告诉我们出错的commit:
```sh
5a21ae52ad735a5193cbc7c40af414b02500a87b is the first bad commit
commit 5a21ae52ad735a5193cbc7c40af414b02500a87b
Author: jiwei-dot
Date:   Sat Jun 7 00:31:55 2025 +0800

    this is a bug commit

 test.cpp | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```
最后我们运行下面这个命令退出二分搜索模式：
```sh
git bisect reset
```