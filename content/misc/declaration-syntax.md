+++
date = '2026-02-07T15:04:42+08:00'
draft = false
title = 'Declaration Syntax'
+++
## [Go's Declaration Syntax](https://go.dev/blog/declaration-syntax)

1. C syntax: C took an unusual and clever approach to declaration syntax. Instead of describing the types with special syntax, one writes an expression involving the item being declared, and states what type that expression will have. **In general, to figure out how to write the type of a new variable, write an expression involving that variable that evaluates to a basic type, then put the basic type on the left and the expression on the right.**
    > Declaration Mimics Use: 在声明变量时写下的样子和将来在代码里使用它时的样子是一模一样的
    > 
    > `int (*fp)(int a, int b);`: Here, fp is a pointer to a function because if you write the expression (*fp)(a, b) you’ll call a function that returns int.
    >
    > Because type and declaration syntax are the same, it can be difficult to parse expressions with types in the middle. This is why, for instance, C casts always parenthesize the type, as in `(int)M_PI`
    
2. Go syntax: Just read them left to right(`name type`), gain clarity at the cost of a separate syntax.
    > if f returns a function: `f func(func(int,int) int, int) func(int, int) int`
    >
    > It still reads clearly, from left to right, and it’s always obvious which name is being declared - the name comes first.
    >
    > The distinction between type and expression syntax makes it easy to write and invoke closures in Go: `sum := func(a, b int) int { return a+b } (3, 4)`
    > 
    > Go’s pointer syntax is tied to the familiar C form, but those ties mean that we cannot break completely from using parentheses to disambiguate types and expressions in the grammar. `(*int)(nil)`
