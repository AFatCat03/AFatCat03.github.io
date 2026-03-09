+++
date = '2026-03-09T14:29:29+08:00'
draft = false
title = 'Go'
+++

1. for 初始化里声明的变量, 作用域是整个 for 语句, 大于 for { ... } 循环体这个 block, 但又小于外层函数体作用域
```text
函数作用域
└── for 语句作用域（err := rpc.ErrMaybe）
    └── for 循环体 block
        └── state, version, err := ... 里的这个 err 会 shadow 之前的 err
```
