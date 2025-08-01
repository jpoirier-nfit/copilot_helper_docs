---
source_url: https://go.dev/blog/using-go-modules
title: Using Go Modules - The Go Programming Language
crawl_date: 2025-07-25T12:08:27.442108
watsonmd_version: 0.1.0
---

# [The Go Blog](/blog/)

# Using Go Modules

Tyler Bui-Palsulich and Eno Compton  
19 March 2019 

## Introduction

This post is part 1 in a series.

  * **Part 1 — Using Go Modules** (this post)
  * Part 2 — [Migrating To Go Modules](/blog/migrating-to-go-modules)
  * Part 3 — [Publishing Go Modules](/blog/publishing-go-modules)
  * Part 4 — [Go Modules: v2 and Beyond](/blog/v2-go-modules)
  * Part 5 — [Keeping Your Modules Compatible](/blog/module-compatibility)



**Note:** For documentation on managing dependencies with modules, see [Managing dependencies](/doc/modules/managing-dependencies).

Go 1.11 and 1.12 include preliminary [support for modules](/doc/go1.11#modules), Go’s [new dependency management system](/blog/versioning-proposal) that makes dependency version information explicit and easier to manage. This blog post is an introduction to the basic operations needed to get started using modules.

A module is a collection of [Go packages](/ref/spec#Packages) stored in a file tree with a `go.mod` file at its root. The `go.mod` file defines the module’s _module path_ , which is also the import path used for the root directory, and its _dependency requirements_ , which are the other modules needed for a successful build. Each dependency requirement is written as a module path and a specific [semantic version](http://semver.org/).

As of Go 1.11, the go command enables the use of modules when the current directory or any parent directory has a `go.mod`, provided the directory is _outside_ `$GOPATH/src`. (Inside `$GOPATH/src`, for compatibility, the go command still runs in the old GOPATH mode, even if a `go.mod` is found. See the [go command documentation](/cmd/go/#hdr-Preliminary_module_support) for details.) Starting in Go 1.13, module mode will be the default for all development.

This post walks through a sequence of common operations that arise when developing Go code with modules:

  * Creating a new module.
  * Adding a dependency.
  * Upgrading dependencies.
  * Adding a dependency on a new major version.
  * Upgrading a dependency to a new major version.
  * Removing unused dependencies.



## Creating a new module

Let’s create a new module.

Create a new, empty directory somewhere outside `$GOPATH/src`, `cd` into that directory, and then create a new source file, `hello.go`:
    
    
    package hello
    
    func Hello() string {
        return "Hello, world."
    }
    

Let’s write a test, too, in `hello_test.go`:
    
    
    package hello
    
    import "testing"
    
    func TestHello(t *testing.T) {
        want := "Hello, world."
        if got := Hello(); got != want {
            t.Errorf("Hello() = %q, want %q", got, want)
        }
    }
    

At this point, the directory contains a package, but not a module, because there is no `go.mod` file. If we were working in `/home/gopher/hello` and ran `go test` now, we’d see:
    
    
    $ go test
    PASS
    ok      _/home/gopher/hello 0.020s
    $
    

The last line summarizes the overall package test. Because we are working outside `$GOPATH` and also outside any module, the `go` command knows no import path for the current directory and makes up a fake one based on the directory name: `_/home/gopher/hello`.

Let’s make the current directory the root of a module by using `go mod init` and then try `go test` again:
    
    
    $ go mod init example.com/hello
    go: creating new go.mod: module example.com/hello
    $ go test
    PASS
    ok      example.com/hello   0.020s
    $
    

Congratulations! You’ve written and tested your first module.

The `go mod init` command wrote a `go.mod` file:
    
    
    $ cat go.mod
    module example.com/hello
    
    go 1.12
    $
    

The `go.mod` file only appears in the root of the module. Packages in subdirectories have import paths consisting of the module path plus the path to the subdirectory. For example, if we created a subdirectory `world`, we would not need to (nor want to) run `go mod init` there. The package would automatically be recognized as part of the `example.com/hello` module, with import path `example.com/hello/world`.

## Adding a dependency

The primary motivation for Go modules was to improve the experience of using (that is, adding a dependency on) code written by other developers.

Let’s update our `hello.go` to import `rsc.io/quote` and use it to implement `Hello`:
    
    
    package hello
    
    import "rsc.io/quote"
    
    func Hello() string {
        return quote.Hello()
    }
    

Now let’s run the test again:
    
    
    $ go test
    go: finding rsc.io/quote v1.5.2
    go: downloading rsc.io/quote v1.5.2
    go: extracting rsc.io/quote v1.5.2
    go: finding rsc.io/sampler v1.3.0
    go: finding golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
    go: downloading rsc.io/sampler v1.3.0
    go: extracting rsc.io/sampler v1.3.0
    go: downloading golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
    go: extracting golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
    PASS
    ok      example.com/hello   0.023s
    $
    

The `go` command resolves imports by using the specific dependency module versions listed in `go.mod`. When it encounters an `import` of a package not provided by any module in `go.mod`, the `go` command automatically looks up the module containing that package and adds it to `go.mod`, using the latest version. (“Latest” is defined as the latest tagged stable (non-[prerelease](https://semver.org/#spec-item-9)) version, or else the latest tagged prerelease version, or else the latest untagged version.) In our example, `go test` resolved the new import `rsc.io/quote` to the module `rsc.io/quote v1.5.2`. It also downloaded two dependencies used by `rsc.io/quote`, namely `rsc.io/sampler` and `golang.org/x/text`. Only direct dependencies are recorded in the `go.mod` file:
    
    
    $ cat go.mod
    module example.com/hello
    
    go 1.12
    
    require rsc.io/quote v1.5.2
    $
    

A second `go test` command will not repeat this work, since the `go.mod` is now up-to-date and the downloaded modules are cached locally (in `$GOPATH/pkg/mod`):
    
    
    $ go test
    PASS
    ok      example.com/hello   0.020s
    $
    

Note that while the `go` command makes adding a new dependency quick and easy, it is not without cost. Your module now literally _depends_ on the new dependency in critical areas such as correctness, security, and proper licensing, just to name a few. For more considerations, see Russ Cox’s blog post, “[Our Software Dependency Problem](https://research.swtch.com/deps).”

As we saw above, adding one direct dependency often brings in other indirect dependencies too. The command `go list -m all` lists the current module and all its dependencies:
    
    
    $ go list -m all
    example.com/hello
    golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
    rsc.io/quote v1.5.2
    rsc.io/sampler v1.3.0
    $
    

In the `go list` output, the current module, also known as the _main module_ , is always the first line, followed by dependencies sorted by module path.

The `golang.org/x/text` version `v0.0.0-20170915032832-14c0d48ead0c` is an example of a [pseudo-version](/ref/mod#pseudo-versions), which is the `go` command’s version syntax for a specific untagged commit.

In addition to `go.mod`, the `go` command maintains a file named `go.sum` containing the expected [cryptographic hashes](/cmd/go/#hdr-Module_downloading_and_verification) of the content of specific module versions:
    
    
    $ cat go.sum
    golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZO...
    golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:Nq...
    rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3...
    rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPX...
    rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/Q...
    rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9...
    $
    

The `go` command uses the `go.sum` file to ensure that future downloads of these modules retrieve the same bits as the first download, to ensure the modules your project depends on do not change unexpectedly, whether for malicious, accidental, or other reasons. Both `go.mod` and `go.sum` should be checked into version control.

## Upgrading dependencies

With Go modules, versions are referenced with semantic version tags. A semantic version has three parts: major, minor, and patch. For example, for `v0.1.2`, the major version is 0, the minor version is 1, and the patch version is 2. Let’s walk through a couple minor version upgrades. In the next section, we’ll consider a major version upgrade.

From the output of `go list -m all`, we can see we’re using an untagged version of `golang.org/x/text`. Let’s upgrade to the latest tagged version and test that everything still works:
    
    
    $ go get golang.org/x/text
    go: finding golang.org/x/text v0.3.0
    go: downloading golang.org/x/text v0.3.0
    go: extracting golang.org/x/text v0.3.0
    $ go test
    PASS
    ok      example.com/hello   0.013s
    $
    

Woohoo! Everything passes. Let’s take another look at `go list -m all` and the `go.mod` file:
    
    
    $ go list -m all
    example.com/hello
    golang.org/x/text v0.3.0
    rsc.io/quote v1.5.2
    rsc.io/sampler v1.3.0
    $ cat go.mod
    module example.com/hello
    
    go 1.12
    
    require (
        golang.org/x/text v0.3.0 // indirect
        rsc.io/quote v1.5.2
    )
    $
    

The `golang.org/x/text` package has been upgraded to the latest tagged version (`v0.3.0`). The `go.mod` file has been updated to specify `v0.3.0` too. The `indirect` comment indicates a dependency is not used directly by this module, only indirectly by other module dependencies. See `go help modules` for details.

Now let’s try upgrading the `rsc.io/sampler` minor version. Start the same way, by running `go get` and running tests:
    
    
    $ go get rsc.io/sampler
    go: finding rsc.io/sampler v1.99.99
    go: downloading rsc.io/sampler v1.99.99
    go: extracting rsc.io/sampler v1.99.99
    $ go test
    --- FAIL: TestHello (0.00s)
        hello_test.go:8: Hello() = "99 bottles of beer on the wall, 99 bottles of beer, ...", want "Hello, world."
    FAIL
    exit status 1
    FAIL    example.com/hello   0.014s
    $
    

Uh, oh! The test failure shows that the latest version of `rsc.io/sampler` is incompatible with our usage. Let’s list the available tagged versions of that module:
    
    
    $ go list -m -versions rsc.io/sampler
    rsc.io/sampler v1.0.0 v1.2.0 v1.2.1 v1.3.0 v1.3.1 v1.99.99
    $
    

We had been using v1.3.0; v1.99.99 is clearly no good. Maybe we can try using v1.3.1 instead:
    
    
    $ go get rsc.io/sampler@v1.3.1
    go: finding rsc.io/sampler v1.3.1
    go: downloading rsc.io/sampler v1.3.1
    go: extracting rsc.io/sampler v1.3.1
    $ go test
    PASS
    ok      example.com/hello   0.022s
    $
    

Note the explicit `@v1.3.1` in the `go get` argument. In general each argument passed to `go get` can take an explicit version; the default is `@latest`, which resolves to the latest version as defined earlier.

## Adding a dependency on a new major version

Let’s add a new function to our package: `func Proverb` returns a Go concurrency proverb, by calling `quote.Concurrency`, which is provided by the module `rsc.io/quote/v3`. First we update `hello.go` to add the new function:
    
    
    package hello
    
    import (
        "rsc.io/quote"
        quoteV3 "rsc.io/quote/v3"
    )
    
    func Hello() string {
        return quote.Hello()
    }
    
    func Proverb() string {
        return quoteV3.Concurrency()
    }
    

Then we add a test to `hello_test.go`:
    
    
    func TestProverb(t *testing.T) {
        want := "Concurrency is not parallelism."
        if got := Proverb(); got != want {
            t.Errorf("Proverb() = %q, want %q", got, want)
        }
    }
    

Then we can test our code:
    
    
    $ go test
    go: finding rsc.io/quote/v3 v3.1.0
    go: downloading rsc.io/quote/v3 v3.1.0
    go: extracting rsc.io/quote/v3 v3.1.0
    PASS
    ok      example.com/hello   0.024s
    $
    

Note that our module now depends on both `rsc.io/quote` and `rsc.io/quote/v3`:
    
    
    $ go list -m rsc.io/q...
    rsc.io/quote v1.5.2
    rsc.io/quote/v3 v3.1.0
    $
    

Each different major version (`v1`, `v2`, and so on) of a Go module uses a different module path: starting at `v2`, the path must end in the major version. In the example, `v3` of `rsc.io/quote` is no longer `rsc.io/quote`: instead, it is identified by the module path `rsc.io/quote/v3`. This convention is called [semantic import versioning](https://research.swtch.com/vgo-import), and it gives incompatible packages (those with different major versions) different names. In contrast, `v1.6.0` of `rsc.io/quote` should be backwards-compatible with `v1.5.2`, so it reuses the name `rsc.io/quote`. (In the previous section, `rsc.io/sampler` `v1.99.99` _should_ have been backwards-compatible with `rsc.io/sampler` `v1.3.0`, but bugs or incorrect client assumptions about module behavior can both happen.)

The `go` command allows a build to include at most one version of any particular module path, meaning at most one of each major version: one `rsc.io/quote`, one `rsc.io/quote/v2`, one `rsc.io/quote/v3`, and so on. This gives module authors a clear rule about possible duplication of a single module path: it is impossible for a program to build with both `rsc.io/quote v1.5.2` and `rsc.io/quote v1.6.0`. At the same time, allowing different major versions of a module (because they have different paths) gives module consumers the ability to upgrade to a new major version incrementally. In this example, we wanted to use `quote.Concurrency` from `rsc/quote/v3 v3.1.0` but are not yet ready to migrate our uses of `rsc.io/quote v1.5.2`. The ability to migrate incrementally is especially important in a large program or codebase.

## Upgrading a dependency to a new major version

Let’s complete our conversion from using `rsc.io/quote` to using only `rsc.io/quote/v3`. Because of the major version change, we should expect that some APIs may have been removed, renamed, or otherwise changed in incompatible ways. Reading the docs, we can see that `Hello` has become `HelloV3`:
    
    
    $ go doc rsc.io/quote/v3
    package quote // import "rsc.io/quote/v3"
    
    Package quote collects pithy sayings.
    
    func Concurrency() string
    func GlassV3() string
    func GoV3() string
    func HelloV3() string
    func OptV3() string
    $
    

We can update our use of `quote.Hello()` in `hello.go` to use `quoteV3.HelloV3()`:
    
    
    package hello
    
    import quoteV3 "rsc.io/quote/v3"
    
    func Hello() string {
        return quoteV3.HelloV3()
    }
    
    func Proverb() string {
        return quoteV3.Concurrency()
    }
    

And then at this point, there’s no need for the renamed import anymore, so we can undo that:
    
    
    package hello
    
    import "rsc.io/quote/v3"
    
    func Hello() string {
        return quote.HelloV3()
    }
    
    func Proverb() string {
        return quote.Concurrency()
    }
    

Let’s re-run the tests to make sure everything is working:
    
    
    $ go test
    PASS
    ok      example.com/hello       0.014s
    

## Removing unused dependencies

We’ve removed all our uses of `rsc.io/quote`, but it still shows up in `go list -m all` and in our `go.mod` file:
    
    
    $ go list -m all
    example.com/hello
    golang.org/x/text v0.3.0
    rsc.io/quote v1.5.2
    rsc.io/quote/v3 v3.1.0
    rsc.io/sampler v1.3.1
    $ cat go.mod
    module example.com/hello
    
    go 1.12
    
    require (
        golang.org/x/text v0.3.0 // indirect
        rsc.io/quote v1.5.2
        rsc.io/quote/v3 v3.0.0
        rsc.io/sampler v1.3.1 // indirect
    )
    $
    

Why? Because building a single package, like with `go build` or `go test`, can easily tell when something is missing and needs to be added, but not when something can safely be removed. Removing a dependency can only be done after checking all packages in a module, and all possible build tag combinations for those packages. An ordinary build command does not load this information, and so it cannot safely remove dependencies.

The `go mod tidy` command cleans up these unused dependencies:
    
    
    $ go mod tidy
    $ go list -m all
    example.com/hello
    golang.org/x/text v0.3.0
    rsc.io/quote/v3 v3.1.0
    rsc.io/sampler v1.3.1
    $ cat go.mod
    module example.com/hello
    
    go 1.12
    
    require (
        golang.org/x/text v0.3.0 // indirect
        rsc.io/quote/v3 v3.1.0
        rsc.io/sampler v1.3.1 // indirect
    )
    
    $ go test
    PASS
    ok      example.com/hello   0.020s
    $
    

## Conclusion

Go modules are the future of dependency management in Go. Module functionality is now available in all supported Go versions (that is, in Go 1.11 and Go 1.12).

This post introduced these workflows using Go modules:

  * `go mod init` creates a new module, initializing the `go.mod` file that describes it.
  * `go build`, `go test`, and other package-building commands add new dependencies to `go.mod` as needed.
  * `go list -m all` prints the current module’s dependencies.
  * `go get` changes the required version of a dependency (or adds a new dependency).
  * `go mod tidy` removes unused dependencies.



We encourage you to start using modules in your local development and to add `go.mod` and `go.sum` files to your projects. To provide feedback and help shape the future of dependency management in Go, please send us [bug reports](/issue/new) or [experience reports](/wiki/ExperienceReports).

Thanks for all your feedback and help improving modules.

**Next article:**[Debugging what you deploy in Go 1.12](/blog/debug-opt)  
**Previous article:**[The New Go Developer Network](/blog/go-developer-network)  
**[Blog Index](/blog/all)**