+++
date = '2026-02-03T21:27:45+08:00'
draft = false
title = 'Week2 Day2'
+++
## Week2 Day 2: 切片内存布局与表格驱动测试

1. `go test -cover`: Whilst striving for 100% coverage should not be your end goal, the coverage tool can help identify areas of your code not covered by tests.
    - `go test -coverprofile=coverage.out`, `go tool cover -html=coverage.out`: 打开浏览器, 以红色标记未被测试覆盖的代码行, 帮助识别逻辑盲区

2. [Table driven tests](https://go.dev/wiki/TableDrivenTests) are useful when you want to build a list of test cases that can be tested in the same manner. They are a great fit when you wish to test various implementations of an interface, or if the data being passed in to a function has lots of different requirements that need testing.
    - One final tip with table driven tests is to use t.Run and to name the test cases. And you can run specific tests within your table with `go test -run TestArea/Rectangle`.
    ```go
    func TestArea(t *testing.T) {

        areaTests := []struct {
            name    string
            shape   Shape
            hasArea float64
        }{
            {name: "Rectangle", shape: Rectangle{Width: 12, Height: 6}, hasArea: 72.0},
            {name: "Circle", shape: Circle{Radius: 10}, hasArea: 314.1592653589793},
            {name: "Triangle", shape: Triangle{Base: 12, Height: 6}, hasArea: 36.0},
        }

        for _, tt := range areaTests {
            // using tt.name from the case to use it as the `t.Run` test name
            t.Run(tt.name, func(t *testing.T) {
                got := tt.shape.Area()
                if got != tt.hasArea {
                    t.Errorf("%#v got %g want %g", tt.shape, got, tt.hasArea)
                }
            })

        }

    }
    ```

3. slice
    - [slice结构](https://golang.google.cn/src/runtime/slice.go)
        ```go
        type slice struct {
            array unsafe.Pointer // 指向底层数组中第一个可访问元素的指针(不一定是底层数组的起始位置)
            len   int // 切片当前包含的元素个数
            cap   int // 从 array 指针开始，到底层数组末尾的元素个数
        }
        ```
    - 切片存在时底层数组不会被GC回收, 即使其只切取了数组的一小部分, 可以使用 [copy](https://golang.google.cn/pkg/builtin/#copy) 进行防御性拷贝
        - `func copy(dst, src []Type) int`: The copy built-in function copies elements from a source slice into a destination slice.  The source and destination may overlap. Copy returns the number of elements copied, which will be the minimum of len(src) and len(dst).
    - [append](https://golang.google.cn/pkg/builtin/#append): `func append(slice []Type, elems ...Type) []Type`. The append built-in function appends elements to the end of a slice. *If it has sufficient capacity, the destination is resliced to accommodate the new elements. If it does not, a **new** underlying array will be allocated.* Append returns the updated slice.
        > `fmt.Printf`用于 Slice时: %p address of 0th element in base 16 notation, with leading 0x
        ```go {title="测试重新分配地址"}
        var s []int = make([]int, 0)
        prevCap := -1
        for i := 0; i < 1024; i++ {
            if cap(s) != prevCap {
                fmt.Printf("Address of 0th element: %p, len: %d, cap: %d\n", s, len(s), cap(s))
                prevCap = cap(s)
            }
            s = append(s, 1)
        }
        ```
        ```console {title="结果"}
        $ go run slice_anatomy.go 
        Address of 0th element: 0x58f760, len: 0, cap: 0
        Address of 0th element: 0xc0000b2008, len: 1, cap: 1
        Address of 0th element: 0xc0000b2020, len: 2, cap: 2
        Address of 0th element: 0xc0000d6000, len: 3, cap: 4
        Address of 0th element: 0xc0000d8000, len: 5, cap: 8
        Address of 0th element: 0xc0000da000, len: 9, cap: 16
        Address of 0th element: 0xc0000dc000, len: 17, cap: 32
        Address of 0th element: 0xc0000de000, len: 33, cap: 64
        Address of 0th element: 0xc0000e0000, len: 65, cap: 128
        Address of 0th element: 0xc0000e2000, len: 129, cap: 256
        Address of 0th element: 0xc0000e4000, len: 257, cap: 512
        Address of 0th element: 0xc0000e6000, len: 513, cap: 848
        Address of 0th element: 0xc0000f0000, len: 849, cap: 1280
        ```
        ```console {title="cap够时不重新分配底层数组"}
        $ go run slice_anatomy.go 
        Address of 0th element: 0x58f760, len: 0, cap: 0
        Address of 0th element: 0xc000012098, len: 1, cap: 1
        Address of 0th element: 0xc0000120b0, len: 2, cap: 2
        Address of 0th element: 0xc00001c120, len: 3, cap: 4
        Address of 0th element: 0xc00001c120, len: 4, cap: 4
        Address of 0th element: 0xc00001e080, len: 5, cap: 8
        Address of 0th element: 0xc00001e080, len: 6, cap: 8
        Address of 0th element: 0xc00001e080, len: 7, cap: 8
        Address of 0th element: 0xc00001e080, len: 8, cap: 8
        Address of 0th element: 0xc000024100, len: 9, cap: 16
        ```