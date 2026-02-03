+++
date = '2026-01-31T17:47:42+08:00'
draft = false
title = 'Week2 Day1'
+++
## Day 1: TDD 思维觉醒与基础工具链

1. [`go mod init`](https://golang.google.cn/ref/mod#go-mod-init): The go mod init command initializes and writes a new go.mod file in the current directory, in effect creating a new module rooted at the current directory. The go.mod file must not already exist. init accepts one optional argument, the module path for the new module.
    > Enable dependency tracking for your code
    - When your code imports packages contained in other modules, you manage those dependencies through your code's own module. That module is defined by a go.mod file that tracks the modules that provide those packages. That go.mod file stays with your code, including in your source code repository.
    - In actual development, the module path will typically be the repository location where your source code will be kept. For example, the module path might be github.com/mymodule. If you plan to publish your module for others to use, the module path must be a location from which Go tools can download your module.

2. [`go mod edit`](https://golang.google.cn/ref/mod#go-mod-edit): The go mod edit command provides a command-line interface for editing and formatting go.mod files, for use primarily by tools and scripts. By default, go mod edit reads and writes the go.mod file of the main module, but a different target file can be specified after the editing flags.
    > `go mod edit -replace example.com/greetings=../greetings`: The command specifies that example.com/greetings should be replaced with ../greetings for the purpose of locating the dependency.

3. [`go mod tidy`](https://golang.google.cn/ref/mod#go-mod-tidy): go mod tidy ensures that the go.mod file matches the source code in the module. It adds any missing module requirements necessary to build the current module’s packages and dependencies, and it removes requirements on modules that don’t provide any relevant packages. It also adds any missing entries to go.sum and removes unnecessary entries.

4. [Authenticating modules](https://golang.google.cn/ref/mod#authenticating)
    - When the go command downloads a module zip file or go.mod file into the module cache, it computes a cryptographic hash and compares it with a known value to verify the file hasn’t changed since it was first downloaded. The go command reports a security error if a downloaded file does not have the correct hash.
    - For go.mod files, the go command computes the hash from the file content. For module zip files, the go command computes the hash from the names and contents of files within the archive in a deterministic order. The hash is not affected by file order, compression, alignment, and other metadata.
    - *The go command compares each hash with the corresponding line in the main module’s go.sum file.* If the hash is different from the hash in go.sum, the go command reports a security error and deletes the downloaded file without adding it into the module cache.


5. In a module, you collect one or more related packages for a discrete and useful set of functions. Go code is grouped into packages, and packages are grouped into modules. Your module specifies dependencies needed to run your code, including the Go version and the set of other modules it requires.

6. The `go test` command executes test functions (whose names begin with Test) in test files (whose names end with _test.go). You can add the -v flag to get verbose output that lists all of the tests and their results.
    - Ending a file's name with _test.go tells the go test command that this file contains test functions. 
    - Test function names have the form Test*Name*, where *Name* says something about the specific test. Also, test functions take a pointer to the testing package's [testing.T type](https://golang.google.cn/pkg/testing/#T) as a parameter. You use this parameter's methods for reporting and logging from your test.
    - `t.Run`, subtests: Sometimes, it is useful to group tests around a "thing" and then have subtests describing different scenarios. 
        - A benefit of this approach is you can set up shared code that can be used in the other tests. For helper functions, it's a good idea to accept a `testing.TB` which is an interface that `*testing.T` and `*testing.B` both satisfy, so you can call helper functions from a test, or a benchmark. `t.Helper()` is needed to tell the test suite that this method is a helper. By doing this, when it fails, the line number reported will be in our *function call* rather than inside our test helper. This will help other developers track down problems more easily.

7. The [go build command](https://golang.google.cn/cmd/go/#hdr-Compile_packages_and_dependencies) compiles the packages, along with their dependencies, but it doesn't install the results. The [go install command](https://golang.google.cn/ref/mod#go-install) compiles and installs the packages.

8. **Cycle**
    ![TDD](/images/week2/day1/1.png)
    > It is important that your tests are clear specifications of what the code needs to do.
    > 
    > Seeing the test fail is an important check because it also lets you see what the error message looks like.
    >
    > By ensuring your tests are *fast* and setting up your tools so that running tests is simple you can get in to a state of flow when writing your code.
    - Write a test
    - Make the compiler pass
    - Run the test, see that it fails and check the error message is meaningful
    - Write enough code to make the test pass
    - Refactor

9. The TDD process and *why* the steps are important
    - *Write a failing test and see it fail* so we know we have written a *relevant* test for our requirements and seen that it produces an *easy to understand description of the failure*
    - Writing the smallest amount of code to make it pass so we know we have working software
    - *Then* refactor, backed with the safety of our tests to ensure we have well-crafted code that is easy to work with

10. **Go source files can only have one package per directory.** 
    > 该限制只看当前目录, 不管其父目录或者子目录的package
    - 目录名应等于包名 (Convention)
    - 父子目录通常代表逻辑层级

11. [Testable Examples](https://go.dev/blog/examples): Example functions begin with `Example` (much like test functions begin with `Test`), and reside in a package's `_test.go` files. 
    - Example functions are compiled whenever tests are executed. Because such examples are validated by the Go compiler, you can be confident your documentation's examples always reflect current code behavior.

12. [Benchmarking](https://pkg.go.dev/testing#hdr-Benchmarks): `go test -bench=.`
    - The `testing.B` gives you access to the loop function. `Loop()` returns true as long as the benchmark should continue running. When the benchmark code is executed, it measures how long it takes. After `Loop()` returns false, `b.N` contains the total number of iterations that ran.
    - **The number of times the code is run shouldn't matter to you, the framework will determine what is a "good" value for that to let you have some decent results.**
    - Only the body of the loop is timed; it automatically excludes setup and cleanup code from benchmark timing.
    ```go
    func BenchmarkXxx(b *testing.B) {
        //... setup ...
        for b.Loop() {
            //... code to measure ...
        }
        //... cleanup ...
    }
    ```
