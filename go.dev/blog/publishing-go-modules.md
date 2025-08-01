---
source_url: https://go.dev/blog/publishing-go-modules
title: Publishing Go Modules - The Go Programming Language
crawl_date: 2025-07-25T12:08:27.461800
watsonmd_version: 0.1.0
---

# [The Go Blog](/blog/)

# Publishing Go Modules

Tyler Bui-Palsulich  
26 September 2019 

## Introduction

This post is part 3 in a series.

  * Part 1 — [Using Go Modules](/blog/using-go-modules)
  * Part 2 — [Migrating To Go Modules](/blog/migrating-to-go-modules)
  * **Part 3 — Publishing Go Modules** (this post)
  * Part 4 — [Go Modules: v2 and Beyond](/blog/v2-go-modules)
  * Part 5 — [Keeping Your Modules Compatible](/blog/module-compatibility)



**Note:** For documentation on developing modules, see [Developing and publishing modules](/doc/modules/developing).

This post discusses how to write and publish modules so other modules can depend on them.

Please note: this post covers development up to and including `v1`. If you are interested in `v2`, please see [Go Modules: v2 and Beyond](/blog/v2-go-modules).

This post uses [Git](https://git-scm.com/) in examples. [Mercurial](https://www.mercurial-scm.org/), [Bazaar](http://wiki.bazaar.canonical.com/), and others are supported as well.

## Project setup

For this post, you’ll need an existing project to use as an example. So, start with the files from the end of the [Using Go Modules](/blog/using-go-modules) article:
    
    
    $ cat go.mod
    module example.com/hello
    
    go 1.12
    
    require rsc.io/quote/v3 v3.1.0
    
    $ cat go.sum
    golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
    golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
    rsc.io/quote/v3 v3.1.0 h1:9JKUTTIUgS6kzR9mK1YuGKv6Nl+DijDNIc0ghT58FaY=
    rsc.io/quote/v3 v3.1.0/go.mod h1:yEA65RcK8LyAZtP9Kv3t0HmxON59tX3rD+tICJqUlj0=
    rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
    rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9JXDnKaTXpA=
    
    $ cat hello.go
    package hello
    
    import "rsc.io/quote/v3"
    
    func Hello() string {
        return quote.HelloV3()
    }
    
    func Proverb() string {
        return quote.Concurrency()
    }
    
    $ cat hello_test.go
    package hello
    
    import (
        "testing"
    )
    
    func TestHello(t *testing.T) {
        want := "Hello, world."
        if got := Hello(); got != want {
            t.Errorf("Hello() = %q, want %q", got, want)
        }
    }
    
    func TestProverb(t *testing.T) {
        want := "Concurrency is not parallelism."
        if got := Proverb(); got != want {
            t.Errorf("Proverb() = %q, want %q", got, want)
        }
    }
    
    $
    

Next, create a new `git` repository and add an initial commit. If you’re publishing your own project, be sure to include a `LICENSE` file. Change to the directory containing the `go.mod` then create the repo:
    
    
    $ git init
    $ git add LICENSE go.mod go.sum hello.go hello_test.go
    $ git commit -m "hello: initial commit"
    $
    

## Semantic versions and modules

Every required module in a `go.mod` has a [semantic version](https://semver.org), the minimum version of that dependency to use to build the module.

A semantic version has the form `vMAJOR.MINOR.PATCH`.

  * Increment the `MAJOR` version when you make a [backwards incompatible](/doc/go1compat) change to the public API of your module. This should only be done when absolutely necessary.
  * Increment the `MINOR` version when you make a backwards compatible change to the API, like changing dependencies or adding a new function, method, struct field, or type.
  * Increment the `PATCH` version after making minor changes that don’t affect your module’s public API or dependencies, like fixing a bug.



You can specify pre-release versions by appending a hyphen and dot separated identifiers (for example, `v1.0.1-alpha` or `v2.2.2-beta.2`). Normal releases are preferred by the `go` command over pre-release versions, so users must ask for pre-release versions explicitly (for example, `go get example.com/hello@v1.0.1-alpha`) if your module has any normal releases.

`v0` major versions and pre-release versions do not guarantee backwards compatibility. They let you refine your API before making stability commitments to your users. However, `v1` major versions and beyond require backwards compatibility within that major version.

The version referenced in a `go.mod` may be an explicit release tagged in the repository (for example, `v1.5.2`), or it may be a [pseudo-version](/ref/mod#pseudo-versions) based on a specific commit (for example, `v0.0.0-20170915032832-14c0d48ead0c`). Pseudo-versions are a special type of pre-release version. Pseudo-versions are useful when a user needs to depend on a project that has not published any semantic version tags, or develop against a commit that hasn’t been tagged yet, but users should not assume that pseudo-versions provide a stable or well-tested API. Tagging your modules with explicit versions signals to your users that specific versions are fully tested and ready to use.

Once you start tagging your repo with versions, it’s important to keep tagging new releases as you develop your module. When users request a new version of your module (with `go get -u` or `go get example.com/hello`), the `go` command will choose the greatest semantic release version available, even if that version is several years old and many changes behind the primary branch. Continuing to tag new releases will make your ongoing improvements available to your users.

Do not delete version tags from your repo. If you find a bug or a security issue with a version, release a new version. If people depend on a version that you have deleted, their builds may fail. Similarly, once you release a version, do not change or overwrite it. The [module mirror and checksum database](/blog/module-mirror-launch) store modules, their versions, and signed cryptographic hashes to ensure that the build of a given version remains reproducible over time.

## v0: the initial, unstable version

Let’s tag the module with a `v0` semantic version. A `v0` version does not make any stability guarantees, so nearly all projects should start with `v0` as they refine their public API.

Tagging a new version has a few steps:

  1. Run `go mod tidy`, which removes any dependencies the module might have accumulated that are no longer necessary.

  2. Run `go test ./...` a final time to make sure everything is working.

  3. Tag the project with a new version using [`git tag`](https://git-scm.com/docs/git-tag).

  4. Push the new tag to the origin repository.



    
    
    $ go mod tidy
    $ go test ./...
    ok      example.com/hello       0.015s
    $ git add go.mod go.sum hello.go hello_test.go
    $ git commit -m "hello: changes for v0.1.0"
    $ git tag v0.1.0
    $ git push origin v0.1.0
    $
    

Now other projects can depend on `v0.1.0` of `example.com/hello`. For your own module, you can run `go list -m example.com/hello@v0.1.0` to confirm the latest version is available (this example module does not exist, so no versions are available). If you don’t see the latest version immediately and you’re using the Go module proxy (the default since Go 1.13), try again in a few minutes to give the proxy time to load the new version.

If you add to the public API, make a breaking change to a `v0` module, or upgrade the minor or version of one of your dependencies, increment the `MINOR` version for your next release. For example, the next release after `v0.1.0` would be `v0.2.0`.

If you fix a bug in an existing version, increment the `PATCH` version. For example, the next release after `v0.1.0` would be `v0.1.1`.

## v1: the first stable version

Once you are absolutely sure your module’s API is stable, you can release `v1.0.0`. A `v1` major version communicates to users that no incompatible changes will be made to the module’s API. They can upgrade to new `v1` minor and patch releases, and their code should not break. Function and method signatures will not change, exported types will not be removed, and so on. If there are changes to the API, they will be backwards compatible (for example, adding a new field to a struct) and will be included in a new minor release. If there are bug fixes (for example, a security fix), they will be included in a patch release (or as part of a minor release).

Sometimes, maintaining backwards compatibility can lead to awkward APIs. That’s OK. An imperfect API is better than breaking users’ existing code.

The standard library’s `strings` package is a prime example of maintaining backwards compatibility at the cost of API consistency.

  * [`Split`](https://godoc.org/strings#Split) slices a string into all substrings separated by a separator and returns a slice of the substrings between those separators.
  * [`SplitN`](https://godoc.org/strings#SplitN) can be used to control the number of substrings to return.



However, [`Replace`](https://godoc.org/strings#Replace) took a count of how many instances of the string to replace from the beginning (unlike `Split`).

Given `Split` and `SplitN`, you would expect functions like `Replace` and `ReplaceN`. But, we couldn’t change the existing `Replace` without breaking callers, which we promised not to do. So, in Go 1.12, we added a new function, [`ReplaceAll`](https://godoc.org/strings#ReplaceAll). The resulting API is a little odd, since `Split` and `Replace` behave differently, but that inconsistency is better than a breaking change.

Let’s say you’re happy with the API of `example.com/hello` and you want to release `v1` as the first stable version.

Tagging `v1` uses the same process as tagging a `v0` version: run `go mod tidy` and `go test ./...`, tag the version, and push the tag to the origin repository:
    
    
    $ go mod tidy
    $ go test ./...
    ok      example.com/hello       0.015s
    $ git add go.mod go.sum hello.go hello_test.go
    $ git commit -m "hello: changes for v1.0.0"
    $ git tag v1.0.0
    $ git push origin v1.0.0
    $
    

At this point, the `v1` API of `example.com/hello` is solidified. This communicates to everyone that our API is stable and they should feel comfortable using it.

## Conclusion

This post walked through the process of tagging a module with semantic versions and when to release `v1`. A future post will cover how to maintain and publish modules at `v2` and beyond.

To provide feedback and help shape the future of dependency management in Go, please send us [bug reports](/issue/new) or [experience reports](/wiki/ExperienceReports).

Thanks for all your feedback and help improving Go modules.

**Next article:**[Working with Errors in Go 1.13](/blog/go1.13-errors)  
**Previous article:**[Go 1.13 is released](/blog/go1.13)  
**[Blog Index](/blog/all)**