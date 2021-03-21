---
title: LD_AUDIT对抗LD_PRELOAD
mathjax: false
date: 2020-10-25 14:56:26
categories:
- Linux
tags:
- Linux
---

- 拦截LD_PRELOAD加载的.so
- 获取LD_PRELOAD中劫持的函数
- 利用LD_AUDIT劫持函数


<!-- more -->

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <dlfcn.h>
#include <link.h>
#include <dirent.h>

// 拦截LD_PRELOAD加载的.so
// const char *preloaded;

// unsigned int la_version(unsigned int version)
// {
//     preloaded = getenv("LD_PRELOAD");
//     return version;
// }

// char *la_objsearch(const char *name, uintptr_t *cookie, unsigned int flag)
// {
//     if (NULL != preloaded && strcmp(name, preloaded) == 0)
//     {
//         fprintf(stderr, "Disabling the loading of a 'preload' library: %s\n", name);
//         return NULL;
//     }
//     return (char *)name;
// }

// 获取LD_PRELOAD中劫持的函数
uintptr_t *preloaded_cookie = NULL;
const char *preloaded;

unsigned int la_version(unsigned int version)
{
    preloaded = getenv("LD_PRELOAD");
    return version;
}

unsigned int la_objopen(struct link_map *map, Lmid_t lmid, uintptr_t *cookie)
{
    if (NULL != preloaded && strcmp(map->l_name, preloaded) == 0)
    {
        fprintf(stderr, "A 'preload' library is about to load: %s. Following its function binding\n", map->l_name);
        preloaded_cookie = cookie;

        return LA_FLG_BINDTO | LA_FLG_BINDFROM; // 导向la_symbind64处理函数地址
    }
    return 0;
}

uintptr_t la_symbind64(Elf64_Sym *sym, unsigned int ndx, uintptr_t *refcook, uintptr_t *defcook, unsigned int *flags, const char *symname)
{
    if (refcook == preloaded_cookie)
    {
        fprintf(stderr, "Function '%s' is intercepted\n", symname);
    }
    return sym->st_value;
}

// 利用LD_AUDIT劫持函数
// unsigned int la_version(unsigned int version)
// {
//     return version;
// }

// unsigned int la_objopen(struct link_map *map, Lmid_t lmid, uintptr_t *cookie)
// {
//     return LA_FLG_BINDTO | LA_FLG_BINDFROM;
// }

// uintptr_t la_symbind64(Elf64_Sym *sym, unsigned int ndx, uintptr_t *refcook, uintptr_t *defcook, unsigned int *flags, const char *symname)
// {
//     if (strcmp(symname, "readdir") == 0)
//     {
//         fprintf(stderr, "'readdir' is called, intercepting.\n");
//         // readdir is the tampered function declared in processhider.c
//         return readdir; // fix here
//     }
//     return sym->st_value;
// }
```
编译
```sh
gcc -Wall -fPIC -shared -o libaudit.so libaudit.c -ldl
```

[原文地址](https://labs.sentinelone.com/leveraging-ld_audit-to-beat-the-traditional-linux-library-preloading-technique/)