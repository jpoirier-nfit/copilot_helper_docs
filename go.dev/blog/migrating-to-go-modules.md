---
source_url: https://go.dev/blog/migrating-to-go-modules
title: Migrating to Go Modules - The Go Programming Language
crawl_date: 2025-07-25T12:08:27.452312
watsonmd_version: 0.1.0
---

# [The Go Blog](/blog/)

# Migrating to Go Modules

Jean Barkhuysen  
21 August 2019 

## Introduction

This post is part 2 in a series.

  * Part 1 — [Using Go Modules](/blog/using-go-modules)
  * **Part 2 — Migrating To Go Modules** (this post)
  * Part 3 — [Publishing Go Modules](/blog/publishing-go-modules)
  * Part 4 — [Go Modules: v2 and Beyond](/blog/v2-go-modules)
  * Part 5 — [Keeping Your Modules Compatible](/blog/module-compatibility)



**Note:** For documentation, see [Managing dependencies](/doc/modules/managing-dependencies) and [Developing and publishing modules](/doc/modules/developing).

Go projects use a wide variety of dependency management strategies. [Vendoring](/cmd/go/#hdr-Vendor_Directories) tools such as [dep](https://github.com/golang/dep) and [glide](https://github.com/Masterminds/glide) are popular, but they have wide differences in behavior and don’t always work well together. Some projects store their entire GOPATH directory in a single Git repository. Others simply rely on `go get` and expect fairly recent versions of dependencies to be installed in GOPATH.

Go’s module system, introduced in Go 1.11, provides an official dependency management solution built into the `go` command. This article describes tools and techniques for converting a project to modules.

Please note: if your project is already tagged at v2.0.0 or higher, you will need to update your module path when you add a `go.mod` file. We’ll explain how to do that without breaking your users in a future article focused on v2 and beyond.

## Migrating to Go modules in your project

A project might be in one of three states when beginning the transition to Go modules:

  * A brand new Go project.
  * An established Go project with a non-modules dependency manager.
  * An established Go project without any dependency manager.



The first case is covered in [Using Go Modules](/blog/using-go-modules); we’ll address the latter two in this post.

## With a dependency manager

To convert a project that already uses a dependency management tool, run the following commands:
    
    
    $ git clone https://github.com/my/project
    [...]
    $ cd project
    $ cat Godeps/Godeps.json
    {
        "ImportPath": "github.com/my/project",
        "GoVersion": "go1.12",
        "GodepVersion": "v80",
        "Deps": [
            {
                "ImportPath": "rsc.io/binaryregexp",
                "Comment": "v0.2.0-1-g545cabd",
                "Rev": "545cabda89ca36b48b8e681a30d9d769a30b3074"
            },
            {
                "ImportPath": "rsc.io/binaryregexp/syntax",
                "Comment": "v0.2.0-1-g545cabd",
                "Rev": "545cabda89ca36b48b8e681a30d9d769a30b3074"
            }
        ]
    }
    $ go mod init github.com/my/project
    go: creating new go.mod: module github.com/my/project
    go: copying requirements from Godeps/Godeps.json
    $ cat go.mod
    module github.com/my/project
    
    go 1.12
    
    require rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
    $
    

`go mod init` creates a new go.mod file and automatically imports dependencies from `Godeps.json`, `Gopkg.lock`, or a number of [other supported formats](https://go.googlesource.com/go/+/362625209b6cd2bc059b6b0a67712ddebab312d9/src/cmd/go/internal/modconv/modconv.go#9). The argument to `go mod init` is the module path, the location where the module may be found.

This is a good time to pause and run `go build ./...` and `go test ./...` before continuing. Later steps may modify your `go.mod` file, so if you prefer to take an iterative approach, this is the closest your `go.mod` file will be to your pre-modules dependency specification.
    
    
    $ go mod tidy
    go: downloading rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
    go: extracting rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
    $ cat go.sum
    rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca h1:FKXXXJ6G2bFoVe7hX3kEX6Izxw5ZKRH57DFBJmHCbkU=
    rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca/go.mod h1:qTv7/COck+e2FymRvadv62gMdZztPaShugOCi3I+8D8=
    $
    

`go mod tidy` finds all the packages transitively imported by packages in your module. It adds new module requirements for packages not provided by any known module, and it removes requirements on modules that don’t provide any imported packages. If a module provides packages that are only imported by projects that haven’t migrated to modules yet, the module requirement will be marked with an `// indirect` comment. It is always good practice to run `go mod tidy` before committing a `go.mod` file to version control.

Let’s finish by making sure the code builds and tests pass:
    
    
    $ go build ./...
    $ go test ./...
    [...]
    $
    

Note that other dependency managers may specify dependencies at the level of individual packages or entire repositories (not modules), and generally do not recognize the requirements specified in the `go.mod` files of dependencies. Consequently, you may not get exactly the same version of every package as before, and there’s some risk of upgrading past breaking changes. Therefore, it’s important to follow the above commands with an audit of the resulting dependencies. To do so, run
    
    
    $ go list -m all
    go: finding rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
    github.com/my/project
    rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
    $
    

and compare the resulting versions with your old dependency management file to ensure that the selected versions are appropriate. If you find a version that wasn’t what you wanted, you can find out why using `go mod why -m` and/or `go mod graph`, and upgrade or downgrade to the correct version using `go get`. (If the version you request is older than the version that was previously selected, `go get` will downgrade other dependencies as needed to maintain compatibility.) For example,
    
    
    $ go mod why -m rsc.io/binaryregexp
    [...]
    $ go mod graph | grep rsc.io/binaryregexp
    [...]
    $ go get rsc.io/binaryregexp@v0.2.0
    $
    

## Without a dependency manager

For a Go project without a dependency management system, start by creating a `go.mod` file:
    
    
    $ git clone https://go.googlesource.com/blog
    [...]
    $ cd blog
    $ go mod init golang.org/x/blog
    go: creating new go.mod: module golang.org/x/blog
    $ cat go.mod
    module golang.org/x/blog
    
    go 1.12
    $
    

Without a configuration file from a previous dependency manager, `go mod init` will create a `go.mod` file with only the `module` and `go` directives. In this example, we set the module path to `golang.org/x/blog` because that is its [custom import path](/cmd/go/#hdr-Remote_import_paths). Users may import packages with this path, and we must be careful not to change it.

The `module` directive declares the module path, and the `go` directive declares the expected version of the Go language used to compile the code within the module.

Next, run `go mod tidy` to add the module’s dependencies:
    
    
    $ go mod tidy
    go: finding golang.org/x/website latest
    go: finding gopkg.in/tomb.v2 latest
    go: finding golang.org/x/net latest
    go: finding golang.org/x/tools latest
    go: downloading github.com/gorilla/context v1.1.1
    go: downloading golang.org/x/tools v0.0.0-20190813214729-9dba7caff850
    go: downloading golang.org/x/net v0.0.0-20190813141303-74dc4d7220e7
    go: extracting github.com/gorilla/context v1.1.1
    go: extracting golang.org/x/net v0.0.0-20190813141303-74dc4d7220e7
    go: downloading gopkg.in/tomb.v2 v2.0.0-20161208151619-d5d1b5820637
    go: extracting gopkg.in/tomb.v2 v2.0.0-20161208151619-d5d1b5820637
    go: extracting golang.org/x/tools v0.0.0-20190813214729-9dba7caff850
    go: downloading golang.org/x/website v0.0.0-20190809153340-86a7442ada7c
    go: extracting golang.org/x/website v0.0.0-20190809153340-86a7442ada7c
    $ cat go.mod
    module golang.org/x/blog
    
    go 1.12
    
    require (
        github.com/gorilla/context v1.1.1
        golang.org/x/net v0.0.0-20190813141303-74dc4d7220e7
        golang.org/x/text v0.3.2
        golang.org/x/tools v0.0.0-20190813214729-9dba7caff850
        golang.org/x/website v0.0.0-20190809153340-86a7442ada7c
        gopkg.in/tomb.v2 v2.0.0-20161208151619-d5d1b5820637
    )
    $ cat go.sum
    cloud.google.com/go v0.26.0/go.mod h1:aQUYkXzVsufM+DwF1aE+0xfcU+56JwCaLick0ClmMTw=
    cloud.google.com/go v0.34.0/go.mod h1:aQUYkXzVsufM+DwF1aE+0xfcU+56JwCaLick0ClmMTw=
    git.apache.org/thrift.git v0.0.0-20180902110319-2566ecd5d999/go.mod h1:fPE2ZNJGynbRyZ4dJvy6G277gSllfV2HJqblrnkyeyg=
    git.apache.org/thrift.git v0.0.0-20181218151757-9b75e4fe745a/go.mod h1:fPE2ZNJGynbRyZ4dJvy6G277gSllfV2HJqblrnkyeyg=
    github.com/beorn7/perks v0.0.0-20180321164747-3a771d992973/go.mod h1:Dwedo/Wpr24TaqPxmxbtue+5NUziq4I4S80YR8gNf3Q=
    [...]
    $
    

`go mod tidy` added module requirements for all the packages transitively imported by packages in your module and built a `go.sum` with checksums for each library at a specific version. Let’s finish by making sure the code still builds and tests still pass:
    
    
    $ go build ./...
    $ go test ./...
    ok      golang.org/x/blog   0.335s
    ?       golang.org/x/blog/content/appengine [no test files]
    ok      golang.org/x/blog/content/cover 0.040s
    ?       golang.org/x/blog/content/h2push/server [no test files]
    ?       golang.org/x/blog/content/survey2016    [no test files]
    ?       golang.org/x/blog/content/survey2017    [no test files]
    ?       golang.org/x/blog/support/racy  [no test files]
    $
    

Note that when `go mod tidy` adds a requirement, it adds the latest version of the module. If your `GOPATH` included an older version of a dependency that subsequently published a breaking change, you may see errors in `go mod tidy`, `go build`, or `go test`. If this happens, try downgrading to an older version with `go get` (for example, `go get github.com/broken/module@v1.1.0`), or take the time to make your module compatible with the latest version of each dependency.

### Tests in module mode

Some tests may need tweaks after migrating to Go modules.

If a test needs to write files in the package directory, it may fail when the package directory is in the module cache, which is read-only. In particular, this may cause `go test all` to fail. The test should copy files it needs to write to a temporary directory instead.

If a test relies on relative paths (`../package-in-another-module`) to locate and read files in another package, it will fail if the package is in another module, which will be located in a versioned subdirectory of the module cache or a path specified in a `replace` directive. If this is the case, you may need to copy the test inputs into your module, or convert the test inputs from raw files to data embedded in `.go` source files.

If a test expects `go` commands within the test to run in GOPATH mode, it may fail. If this is the case, you may need to add a `go.mod` file to the source tree to be tested, or set `GO111MODULE=off` explicitly.

## Publishing a release

Finally, you should tag and publish a release version for your new module. This is optional if you haven’t released any versions yet, but without an official release, downstream users will depend on specific commits using [pseudo-versions](/cmd/go/#hdr-Pseudo_versions), which may be more difficult to support.
    
    
    $ git tag v1.2.0
    $ git push origin v1.2.0
    

Your new `go.mod` file defines a canonical import path for your module and adds new minimum version requirements. If your users are already using the correct import path, and your dependencies haven’t made breaking changes, then adding the `go.mod` file is backwards-compatible — but it’s a significant change, and may expose existing problems. If you have existing version tags, you should increment the [minor version](https://semver.org/#spec-item-7). See [Publishing Go Modules](/blog/publishing-go-modules) to learn how to increment and publish versions.

## Imports and canonical module paths

Each module declares its module path in its `go.mod` file. Each `import` statement that refers to a package within the module must have the module path as a prefix of the package path. However, the `go` command may encounter a repository containing the module through many different [remote import paths](/cmd/go/#hdr-Remote_import_paths). For example, both `golang.org/x/lint` and `github.com/golang/lint` resolve to repositories containing the code hosted at [go.googlesource.com/lint](https://go.googlesource.com/lint). The [`go.mod` file](https://go.googlesource.com/lint/+/refs/heads/master/go.mod) contained in that repository declares its path to be `golang.org/x/lint`, so only that path corresponds to a valid module.

Go 1.4 provided a mechanism for declaring canonical import paths using [`// import` comments](/cmd/go/#hdr-Import_path_checking), but package authors did not always provide them. As a result, code written prior to modules may have used a non-canonical import path for a module without surfacing an error for the mismatch. When using modules, the import path must match the canonical module path, so you may need to update `import` statements: for example, you may need to change `import "github.com/golang/lint"` to `import "golang.org/x/lint"`.

Another scenario in which a module’s canonical path may differ from its repository path occurs for Go modules at major version 2 or higher. A Go module with a major version above 1 must include a major-version suffix in its module path: for example, version `v2.0.0` must have the suffix `/v2`. However, `import` statements may have referred to the packages within the module _without_ that suffix. For example, non-module users of `github.com/russross/blackfriday/v2` at `v2.0.1` may have imported it as `github.com/russross/blackfriday` instead, and will need to update the import path to include the `/v2` suffix.

## Conclusion

Converting to Go modules should be a straightforward process for most users. Occasional issues may arise due to non-canonical import paths or breaking changes within a dependency. Future posts will explore [publishing new versions](/blog/publishing-go-modules), v2 and beyond, and ways to debug strange situations.

To provide feedback and help shape the future of dependency management in Go, please send us [bug reports](/issue/new) or [experience reports](/wiki/ExperienceReports).

Thanks for all your feedback and help improving modules.

**Next article:**[Module Mirror and Checksum Database Launched](/blog/module-mirror-launch)  
**Previous article:**[Contributors Summit 2019](/blog/contributors-summit-2019)  
**[Blog Index](/blog/all)**