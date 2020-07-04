---
title:      "Shell脚本中$符开头变量解释"
date:       2019-05-13 20:16:54
author:     "bruce"
toc: true
tags:
    - shell
    - linux
---


shell脚本中经常会看到$开头的特殊变量，那怎么理解它们呢？ 


## 变量

- $$ 

Shell本身的PID（ProcessID） 

- $! 

Shell最后运行的后台Process的PID 

- $? 

最后运行的命令的结束代码（返回值） 

- $- 

使用Set命令设定的Flag一览 

- $* 

所有参数列表。如"$*"用「"」括起来的情况、以"$1 $2 … $n"的形式输出所有参数。 

- $@ 

所有参数列表。如"$@"用「"」括起来的情况、以"$1" "$2" … "$n" 的形式输出所有参数。 

- $# 

添加到Shell的参数个数 

- $0 

Shell本身的文件名 

- $1～$n 

添加到Shell的各参数值。$1是第1参数、$2是第2参数…。 


## 示例

``` shell
 1 #!/bin/bash
 2 #
 3 printf "The complete list is %s\n" "$$"
 4 printf "The complete list is %s\n" "$!"
 5 printf "The complete list is %s\n" "$?"
 6 printf "The complete list is %s\n" "$*"
 7 printf "The complete list is %s\n" "$@"
 8 printf "The complete list is %s\n" "$#"
 9 printf "The complete list is %s\n" "$0"
10 printf "The complete list is %s\n" "$1"
11 printf "The complete list is %s\n" "$2

```

## 结果

```
[Aric@localhost ~]$ bash params.sh 123456 QQ
The complete list is 24249
The complete list is
The complete list is 0
The complete list is 123456 QQ
The complete list is 123456
The complete list is QQ
The complete list is 2
The complete list is params.sh
The complete list is 123456
The complete list is QQ
```


