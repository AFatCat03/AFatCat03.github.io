+++
date = '2026-02-10T22:29:21+08:00'
draft = false
title = 'Effective Go'
+++

## [Effective Go](https://golang.google.cn/doc/effective_go)

### Formatting
- The gofmt program (also available as go fmt, which operates at the package level rather than source file level) reads a Go program and emits the source in a standard style of indentation and vertical alignment, retaining and if necessary reformatting comments.

### Commentary
- Go provides C-style /* */ block comments and C++-style // line comments. Line comments are the norm; block comments appear mostly as package comments, but are useful within an expression or to disable large swaths of code.
- Comments that appear before top-level declarations, with no intervening newlines, are considered to document the declaration itself. These “doc comments” are the primary documentation for a given Go package or command. For more about doc comments, see “[Go Doc Comments](https://golang.google.cn/doc/comment)”.
    > 可以用`go doc`查看

### Names
- Names are as important in Go as in any other language. They even have semantic effect: **the visibility of a name outside a package is determined by whether its first character is upper case.**
#### Package names
- By convention, packages are given lower case, single-word names; there should be no need for underscores or mixedCaps.
- Err on the side of brevity, since everyone using your package will be typing that name.
- Another convention is that the package name is the base name of its source directory; the package in src/encoding/base64 is imported as "encoding/base64" but has name base64, not encoding_base64 and not encodingBase64.
- The importer of a package will use the name to refer to its contents, so exported names in the package can use that fact to avoid repetition. Use the package structure to help you choose good names.
    > For instance, the buffered reader type in the bufio package is called Reader, not BufReader, because users see it as bufio.Reader, which is a clear, concise name. Moreover, because imported entities are always addressed with their package name, bufio.Reader does not conflict with io.Reader.
- A helpful doc comment can often be more valuable than an extra long name.
#### Getters
- Go doesn't provide automatic support for getters and setters. There's nothing wrong with providing getters and setters yourself, and it's often appropriate to do so, but it's neither idiomatic nor necessary to put Get into the getter's name. If you have a field called owner (lower case, unexported), the getter method should be called Owner (upper case, exported), not GetOwner. The use of upper-case names for export provides the hook to discriminate the field from the method. A setter function, if needed, will likely be called SetOwner. Both names read well in practice:
```go
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```
#### Interface names
- By convention, one-method interfaces are named by the method name plus an -er suffix or similar modification to construct an agent noun: Reader, Writer, Formatter, CloseNotifier etc.
- There are a number of such names and it's productive to honor them and the function names they capture. Read, Write, Close, Flush, String and so on have canonical signatures and meanings. To avoid confusion, don't give your method one of those names unless it has the same signature and meaning. Conversely, if your type implements a method with the same meaning as a method on a well-known type, give it the same name and signature; call your string-converter method String not ToString.
#### MixedCaps
- **Finally, the convention in Go is to use MixedCaps or mixedCaps rather than underscores to write multiword names.**

### Semicolons
- Like C, Go's formal grammar uses semicolons to terminate statements, but unlike in C, those semicolons do not appear in the source. Instead the lexer uses a simple *rule* to insert semicolons automatically as it scans, so the input text is mostly free of them.
- The rule is this.
    - If the last token before a newline is an identifier (which includes words like int and float64), a basic literal such as a number or string constant, or one of the tokens(`break continue fallthrough return ++ -- ) }`), the lexer always inserts a semicolon after the token. This could be summarized as, **“if the newline comes after a token that could end a statement, insert a semicolon”.**
    - A semicolon can also be omitted immediately before a closing brace, so a statement such as `go func() { for { dst <- <-src } }()` needs no semicolons.
- Idiomatic Go programs have semicolons only in places such as for loop clauses, to separate the initializer, condition, and continuation elements. They are also necessary to separate multiple statements on a line, should you write code that way.
- One consequence of the semicolon insertion rules is that you **cannot** put the opening brace of a control structure (if, for, switch, or select) on the next line. If you do, a semicolon will be inserted before the brace, which could cause unwanted effects.
```go {title="right"}
if i < f() {
    g()
}
```
```go {title="wrong"}
if i < f()  // wrong!
{           // wrong!
    g()
}
```

### Control structures
The control structures of Go are related to those of C but differ in important ways. There is no do or while loop, only a slightly generalized for; switch is more flexible; if and switch accept an optional initialization statement like that of for; break and continue statements take an optional label to identify what to break or continue; and there are new control structures including a type switch and a multiway communications multiplexer, select. The syntax is also slightly different: there are no parentheses and the bodies must always be brace-delimited.
#### If
```go {title="a simple if"}
if x > 0 {
    return y
}
```
```go {title="Since if and switch accept an initialization statement, it's common to see one used to set up a local variable."}
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```
#### Redeclaration and reassignment
- In a := declaration a variable v may appear even if it has already been declared, provided:
    - this declaration is in the same scope as the existing declaration of v (if v is already declared in an outer scope, the declaration will create a new variable §),
    - the corresponding value in the initialization is assignable to v, and
    - there is *at least one* other variable that is created by the declaration.
- § It's worth noting here that in Go the scope of function parameters and return values is the **same** as the function body, even though they appear lexically outside the braces that enclose the body.
#### For
- The Go for loop is similar to—but not the same as—C's. It unifies for and while and there is no do-while. There are three forms, only one of which has semicolons.
    ```go
    // Like a C for
    for init; condition; post { }

    // Like a C while
    for condition { }

    // Like a C for(;;)
    for { }
    ```
- Short declarations make it easy to declare the index variable right in the loop.
    ```go
    sum := 0
    for i := 0; i < 10; i++ {
        sum += i
    }
    ```
- If you're looping over an array, slice, string, or map, or reading from a channel, a range clause can manage the loop.
    ```go
    for key, value := range oldMap {
        newMap[key] = value
    }
    ```
    ```go {title="only need the first item in the range (the key or index)"}
    for key := range m {
        if key.expired() {
            delete(m, key)
        }
    }
    ```
    ```go {title="only need the second item in the range (the value)"}
    sum := 0
    for _, value := range array {
        sum += value
    }
    ```
- For strings, the range does more work for you, breaking out individual Unicode code points by parsing the UTF-8. Erroneous encodings consume one byte and produce the replacement rune U+FFFD. (The name (with associated builtin type) rune is Go terminology for a single Unicode code point. See [the language specification](https://golang.google.cn/ref/spec#Rune_literals) for details.)
    ```go
    for pos, char := range "日本\x80語" { // \x80 is an illegal UTF-8 encoding
        fmt.Printf("character %#U starts at byte position %d\n", char, pos)
    }
    ```
    ```console {title="result"}
    character U+65E5 '日' starts at byte position 0
    character U+672C '本' starts at byte position 3
    character U+FFFD '�' starts at byte position 6
    character U+8A9E '語' starts at byte position 7
    ```
- **Finally, Go has no comma operator and ++ and -- are statements not expressions.** Thus if you want to run multiple variables in a for you should use parallel assignment (although that precludes ++ and --).
    ```go
    // Reverse a
    for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
        a[i], a[j] = a[j], a[i]
    }
    ```
#### Switch
- Go's switch is more general than C's. The expressions need not be constants or even integers, the cases are evaluated top to bottom until a match is found, and if the switch has no expression it switches on true.
    ```go {title="It's therefore possible—and idiomatic—to write an if-else-if-else chain as a switch."}
    func unhex(c byte) byte {
        switch {
        case '0' <= c && c <= '9':
            return c - '0'
        case 'a' <= c && c <= 'f':
            return c - 'a' + 10
        case 'A' <= c && c <= 'F':
            return c - 'A' + 10
        }
        return 0
    }
    ```
- There is no automatic fall through, but cases can be presented in comma-separated lists.
    ```go
        func shouldEscape(c byte) bool {
        switch c {
        case ' ', '?', '&', '=', '#', '+', '%':
            return true
        }
        return false
    }
    ```
- Although they are not nearly as common in Go as some other C-like languages, break statements can be used to terminate a switch early. Sometimes, though, it's necessary to break out of a surrounding loop, not the switch, and in Go that can be accomplished by putting a label on the loop and "breaking" to that label. This example shows both uses.
    ```go
    Loop:
        for n := 0; n < len(src); n += size {
            switch {
            case src[n] < sizeOne:
                if validateOnly {
                    break
                }
                size = 1
                update(src[n])

            case src[n] < sizeTwo:
                if n+1 >= len(src) {
                    err = errShortInput
                    break Loop
                }
                if validateOnly {
                    break
                }
                size = 2
                update(src[n] + src[n+1]<<shift)
            }
        }
    ```
- Of course, the continue statement also accepts an optional label but it applies only to loops.
#### Type switch
A switch can also be used to discover the dynamic type of an interface variable. Such a type switch uses the syntax of a type assertion with the keyword `type` inside the parentheses. If the switch declares a variable in the expression, the variable will have the corresponding type in each clause. It's also idiomatic to reuse the name in such cases, in effect declaring a new variable with the same name but a different type in each case.
```go
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t)     // %T prints whatever type t has
case bool:
    fmt.Printf("boolean %t\n", t)             // t has type bool
case int:
    fmt.Printf("integer %d\n", t)             // t has type int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t has type *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t has type *int
}
```

### Functions
#### Multiple return values
- One of Go's unusual features is that functions and methods can return multiple values. This form can be used to improve on a couple of clumsy idioms in C programs: in-band error returns such as -1 for EOF and modifying an argument passed by address.
#### Named result parameters
- The return or result "parameters" of a Go function can be given names and used as regular variables, just like the incoming parameters. When named, they are initialized to the zero values for their types when the function begins; if the function executes a return statement with no arguments, the current values of the result parameters are used as the returned values.
- The names are not mandatory but they can make code shorter and clearer: they're documentation.
#### Defer
- Go's defer statement schedules a function call (the deferred function) to be run immediately before the function executing the defer returns. It's an unusual but effective way to deal with situations such as resources that must be released regardless of which path a function takes to return.
    ```go {title="The canonical examples are unlocking a mutex or closing a file."}
    // Contents returns the file's contents as a string.
    func Contents(filename string) (string, error) {
        f, err := os.Open(filename)
        if err != nil {
            return "", err
        }
        defer f.Close()  // f.Close will run when we're finished.

        var result []byte
        buf := make([]byte, 100)
        for {
            n, err := f.Read(buf[0:])
            result = append(result, buf[0:n]...) // append is discussed later.
            if err != nil {
                if err == io.EOF {
                    break
                }
                return "", err  // f will be closed if we return here.
            }
        }
        return string(result), nil // f will be closed if we return here.
    }
    ```
- Deferring a call to a function such as Close has two advantages.
    - First, it guarantees that you will never forget to close the file, a mistake that's easy to make if you later edit the function to add a new return path.
    - Second, it means that the close sits near the open, which is much clearer than placing it at the end of the function.
- **The arguments to the deferred function (which include the receiver if the function is a method) are evaluated when the *defer* executes, not when the *call* executes.**
    - Besides avoiding worries about variables changing values as the function executes, this means that a single deferred call site can defer multiple function executions.
- **Deferred functions are executed in LIFO order.**

### Data
Go has two allocation primitives, the built-in functions `new` and `make`. They do different things and apply to different types.
#### Allocation with new
- It's a built-in function that allocates memory, but unlike its namesakes in some other languages it does not *initialize* the memory, it only *zeros* it.
    - That is, new(T) allocates zeroed storage for a new item of type T and returns its address, a value of type *T. In Go terminology, it returns a pointer to a newly allocated zero value of type T.
- Since the memory returned by new is zeroed, it's helpful to arrange when designing your data structures that **the zero value of each type can be used without further initialization**. This means a user of the data structure can create one with new and get right to work.
    - For example, the documentation for bytes.Buffer states that "the zero value for Buffer is an empty buffer ready to use." Similarly, sync.Mutex does not have an explicit constructor or Init method. Instead, the zero value for a sync.Mutex is defined to be an unlocked mutex.
    - **The zero-value-is-useful property works transitively.**
#### Constructors and composite literals
- Sometimes the zero value isn't good enough and an initializing constructor is necessary, as in this example derived from package os.
    ```go
    func NewFile(fd int, name string) *File {
        if fd < 0 {
            return nil
        }
        f := new(File)
        f.fd = fd
        f.name = name
        f.dirinfo = nil
        f.nepipe = 0
        return f
    }
    ```
- There's a lot of boilerplate in there. We can simplify it using a *composite literal*, which is an expression that creates a new instance each time it is evaluated.
    ```go
    func NewFile(fd int, name string) *File {
        if fd < 0 {
            return nil
        }
        f := File{fd, name, nil, 0}
        return &f
    }
    ```
- Note that, unlike in C, it's perfectly OK to return the address of a local variable; the storage associated with the variable survives after the function returns. 
    - In fact, taking the address of a composite literal allocates a fresh instance each time it is evaluated, so we can combine these last two lines. `return &File{fd, name, nil, 0}`
- The fields of a composite literal are laid out in order and must all be present. However, by labeling the elements explicitly as field:value pairs, the initializers can appear in any order, with the missing ones left as their respective zero values. `return &File{fd: fd, name: name}`
- As a limiting case, if a composite literal contains no fields at all, it creates a zero value for the type. The expressions `new(File)` and `&File{}` are equivalent.
- Composite literals can also be created for arrays, slices, and maps, with the field labels being indices or map keys as appropriate. In these examples, the initializations work regardless of the values of Enone, Eio, and Einval, as long as they are distinct.
    > 对于数组和切片, 如果Enone, Eio, Einval不连续或三者最小值大于0, 空缺的部分会被自动填充零值
    >
    > `a := [...]string   {Enone: "no error"}`: 如果Enone是100, 编译器会创建一个 长度为 101 的数组, 索引 0 到 99 会被自动填充为零值（空字符串 ""），只有索引 100 是 "no error"
    ```go
    a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
    s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
    m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
    ```
#### Allocation with make
- Back to allocation. The built-in function make(T, args) serves a purpose different from new(T). It creates slices, maps, and channels only, and it returns an initialized (not zeroed) value of type T (not *T).
    - The reason for the distinction is that these three types represent, under the covers, references to data structures that **must be initialized before use**. A slice, for example, is a three-item descriptor containing a pointer to the data (inside an array), the length, and the capacity, and until those items are initialized, the slice is nil. For slices, maps, and channels, make initializes the internal data structure and prepares the value for use.
    - For instance, `make([]int, 10, 100)` allocates an array of 100 ints and then creates a slice structure with length 10 and a capacity of 100 pointing at the first 10 elements of the array. In contrast, new([]int) returns a pointer to a newly allocated, zeroed slice structure, that is, a pointer to a nil slice value.
    ```go {title="These examples illustrate the difference between new and make."}
    var p *[]int = new([]int)       // allocates slice structure; *p == nil; rarely useful
    var v  []int = make([]int, 100) // the slice v now refers to a new array of 100 ints

    // Unnecessarily complex:
    var p *[]int = new([]int)
    *p = make([]int, 100, 100)

    // Idiomatic:
    v := make([]int, 100)
    ```
- **Remember that make applies only to maps, slices and channels and does not return a pointer. To obtain an explicit pointer allocate with new or take the address of a variable explicitly.**
#### Arrays
- Arrays are useful when planning the detailed layout of memory and sometimes can help avoid allocation, but primarily they are a building block for slices.
- There are major differences between the ways arrays work in Go and C. In Go,
    - Arrays are values. Assigning one array to another copies all the elements.
    - In particular, if you pass an array to a function, it will receive a *copy* of the array, not a pointer to it.
    - The size of an array is part of its type. The types [10]int and [20]int are distinct.
#### Slices
- Slices wrap arrays to give a more general, powerful, and convenient interface to sequences of data. Except for items with explicit dimension such as transformation matrices, most array programming in Go is done with slices rather than simple arrays.
- Slices hold references to an underlying array, and if you assign one slice to another, both refer to the same array. If a function takes a slice argument, changes it makes to the elements of the slice will be visible to the caller, analogous to passing a pointer to the underlying array. 
    - A Read function can therefore accept a slice argument rather than a pointer and a count; the length within the slice sets an upper limit of how much data to read. `func (f *File) Read(buf []byte) (n int, err error)`
- The length of a slice may be changed as long as it still fits within the limits of the underlying array; just assign it to a slice of itself.
- The capacity of a slice, accessible by the built-in function cap, reports the maximum length the slice may assume.
#### Two-dimensional slices
Sometimes it's necessary to allocate a 2D slice, a situation that can arise when processing scan lines of pixels, for instance. There are two ways to achieve this. One is to allocate each slice independently; the other is to allocate a single array and point the individual slices into it. Which to use depends on your application. If the slices might grow or shrink, they should be allocated independently to avoid overwriting the next line; if not, it can be more efficient to construct the object with a single allocation. For reference, here are sketches of the two methods.
```go {title="a line at a time"}
// Allocate the top-level slice.
picture := make([][]uint8, YSize) // One row per unit of y.
// Loop over the rows, allocating the slice for each row.
for i := range picture {
    picture[i] = make([]uint8, XSize)
}
```
```go {title="one allocation, sliced into lines"}
// Allocate the top-level slice, the same as before.
picture := make([][]uint8, YSize) // One row per unit of y.
// Allocate one large slice to hold all the pixels.
pixels := make([]uint8, XSize*YSize) // Has type []uint8 even though picture is [][]uint8.
// Loop over the rows, slicing each row from the front of the remaining pixels slice.
for i := range picture {
    picture[i], pixels = pixels[:XSize], pixels[XSize:]
}
```
#### Maps
- Maps are a convenient and powerful built-in data structure that associate values of one type (the key) with values of another type (the element or value).
    - The key can be of any type for which the equality operator is defined, such as integers, floating point and complex numbers, strings, pointers, interfaces (as long as the dynamic type supports equality), structs and arrays. Slices cannot be used as map keys, because equality is not defined on them.
- Like slices, maps hold references to an underlying data structure. If you pass a map to a function that changes the contents of the map, the changes will be visible in the caller.
- Maps can be constructed using the usual composite literal syntax with colon-separated key-value pairs, so it's easy to build them during initialization.
    ```go
    var timeZone = map[string]int{
        "UTC":  0*60*60,
        "EST": -5*60*60,
        "CST": -6*60*60,
        "MST": -7*60*60,
        "PST": -8*60*60,
    }
    ```
- Assigning and fetching map values looks syntactically just like doing the same for arrays and slices except that the index doesn't need to be an integer. `offset := timeZone["EST"]`
- An attempt to fetch a map value with a key that is not present in the map will return the zero value for the type of the entries in the map.
- A set can be implemented as a map with value type bool. Set the map entry to true to put the value in the set, and then test it by simple indexing.
- Sometimes you need to distinguish a missing entry from a zero value. Is there an entry for "UTC" or is that 0 because it's not in the map at all? You can discriminate with a form of multiple assignment.
    - In this example, if tz is present, seconds will be set appropriately and ok will be true; if not, seconds will be set to zero and ok will be false. Here's a function that puts it together with a nice error report:
    ```go
    func offset(tz string) int {
        if seconds, ok := timeZone[tz]; ok {
            return seconds
        }
        log.Println("unknown time zone:", tz)
        return 0
    }
    ```
- To delete a map entry, use the delete built-in function, whose arguments are the map and the key to be deleted. It's safe to do this even if the key is already absent from the map. `delete(timeZone, "PDT")  // Now on Standard Time`
#### Printing
- Formatted printing in Go uses a style similar to C's printf family but is richer and more general. The functions live in the fmt package and have capitalized names: fmt.Printf, fmt.Fprintf, fmt.Sprintf and so on. The string functions (Sprintf etc.) return a string rather than filling in a provided buffer.
- You don't need to provide a format string. For each of Printf, Fprintf and Sprintf there is another pair of functions, for instance Print and Println. These functions do not take a format string but instead generate a default format for each argument. The Println versions also insert a blank between arguments and append a newline to the output while the Print versions add blanks only if the operand on neither side is a string.
    ```go {title="same output"}
    fmt.Printf("Hello %d\n", 23)
    fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
    fmt.Println("Hello", 23)
    fmt.Println(fmt.Sprint("Hello ", 23))
    ``` 
- The formatted print functions fmt.Fprint and friends take as a first argument any object that implements the io.Writer interface; the variables os.Stdout and os.Stderr are familiar instances.
> *Here things start to diverge from C.*
- First, the numeric formats such as %d do not take flags for signedness or size; instead, the printing routines use the type of the argument to decide these properties.
- If you just want the default conversion, such as decimal for integers, you can use the catchall format %v (for “value”); the result is exactly what Print and Println would produce. Moreover, that format can print any value, even arrays, slices, structs, and maps.
    - For maps, Printf and friends sort the output lexicographically by key.
- When printing a struct, the modified format %+v annotates the fields of the structure with their names, and for any value the alternate format %#v prints the value in full Go syntax.
    ```go
    type T struct {
        a int
        b float64
        c string
    }
    t := &T{ 7, -2.35, "abc\tdef" }
    fmt.Printf("%v\n", t)
    fmt.Printf("%+v\n", t)
    fmt.Printf("%#v\n", t)
    fmt.Printf("%#v\n", timeZone)
    ```
    ```console
    &{7 -2.35 abc   def}
    &{a:7 b:-2.35 c:abc     def}
    &main.T{a:7, b:-2.35, c:"abc\tdef"}
    map[string]int{"CST":-21600, "EST":-18000, "MST":-25200, "PST":-28800, "UTC":0}
    ```
- That quoted string format is also available through %q when applied to a value of type string or []byte. The alternate format %#q will use backquotes instead if possible. (The %q format also applies to integers and runes, producing a single-quoted rune constant.)
- Also, %x works on strings, byte arrays and byte slices as well as on integers, generating a long hexadecimal string, and with a space in the format (% x) it puts spaces between the bytes.
- Another handy format is %T, which prints the *type* of a value.
- **If you want to control the default format for a custom type, all that's required is to define a method with the signature String() string on the type.** (If you need to print values of type T as well as pointers to T, the receiver for String must be of value type)
- Our String method is able to call Sprintf because the print routines are fully reentrant and can be wrapped this way. There is one important detail to understand about this approach, however: don't construct a String method by calling Sprintf in a way that will recur into your String method indefinitely. This can happen if the Sprintf call attempts to print the receiver directly as a string, which in turn will invoke the method again. It's a common and easy mistake to make, as this example shows.
    ```go
    type MyString string

    func (m MyString) String() string {
        return fmt.Sprintf("MyString=%s", m) // Error: will recur forever.
    }
    ```
    ```go {title="It's also easy to fix: convert the argument to the basic string type, which does not have the method."}
    type MyString string
    func (m MyString) String() string {
        return fmt.Sprintf("MyString=%s", string(m)) // OK: note conversion.
    }
    ```
- Another printing technique is to pass a print routine's arguments directly to another such routine. The signature of Printf uses the type ...interface{} for its final argument to specify that an arbitrary number of parameters (of arbitrary type) can appear after the format(`func Printf(format string, v ...interface{}) (n int, err error) {`). Within the function Printf, v acts like a variable of type []interface{} but if it is passed to another variadic function, it acts like a regular list of arguments. Here is the implementation of the function log.Println we used above. It passes its arguments directly to fmt.Sprintln for the actual formatting.
    ```go
    // Println prints to the standard logger in the manner of fmt.Println.
    func Println(v ...interface{}) {
        // We write ... after v in the nested call to Sprintln to tell the compiler to treat v as a list of arguments; otherwise it would just pass v as a single slice argument.
        std.Output(2, fmt.Sprintln(v...))  // Output takes parameters (int, string)
    }
    ```
- By the way, a ... parameter can be of a specific type, for instance ...int for a min function that chooses the least of a list of integers:
    ```go
    func Min(a ...int) int {
        min := int(^uint(0) >> 1)  // largest int
        for _, i := range a {
            if i < min {
                min = i
            }
        }
        return min
    }
    ```
#### Append
- Schematically, it's like this: `func append(slice []T, elements ...T) []T`, where T is a placeholder for any given type. You can't actually write a function in Go where the type T is determined by the caller. That's why append is built in: it needs support from the compiler.(Go 1.18后支持泛型)
- What append does is append the elements to the end of the slice and return the result. The result needs to be returned because, as with our hand-written Append, the underlying array may change. 

### Initialization
Complex structures can be built during initialization and the ordering issues among initialized objects, even among different packages, are handled correctly.
#### Constants
- Constants in Go are just that—constant. They are created at compile time, even when defined as locals in functions, and can only be numbers, characters (runes), strings or booleans. 
- Because of the compile-time restriction, the expressions that define them must be constant expressions, evaluatable by the compiler.
    > For instance, 1<<3 is a constant expression, while math.Sin(math.Pi/4) is not because the function call to math.Sin needs to happen at run time.
- In Go, enumerated constants are created using the iota enumerator. Since iota can be part of an expression and expressions can be implicitly repeated, it is easy to build intricate sets of values.
    > iota 是一个在 const 块里自动从 0 开始递增的常量, 每进入一个新的 const 块, iota 会重新从 0 开始
    > 
    > 在 const 括号里, 如果下一行什么都不写, Go 编译器会默认复制上一行的公式
    ```go
    type ByteSize float64

    const (
        _           = iota // ignore first value by assigning to blank identifier
        KB ByteSize = 1 << (10 * iota)
        MB
        GB
        TB
        PB
        EB
        ZB
        YB
    )
    ```
#### Variables
Variables can be initialized just like constants but the initializer can be a general expression computed at run time.
```go
var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)
```
#### The init function
- Finally, each source file can define its own niladic init function to set up whatever state is required. (Actually each file can have multiple init functions.) And finally means finally: **init** is called after all the **variable declarations in the package** have evaluated their initializers, and those are evaluated only after all the **imported packages** have been initialized.
    > Imported Packages -> Variable Declarations -> Init Functions
- Besides initializations that cannot be expressed as declarations(比如一个用来填充查找表的循环), a common use of init functions is to verify or repair correctness of the program state before real execution begins.
```go
func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    // gopath may be overridden by --gopath flag on command line.
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```

### Methods
#### Pointers vs. Values
- Methods can be defined for any named type (except a pointer or an interface); the receiver does not have to be a struct.
    > 只能给“定义在当前包 (Current Package)”内的类型定义方法
- The rule about pointers vs. values for receivers is that value methods can be invoked on pointers and values, but pointer methods can only be invoked on pointers.
- This rule arises because pointer methods can modify the receiver; invoking them on a value would cause the method to receive a copy of the value, so any modifications would be discarded. The language therefore disallows this mistake. *There is a handy exception, though. When **the value is addressable**, the language takes care of the common case of invoking a pointer method on a value by inserting the address operator automatically.*

### Interfaces and other types
#### Interfaces
- Interfaces in Go provide a way to specify the behavior of an object: if something can do *this*, then it can be used *here*.
- Interfaces with only one or two methods are common in Go code, and are usually given a name derived from the method, such as io.Writer for something that implements Write.
- **A type need not declare explicitly that it implements an interface. Instead, a type implements the interface just by implementing the interface's methods.**
- A type can implement multiple interfaces.
#### Conversions
- Because the two types (Sequence and []int) are the same if we ignore the type name, it's legal to convert between them. The conversion doesn't create a new value, it just temporarily acts as though the existing value has a new type. (There are other legal conversions, such as from integer to floating point, that do create a new value.)
```go
type Sequence []int
...
fmt.Sprint([]int(s))
```
- It's an idiom in Go programs to convert the type of an expression to access a different set of methods. `sort.IntSlice(s).Sort()`
#### Interface conversions and type assertions
- [Type switches](https://golang.google.cn/doc/effective_go#type_switch) are a form of conversion: they take an interface and, for each case in the switch, in a sense convert it to the type of that case.
- A type assertion takes an interface value and extracts from it a value of the specified explicit type. The syntax borrows from the clause opening a type switch, but with an explicit type rather than the type keyword: `value.(typeName)`, and the result is a new value with the static type typeName. That type must either be the concrete type held by the interface, or a second interface type that the value can be converted to. 
    > But if it turns out that the value does not contain a string, the program will crash with a run-time error. To guard against that, use the "comma, ok" idiom to test, safely, whether the value is a string: `str, ok := value.(string)`.
    > 
    > If the type assertion fails, str will still exist and be of type string, but it will have the zero value, an empty string.
#### Generality
- If a type exists only to implement an interface and will never have exported methods beyond that interface, there is no need to export the type itself. Exporting just the interface makes it clear the value has no interesting behavior beyond what is described in the interface. It also avoids the need to repeat the documentation on every instance of a common method. **In such cases, the constructor should return an interface value rather than the implementing type.**
#### Interfaces and methods
- Interfaces are just sets of methods, which can be defined for (almost) any type.
- Since we can define a method for any type except pointers and interfaces, we can write a method for a function. 
```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers.  If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler object that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, req).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}

...

// Argument server.
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}

...

http.Handle("/args", http.HandlerFunc(ArgServer))
```

### The blank identifier
The blank identifier can be assigned or declared with any value of any type, with the value discarded harmlessly. It's a bit like writing to the Unix /dev/null file: it represents a write-only value to be used as a place-holder where a variable is needed but the actual value is irrelevant.
#### The blank identifier in multiple assignment
If an assignment requires multiple values on the left side, but one of the values will not be used by the program, a blank identifier on the left-hand-side of the assignment avoids the need to create a dummy variable and makes it clear that the value is to be discarded.
> Occasionally you'll see code that discards the error value in order to ignore the error; this is terrible practice. Always check error returns; they're provided for a reason.
#### Unused imports and variables
- It is an error to import a package or to declare a variable without using it. Unused imports bloat the program and slow compilation, while a variable that is initialized but not used is at least a wasted computation and perhaps indicative of a larger bug. When a program is under active development, however, unused imports and variables often arise and it can be annoying to delete them just to have the compilation proceed, only to have them be needed again later. The blank identifier provides a workaround.
- To silence complaints about the unused imports, use a blank identifier to refer to a symbol from the imported package. Similarly, assigning the unused variable fd to the blank identifier will silence the unused variable error.
    > By convention, the global declarations to silence import errors should come right after the imports and be commented, both to make them easy to find and as a reminder to clean things up later.
    ```go
    package main

    import (
        "fmt"
        "io"
        "log"
        "os"
    )

    var _ = fmt.Printf // For debugging; delete when done. fmt.Printf是一个值
    var _ io.Reader    // For debugging; delete when done. io.Reader是一个类型

    func main() {
        fd, err := os.Open("test.go")
        if err != nil {
            log.Fatal(err)
        }
        // TODO: use fd.
        _ = fd
    }
    ```
#### Import for side effect
- An unused import like fmt or io in the previous example should eventually be used or removed: blank assignments identify code as a work in progress. But sometimes it is useful to import a package only for its side effects, without any explicit use. For example, during its init function, the net/http/pprof package registers HTTP handlers that provide debugging information. It has an exported API, but most clients need only the handler registration and access the data through a web page. To import the package only for its side effects, rename the package to the blank identifier: `import _ "net/http/pprof"`
#### Interface checks
In practice, most interface conversions are static and therefore checked at compile time. For example, passing an *os.File to a function expecting an io.Reader will not compile unless *os.File implements the io.Reader interface.

Some interface checks do happen at run-time, though. One instance is in the encoding/json package, which defines a Marshaler interface. When the JSON encoder receives a value that implements that interface, the encoder invokes the value's marshaling method to convert it to JSON instead of doing the standard conversion. The encoder checks this property at run time with a type assertion like: `m, ok := val.(json.Marshaler)`

If it's necessary only to ask whether a type implements an interface, without actually using the interface itself, perhaps as part of an error check, use the blank identifier to ignore the type-asserted value:  `if _, ok := val.(json.Marshaler); ok `

To guarantee that the implementation is correct, a global declaration using the blank identifier can be used in the package(static conversions that would cause the compiler to verify this automatically): `var _ json.Marshaler = (*RawMessage)(nil)`
> In this declaration, the assignment involving a conversion of a *RawMessage to a Marshaler requires that *RawMessage implements Marshaler, and that property will **be checked at compile time**. Should the json.Marshaler interface change, this package will no longer compile and we will be on notice that it needs to be updated.
>
> The appearance of the blank identifier in this construct indicates that the declaration exists only for the type checking, not to create a variable.
>
> Don't do this for every type that satisfies an interface, though. By convention, such declarations are only used when there are no static conversions already present in the code, which is a *rare* event.

### Embedding
- Go does not provide the typical, type-driven notion of subclassing, but it does have the ability to “borrow” pieces of an implementation by embedding types within a struct or interface.
- Interface embedding is very simple. Only interfaces can be embedded within interfaces.
    > A ReadWriter can do what a Reader does and what a Writer does; it is a union of the embedded interfaces.
    ```go {}
    type Reader interface {
        Read(p []byte) (n int, err error)
    }

    type Writer interface {
        Write(p []byte) (n int, err error)
    }

    // ReadWriter is the interface that combines the Reader and Writer interfaces.
    type ReadWriter interface {
        Reader
        Writer
    }
    ``` 
- The same basic idea applies to structs, but with more far-reaching implications. The bufio package has two struct types, bufio.Reader and bufio.Writer, each of which of course implements the analogous interfaces from package io. And bufio also implements a buffered reader/writer, which it does by combining a reader and a writer into one struct using embedding: **it lists the types within the struct but does not give them field names. If we need to refer to an embedded field directly, the type name of the field, ignoring the package qualifier, serves as a field name.**
    > The methods of embedded types come along for free, which means that bufio.ReadWriter not only has the methods of bufio.Reader and bufio.Writer, it also satisfies all three interfaces: io.Reader, io.Writer, and io.ReadWriter.
    >
    > There's an important way in which embedding differs from subclassing. When we embed a type, the methods of that type become methods of the outer type, but when they are invoked the receiver of the method is the inner type, not the outer one. In our example, when the Read method of a bufio.ReadWriter is invoked, it has exactly the same effect as the forwarding method written out above; the receiver is the reader field of the ReadWriter, not the ReadWriter itself.
    ```go
    // ReadWriter stores pointers to a Reader and a Writer.
    // It implements io.ReadWriter.
    type ReadWriter struct {
        *Reader  // *bufio.Reader
        *Writer  // *bufio.Writer
    }
    ```
- Embedding types introduces the problem of name conflicts but the rules to resolve them are simple.
    - First, a field or method X hides any other item X in a more deeply nested part of the type. If log.Logger contained a field or method called Command, the Command field of Job would dominate it. (相当于层序遍历, 浅层的压制深层的)
        ```go
        type Job struct {
            Command string
            *log.Logger
        }
        ```
    - Second, if the same name appears at the same nesting level, it is usually an error; it would be erroneous to embed log.Logger if the Job struct contained another field or method called Logger. However, if the duplicate name is never mentioned in the program outside the type definition, it is OK. This qualification provides some protection against changes made to types embedded from outside; there is no problem if a field is added that conflicts with another field in another subtype if neither field is ever used.

### Concurrency
#### Share by communicating
Concurrent programming in many environments is made difficult by the subtleties required to implement correct access to shared variables. Go encourages a different approach in which shared values are passed around on channels and, in fact, never actively shared by separate threads of execution. Only one goroutine has access to the value at any given time. Data races cannot occur, by design. To encourage this way of thinking we have reduced it to a slogan: **Do not communicate by sharing memory; instead, share memory by communicating.**
> One way to think about this model is to consider a typical single-threaded program running on one CPU. It has no need for synchronization primitives. Now run another such instance; it too needs no synchronization. Now let those two communicate; if the communication is the synchronizer, there's still no need for other synchronization.
#### Goroutines
- They're called *goroutines* because the existing terms—threads, coroutines, processes, and so on—convey inaccurate connotations. 
- A goroutine has a simple model: it is a function executing concurrently with other goroutines in the same address space. It is lightweight, costing little more than the allocation of stack space. And the stacks start small, so they are cheap, and grow by allocating (and freeing) heap storage as required.
- Goroutines are multiplexed onto multiple OS threads so if one should block, such as while waiting for I/O, others continue to run. Their design hides many of the complexities of thread creation and management.
- Prefix a function or method call with the go keyword to run the call in a new goroutine. When the call completes, the goroutine exits, silently. (The effect is similar to the Unix shell's & notation for running a command in the background.)
#### Channels
- Like maps, channels are allocated with make, and the resulting value acts as a reference to an underlying data structure. If an optional integer parameter is provided, it sets the buffer size for the channel. The default is zero, for an unbuffered or synchronous channel.
- Unbuffered channels combine communication—the exchange of a value—with synchronization—guaranteeing that two calculations (goroutines) are in a known state.
- **Receivers always block until there is data to receive. If the channel is unbuffered, the sender blocks until the receiver has received the value. If the channel has a buffer, the sender blocks only until the value has been copied to the buffer; if the buffer is full, this means waiting until some receiver has retrieved a value.**
```go {title="A buffered channel can be used like a semaphore, for instance to limit throughput."}
var sem = make(chan int, MaxOutstanding)

func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
```
#### Channels of channels
One of the most important properties of Go is that a channel is a first-class value that can be allocated and passed around like any other. A common use of this property is to implement safe, parallel demultiplexing.
```go
type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int
}

// client side
func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// Send request
clientRequests <- request
// Wait for response.
fmt.Printf("answer: %d\n", <-request.resultChan)

// server side
func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```
#### Parallelization
- Another application of these ideas is to parallelize a calculation across multiple CPU cores. If the calculation can be broken into separate pieces that can execute independently, it can be parallelized, with a channel to signal when each piece completes.
- The function [runtime.NumCPU](https://golang.google.cn/pkg/runtime#NumCPU) returns the number of hardware CPU cores in the machine
- There is also a function [runtime.GOMAXPROCS](https://golang.google.cn/pkg/runtime#GOMAXPROCS), which reports (or sets) the user-specified number of cores that a Go program can have running simultaneously. It defaults to the value of runtime.NumCPU but can be overridden by setting the similarly named shell environment variable or by calling the function with a positive number. Calling it with zero just queries the value.
- Be sure not to confuse the ideas of concurrency—structuring a program as independently executing components—and parallelism—executing calculations in parallel for efficiency on multiple CPUs. Although the concurrency features of Go can make some problems easy to structure as parallel computations, Go is a concurrent language, not a parallel one, and not all parallelization problems fit Go's model.

### Errors
- Go's multivalue return makes it easy to return a detailed error description alongside the normal return value. It is good style to use this feature to provide detailed error information.
```go {title="By convention, errors have type error, a simple built-in interface."}
type error interface {
    Error() string
}
```
- When feasible, error strings should identify their origin, such as by having a prefix naming the operation or package that generated the error. 
    > For example, in package image, the string representation for a decoding error due to an unknown format is "image: unknown format".
- Callers that care about the precise error details can use a type switch or a type assertion to look for specific errors and extract details.
#### Panic
- The usual way to report an error to a caller is to return an error as an extra return value. But what if the error is unrecoverable? Sometimes the program simply cannot continue. For this purpose, there is a built-in function `panic` that in effect creates a run-time error that will stop the program. The function takes a single argument of arbitrary type—often a string—to be printed as the program dies. It's also a way to indicate that something impossible has happened, such as exiting an infinite loop.
- If the problem can be masked or worked around, it's always better to let things continue to run rather than taking down the whole program. One possible counterexample is during initialization: if the library truly cannot set itself up, it might be reasonable to panic, so to speak.
    ```go
    var user = os.Getenv("USER")

    func init() {
        if user == "" {
            panic("no value for $USER")
        }
    }
    ```
#### Recover
- When panic is called, including implicitly for run-time errors such as indexing a slice out of bounds or failing a type assertion, it immediately stops execution of the current function and begins unwinding the stack of the goroutine, running any deferred functions along the way. If that unwinding reaches the top of the goroutine's stack, the program dies. However, it is possible to use the built-in function recover to regain control of the goroutine and resume normal execution.
- A call to `recover` stops the unwinding and returns the argument passed to panic. *Because the only code that runs while unwinding is inside deferred functions, recover is only useful inside deferred functions*.
- One application of recover is to shut down a failing goroutine inside a server without killing the other executing goroutines.
    > In this example, if do(work) panics, the result will be logged and the goroutine will exit cleanly without disturbing the others. There's no need to do anything else in the deferred closure; calling recover handles the condition completely.
    ```go
    func server(workChan <-chan *Work) {
        for work := range workChan {
            go safelyDo(work)
        }
    }

    func safelyDo(work *Work) {
        defer func() {
            if err := recover(); err != nil {
                log.Println("work failed:", err)
            }
        }()
        do(work)
    }
    ```
- Because recover always returns nil unless called **directly** from a deferred function, deferred code can call library routines that themselves use panic and recover without failing. As an example, the deferred function in safelyDo might call a logging function before calling recover, and that logging code would run unaffected by the panicking state.