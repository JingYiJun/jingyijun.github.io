---
title: "Hello,world"
date: 2022-07-01T08:53:20+08:00
lastmod: 2023-02-01T14:00:00+08:00
slug: "helloworld"
draft: false
math: true
categories:
    - 随笔
tags:
    - Test
---

Hello world! this is my personal blog based on hugo and stack theme.

tests here.

## codes

### c++
```c++
#include <iostream>

int main() {
    std::cout << "Hello, world" << '\n';
    return 0;
}
```

### golang

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, world")
}
```

### shell

```shell
# replace source in debian
cp /etc/apt/sources.list /etc/apt/sources.list.bak
sed -i 's|ftp.debian.org|mirror.sjtu.edu.cn|g' /etc/apt/sources.list
sed -i 's|security.debian.org|mirror.sjtu.edu.cn/debian-security|g' /etc/apt/sources.list
apt update
apt install apt-transport-https ca-certificates -y
sed -i 's|http|https|g' /etc/apt/sources.list
apt update
apt upgrade -y
```

## Katex测试

$$a^2 + b^2 = c^2 $$