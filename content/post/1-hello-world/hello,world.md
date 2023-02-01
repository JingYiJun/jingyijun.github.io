---
title: "Hello,world 你好世界"
description: 
slug: "helloworld"
date: 2022-07-01T08:53:20+08:00
lastmod: 2023-02-01T14:00:00+08:00
image: 
math: true
hidden: false
comments: true
draft: false
categories:
    - 随笔
---

你好世界，这是我的个人博客。

基于hugo，一个快速的静态网站生成器。

以下为测试：

## 代码测试

### c++ 代码测试
```c++
#include <iostream>

int main() {
    std::cout << "Hello, world" << '\n';
    return 0;
}
```

### golang 代码测试

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, world")
}
```

### shell 测试

```shell
# debian 换源脚本
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

换行文本测试：

$$a^2 + b^2 = c^2 $$

行内文本测试：$a^2 + b^2 = c^2$

矩阵测试（由于文本解析，换行需要四个反斜杠）：

```tex
$$ 
\begin{pmatrix}
1 & 0 & 0 \\\\
0 & 1 & 0 \\\\
0 & 0 & 1 \\\\
\end{pmatrix} 
$$
```

$$ 
\begin{pmatrix}
1 & 0 & 0 \\\\
0 & 1 & 0 \\\\
0 & 0 & 1 \\\\
\end{pmatrix} 
$$