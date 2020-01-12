# How to Write Go Code

https://golang.org/doc/code.html

https://go-zh.org/doc/code.html

https://www.youtube.com/watch?v=XCsL89YtqCs

## Introduction

This document demonstrates the development of a simple Go package and introduces the [go tool](https://golang.org/cmd/go/), the standard way to fetch, build, and install Go packages and commands.

The `go` tool requires you to organize your code in a specific way. Please read this document carefully. It explains the simplest way to get up and running with your Go installation.

A similar explanation is available as a [screencast](https://www.youtube.com/watch?v=XCsL89YtqCs).

## Code organization

### Overview

- Go programmers typically keep all their Go code in a single *workspace*.
- A workspace contains many version control *repositories* (managed by Git, for example).
- Each repository contains one or more *packages*.
- Each package consists of one or more Go source files in a single directory.
- The path to a package's directory determines its *import path*.

Note that this differs from other programming environments in which every project has a separate workspace and workspaces are closely tied to version control repositories.

### Workspaces

A workspace is a directory hierarchy with two directories at its root:

- `src` contains Go source files, and
- `bin` contains executable commands.

The `go` tool builds and installs binaries to the `bin` directory.

The `src` subdirectory typically contains multiple version control repositories (such as for Git or Mercurial) that track the development of one or more source packages.

To give you an idea of how a workspace looks in practice, here's an example:

```
bin/
    hello                          # command executable
    outyet                         # command executable
src/
    github.com/golang/example/
        .git/                      # Git repository metadata
	hello/
	    hello.go               # command source
	outyet/
	    main.go                # command source
	    main_test.go           # test source
	stringutil/
	    reverse.go             # package source
	    reverse_test.go        # test source
    golang.org/x/image/
        .git/                      # Git repository metadata
	bmp/
	    reader.go              # package source
	    writer.go              # package source
    ... (many more repositories and packages omitted) ...
```

The tree above shows a workspace containing two repositories (`example` and `image`). The `example` repository contains two commands (`hello` and `outyet`) and one library (`stringutil`). The `image` repository contains the `bmp`package and [several others](https://godoc.org/golang.org/x/image).

A typical workspace contains many source repositories containing many packages and commands. Most Go programmers keep *all* their Go source code and dependencies in a single workspace.

Note that symbolic links should **not** be used to link files or directories into your workspace.

Commands and libraries are built from different kinds of source packages. We will discuss the distinction [later](https://golang.org/doc/code.html#PackageNames).

### The `GOPATH` environment variable

The `GOPATH` environment variable specifies the location of your workspace. It defaults to a directory named `go`inside your home directory, so `$HOME/go` on Unix, `$home/go` on Plan 9, and `%USERPROFILE%\go` (usually `C:\Users\YourName\go`) on Windows.

If you would like to work in a different location, you will need to [set `GOPATH`](https://golang.org/wiki/SettingGOPATH) to the path to that directory. (Another common setup is to set `GOPATH=$HOME`.) Note that `GOPATH` must **not** be the same path as your Go installation.

The command `go` `env` `GOPATH` prints the effective current `GOPATH`; it prints the default location if the environment variable is unset.

For convenience, add the workspace's `bin` subdirectory to your `PATH`:

```
$ export PATH=$PATH:$(go env GOPATH)/bin
```

The scripts in the rest of this document use `$GOPATH` instead of `$(go env GOPATH)` for brevity. To make the scripts run as written if you have not set GOPATH, you can substitute $HOME/go in those commands or else run:

```
$ export GOPATH=$(go env GOPATH)
```

To learn more about the `GOPATH` environment variable, see [`'go help gopath'`](https://golang.org/cmd/go/#hdr-GOPATH_environment_variable).

### Import paths

An *import path* is a string that uniquely identifies a package. A package's import path corresponds to its location inside a workspace or in a remote repository (explained below).

The packages from the standard library are given short import paths such as `"fmt"` and `"net/http"`. For your own packages, you must choose a base path that is unlikely to collide with future additions to the standard library or other external libraries.

If you keep your code in a source repository somewhere, then you should use the root of that source repository as your base path. For instance, if you have a [GitHub](https://github.com/) account at `github.com/user`, that should be your base path.

Note that you don't need to publish your code to a remote repository before you can build it. It's just a good habit to organize your code as if you will publish it someday. In practice you can choose any arbitrary path name, as long as it is unique to the standard library and greater Go ecosystem.

We'll use `github.com/user` as our base path. Create a directory inside your workspace in which to keep source code:

```
$ mkdir -p $GOPATH/src/github.com/user
```

### Your first program

To compile and run a simple program, first choose a package path (we'll use `github.com/user/hello`) and create a corresponding package directory inside your workspace:

```
$ mkdir $GOPATH/src/github.com/user/hello
```

Next, create a file named `hello.go` inside that directory, containing the following Go code.

```
package main

import "fmt"

func main() {
	fmt.Println("Hello, world.")
}
```

Now you can build and install that program with the `go` tool:

```
$ go install github.com/user/hello
```

Note that you can run this command from anywhere on your system. The `go` tool finds the source code by looking for the `github.com/user/hello` package inside the workspace specified by `GOPATH`.

You can also omit the package path if you run `go install` from the package directory:

```
$ cd $GOPATH/src/github.com/user/hello
$ go install
```

This command builds the `hello` command, producing an executable binary. It then installs that binary to the workspace's `bin` directory as `hello` (or, under Windows, `hello.exe`). In our example, that will be `$GOPATH/bin/hello`, which is `$HOME/go/bin/hello`.

The `go` tool will only print output when an error occurs, so if these commands produce no output they have executed successfully.

You can now run the program by typing its full path at the command line:

```
$ $GOPATH/bin/hello
Hello, world.
```

Or, as you have added `$GOPATH/bin` to your `PATH`, just type the binary name:

```
$ hello
Hello, world.
```

If you're using a source control system, now would be a good time to initialize a repository, add the files, and commit your first change. Again, this step is optional: you do not need to use source control to write Go code.

```
$ cd $GOPATH/src/github.com/user/hello
$ git init
Initialized empty Git repository in /home/user/go/src/github.com/user/hello/.git/
$ git add hello.go
$ git commit -m "initial commit"
[master (root-commit) 0b4507d] initial commit
 1 file changed, 7 insertion(+)
 create mode 100644 hello.go
```

Pushing the code to a remote repository is left as an exercise for the reader.

### Your first library

Let's write a library and use it from the `hello` program.

Again, the first step is to choose a package path (we'll use `github.com/user/stringutil`) and create the package directory:

```
$ mkdir $GOPATH/src/github.com/user/stringutil
```

Next, create a file named `reverse.go` in that directory with the following contents.

```
// Package stringutil contains utility functions for working with strings.
package stringutil

// Reverse returns its argument string reversed rune-wise left to right.
func Reverse(s string) string {
	r := []rune(s)
	for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
		r[i], r[j] = r[j], r[i]
	}
	return string(r)
}
```

Now, test that the package compiles with `go build`:

```
$ go build github.com/user/stringutil
```

Or, if you are working in the package's source directory, just:

```
$ go build
```

This won't produce an output file. Instead it saves the compiled package in the local build cache.

After confirming that the `stringutil` package builds, modify your original `hello.go` (which is in`$GOPATH/src/github.com/user/hello`) to use it:

```
package main

import (
	"fmt"

	"github.com/user/stringutil"
)

func main() {
	fmt.Println(stringutil.Reverse("!oG ,olleH"))
}
```

Install the `hello` program:

```
$ go install github.com/user/hello
```

Running the new version of the program, you should see a new, reversed message:

```
$ hello
Hello, Go!
```

After the steps above, your workspace should look like this:

```
bin/
    hello                 # command executable
src/
    github.com/user/
        hello/
            hello.go      # command source
        stringutil/
            reverse.go    # package source
```

### Package names

The first statement in a Go source file must be

```
package name
```

where `*name*` is the package's default name for imports. (All files in a package must use the same `*name*`.)

Go's convention is that the package name is the last element of the import path: the package imported as "`crypto/rot13`" should be named `rot13`.

Executable commands must always use `package main`.

There is no requirement that package names be unique across all packages linked into a single binary, only that the import paths (their full file names) be unique.

See [Effective Go](https://golang.org/doc/effective_go.html#names) to learn more about Go's naming conventions.

## Testing

Go has a lightweight test framework composed of the `go test` command and the `testing` package.

You write a test by creating a file with a name ending in `_test.go` that contains functions named `TestXXX` with signature `func (t *testing.T)`. The test framework runs each such function; if the function calls a failure function such as `t.Error` or `t.Fail`, the test is considered to have failed.

Add a test to the `stringutil` package by creating the file `$GOPATH/src/github.com/user/stringutil/reverse_test.go`containing the following Go code.

```
package stringutil

import "testing"

func TestReverse(t *testing.T) {
	cases := []struct {
		in, want string
	}{
		{"Hello, world", "dlrow ,olleH"},
		{"Hello, 世界", "界世 ,olleH"},
		{"", ""},
	}
	for _, c := range cases {
		got := Reverse(c.in)
		if got != c.want {
			t.Errorf("Reverse(%q) == %q, want %q", c.in, got, c.want)
		}
	}
}
```

Then run the test with `go test`:

```
$ go test github.com/user/stringutil
ok  	github.com/user/stringutil 0.165s
```

As always, if you are running the `go` tool from the package directory, you can omit the package path:

```
$ go test
ok  	github.com/user/stringutil 0.165s
```

Run `go help test` and see the [testing package documentation](https://golang.org/pkg/testing/) for more detail.

## Remote packages

An import path can describe how to obtain the package source code using a revision control system such as Git or Mercurial. The `go` tool uses this property to automatically fetch packages from remote repositories. For instance, the examples described in this document are also kept in a Git repository hosted at GitHub`github.com/golang/example`. If you include the repository URL in the package's import path, `go get` will fetch, build, and install it automatically:

```
$ go get github.com/golang/example/hello
$ $GOPATH/bin/hello
Hello, Go examples!
```

If the specified package is not present in a workspace, `go get` will place it inside the first workspace specified by `GOPATH`. (If the package does already exist, `go get` skips the remote fetch and behaves the same as `go install`.)

After issuing the above `go get` command, the workspace directory tree should now look like this:

```
bin/
    hello                           # command executable
src/
    github.com/golang/example/
	.git/                       # Git repository metadata
        hello/
            hello.go                # command source
        stringutil/
            reverse.go              # package source
            reverse_test.go         # test source
    github.com/user/
        hello/
            hello.go                # command source
        stringutil/
            reverse.go              # package source
            reverse_test.go         # test source
```

The `hello` command hosted at GitHub depends on the `stringutil` package within the same repository. The imports in `hello.go` file use the same import path convention, so the `go get` command is able to locate and install the dependent package, too.

```
import "github.com/golang/example/stringutil"
```

This convention is the easiest way to make your Go packages available for others to use. The [Go Wiki](https://golang.org/wiki/Projects) and [godoc.org](https://godoc.org/) provide lists of external Go projects.

For more information on using remote repositories with the `go` tool, see `go help importpath`.

## What's next

Subscribe to the [golang-announce](https://groups.google.com/group/golang-announce) mailing list to be notified when a new stable version of Go is released.

See [Effective Go](https://golang.org/doc/effective_go.html) for tips on writing clear, idiomatic Go code.

Take [A Tour of Go](https://tour.golang.org/) to learn the language proper.

Visit the [documentation page](https://golang.org/doc/#articles) for a set of in-depth articles about the Go language and its libraries and tools.