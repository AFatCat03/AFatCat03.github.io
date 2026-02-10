+++
date = '2026-02-08T21:20:26+08:00'
draft = false
title = 'How to Write Go Code'
+++
## [How to Write Go Code](https://golang.google.cn/doc/code)

1. Code organization: Repository->Module->Package
    - Go programs are organized into packages. **A *package* is a collection of source files in the same directory that are compiled together.** Functions, types, variables, and constants defined in one source file are visible to all other source files within the same package.
    - **A *repository* contains one or more modules. A *module* is a collection of related Go packages that are released together.** A Go repository typically contains only one module, located at the root of the repository. A file named go.mod there declares the module path: the import path prefix for all packages within the module. The module contains the packages in the directory containing its go.mod file as well as subdirectories of that directory, up to the next subdirectory containing another go.mod file (if any).
        > 一个模块的作用域: 从当前目录开始, 向下递归, 直到遇到子目录里有另一个 go.mod 为止。一旦遇到新的 go.mod, 那个子目录就独立出去了, 不再属于当前模块
        >
        > Note that you don't need to publish your code to a remote repository before you can build it. A module can be defined locally without belonging to a repository. However, it's a good habit to organize your code as if you will publish it someday.
        >
        > Each module's path not only serves as an import path prefix for its packages, but also indicates where the go command should look to download it.
    - **An *import path* is a string used to import a package. A package's import path is its module path joined with its subdirectory within the module.** For example, the module github.com/google/go-cmp contains a package in the directory cmp/. That package's import path is github.com/google/go-cmp/cmp. **Packages in the standard library do not have a module path prefix.**

2. Your first program
    - The first statement in a Go source file must be package name. Executable commands must always use package main.
    - `go install example/user/hello`: builds the hello command, producing an executable binary. It then installs that binary as $HOME/go/bin/hello (or, under Windows, %USERPROFILE%\go\bin\hello.exe).
        > The install directory is controlled by the GOPATH and GOBIN environment variables. If GOBIN is set, binaries are installed to that directory. If GOPATH is set, binaries are installed to the bin subdirectory of the first directory in the GOPATH list. Otherwise, binaries are installed to the bin subdirectory of the default GOPATH ($HOME/go or %USERPROFILE%\go).
        >
    - You can use the go env command to portably set the default value for an environment variable for future go commands: `go env -w GOBIN=/somewhere/else/bin`
    - To unset a variable previously set by go env -w, use go env -u: `go env -u GOBIN`
    - Commands like go install apply within the context of the module containing the current working directory. If the working directory is not within the example/user/hello module, go install may fail. For convenience, go commands accept paths relative to the working directory, and default to the package in the current working directory if no other path is given.
    - **Add the install directory to our PATH to make running binaries easy: `export PATH=$PATH:$(dirname $(go list -f '{{.Target}}' .))`**

3. Importing packages from your module
    - `go build`: This won't produce an output file. Instead it saves the compiled package in the local build cache.

4. Importing packages from remote modules
    - An import path can describe how to obtain the package source code using a revision control system such as Git or Mercurial. The go tool uses this property to automatically fetch packages from remote repositories. For instance, to use github.com/google/go-cmp/cmp in your program: `import "github.com/google/go-cmp/cmp"`
    - Now that you have a dependency on an external module, you need to download that module and record its version in your go.mod file. The `go mod tidy` command adds missing module requirements for imported packages and removes requirements on modules that aren't used anymore.
    - Module dependencies are automatically downloaded to the pkg/mod subdirectory of the directory indicated by the GOPATH environment variable. **The downloaded contents for a given version of a module are shared among all other modules that require that version, so the go command marks those files and directories as *read-only***. To remove all downloaded modules, you can pass the -modcache flag to go clean: `go clean -modcache`

5. Testing
    - Go has a lightweight test framework composed of the go test command and the testing package.
    - You write a test by creating a file with a name ending in _test.go that contains functions named TestXXX with signature func (t *testing.T). The test framework runs each such function; if the function calls a failure function such as t.Error or t.Fail, the test is considered to have failed.