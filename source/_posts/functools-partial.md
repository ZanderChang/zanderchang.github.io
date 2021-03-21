---
title: functools.partial
mathjax: false
date: 2020-09-21 22:02:51
categories:
- Python
tags:
- Python
---

`functools.partial`类似装饰器，可以扩展函数的功能，将某些参数设定为固定值。

<!-- more -->

```py
类func = functools.partial(func, *args, **keywords)

# func: 需要被扩展的函数，返回的函数其实是一个类 func 的函数
# *args: 需要被固定的位置参数
# **kwargs: 需要被固定的关键字参数
# 如果在原来的函数 func 中关键字不存在，将会扩展，如果存在，则会覆盖
```

Example from [Montage](https://github.com/WSP-LAB/Montage)

```py
from multiprocessing import Pool
pool = Pool(conf.num_proc, init_worker) # 进程池

def exec_func(js_path, conf):
    pass

pool_map(pool, exec_func, js_list, conf=conf) # exec_func依次调用js_list中的每个js，每次conf=conf

def pool_map(pool, func, list, **args):
  try:
    func = partial(func, **args) # 固定关键字参数
    return pool.map(func, list) # 进程池依次调用函数
  except KeyboardInterrupt:
    pass
```

- [彻底明白 Python partial()](https://zhuanlan.zhihu.com/p/47124891)