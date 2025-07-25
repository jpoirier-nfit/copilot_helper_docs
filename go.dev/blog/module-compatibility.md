---
source_url: https://go.dev/blog/module-compatibility
title: Keeping Your Modules Compatible - The Go Programming Language
crawl_date: 2025-07-25T12:08:27.479218
watsonmd_version: 0.1.0
---

# [The Go Blog](/blog/)

# Keeping Your Modules Compatible

Jean Barkhuysen and Jonathan Amsterdam  
7 July 2020 

## Introduction

This post is part 5 in a series.

  * Part 1 — [Using Go Modules](/blog/using-go-modules)
  * Part 2 — [Migrating To Go Modules](/blog/migrating-to-go-modules)
  * Part 3 — [Publishing Go Modules](/blog/publishing-go-modules)
  * Part 4 — [Go Modules: v2 and Beyond](/blog/v2-go-modules)
  * **Part 5 — Keeping Your Modules Compatible** (this post)



**Note:** For documentation on developing modules, see [Developing and publishing modules](/doc/modules/developing).

Your modules will evolve over time as you add new features, change behaviors, and reconsider parts of the module’s public surface. As discussed in [Go Modules: v2 and Beyond](/blog/v2-go-modules), breaking changes to a v1+ module must happen as part of a major version bump (or by adopting a new module path).

However, releasing a new major version is hard on your users. They have to find the new version, learn a new API, and change their code. And some users may never update, meaning you have to maintain two versions for your code forever. So it is usually better to change your existing package in a compatible way.

In this post, we’ll explore some techniques for introducing non-breaking changes. The common theme is: add, don’t change or remove. We’ll also talk about how to design your API for compatibility from the outset.

## Adding to a function

Often, breaking changes come in the form of new arguments to a function. We’ll describe some ways to deal with this sort of change, but first let’s look at a technique that doesn’t work.

When adding new arguments with sensible defaults, it’s tempting to add them as a variadic parameter. To extend the function
    
    
    func Run(name string)
    

with an additional `size` argument which defaults to zero, one might propose
    
    
    func Run(name string, size ...int)
    

on the grounds that all existing call sites will continue to work. While that is true, other uses of `Run` could break, like this one:
    
    
    package mypkg
    var runner func(string) = yourpkg.Run
    

The original `Run` function works here because its type is `func(string)`, but the new `Run` function’s type is `func(string, ...int)`, so the assignment fails at compile time.

This example illustrates that call compatibility is not enough for backward compatibility. There is, in fact, no backward-compatible change you can make to a function’s signature.

Instead of changing a function’s signature, add a new function. As an example, after the `context` package was introduced, it became common practice to pass a `context.Context` as the first argument to a function. However, stable APIs could not change an exported function to accept a `context.Context` because it would break all uses of that function.

Instead, new functions were added. For example, the `database/sql` package’s `Query` method’s signature was (and still is)
    
    
    func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
    

When the `context` package was created, the Go team added a new method to `database/sql`:
    
    
    func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
    

To avoid copying code, the old method calls the new one:
    
    
    func (db *DB) Query(query string, args ...interface{}) (*Rows, error) {
        return db.QueryContext(context.Background(), query, args...)
    }
    

Adding a method allows users to migrate to the new API at their own pace. Since the methods read similarly and sort together, and `Context` is in the name of the new method, this extension of the `database/sql` API did not degrade readability or comprehension of the package.

If you anticipate that a function may need more arguments in the future, you can plan ahead by making optional arguments a part of the function’s signature. The simplest way to do that is to add a single struct argument, as the [crypto/tls.Dial](https://pkg.go.dev/crypto/tls?tab=doc#Dial) function does:
    
    
    func Dial(network, addr string, config *Config) (*Conn, error)
    

The TLS handshake conducted by `Dial` requires a network and address, but it has many other parameters with reasonable defaults. Passing a `nil` for `config` uses those defaults; passing a `Config` struct with some fields set will override the defaults for those fields. In the future, adding a new TLS configuration parameter only requires a new field on the `Config` struct, a change that is backward-compatible (almost always—see “Maintaining struct compatibility” below).

Sometimes the techniques of adding a new function and adding options can be combined by making the options struct a method receiver. Consider the evolution of the `net` package’s ability to listen at a network address. Prior to Go 1.11, the `net` package provided only a `Listen` function with the signature
    
    
    func Listen(network, address string) (Listener, error)
    

For Go 1.11, two features were added to `net` listening: passing a context, and allowing the caller to provide a “control function” to adjust the raw connection after creation but before binding. The result could have been a new function that took a context, network, address and control function. Instead, the package authors added a [`ListenConfig`](https://pkg.go.dev/net@go1.11?tab=doc#ListenConfig) struct in anticipation that more options might be needed someday. And rather than define a new top-level function with a cumbersome name, they added a `Listen` method to `ListenConfig`:
    
    
    type ListenConfig struct {
        Control func(network, address string, c syscall.RawConn) error
    }
    
    func (*ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error)
    

Another way to provide new options in the future is the “Option types” pattern, where options are passed as variadic arguments, and each option is a function that changes the state of the value being constructed. They are described in more detail by Rob Pike’s post [Self-referential functions and the design of options](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html). One widely used example is [google.golang.org/grpc](https://pkg.go.dev/google.golang.org/grpc?tab=doc)’s [DialOption](https://pkg.go.dev/google.golang.org/grpc?tab=doc#DialOption).

Option types fulfill the same role as struct options in function arguments: they are an extensible way to pass behavior-modifying configuration. Deciding which to choose is largely a matter of style. Consider this simple usage of gRPC’s `DialOption` option type:
    
    
    grpc.Dial("some-target",
      grpc.WithAuthority("some-authority"),
      grpc.WithMaxDelay(time.Second),
      grpc.WithBlock())
    

This could also have been implemented as a struct option:
    
    
    notgrpc.Dial("some-target", &notgrpc.Options{
      Authority: "some-authority",
      MaxDelay:  time.Second,
      Block:     true,
    })
    

Functional options have some downsides: they require writing the package name before the option for each call; they increase the size of the package namespace; and it’s unclear what the behavior should be if the same option is provided twice. On the other hand, functions which take option structs require a parameter which might almost always be `nil`, which some find unattractive. And when a type’s zero value has a valid meaning, it is clumsy to specify that the option should have its default value, typically requiring a pointer or an additional boolean field.

Either one is a reasonable choice for ensuring future extensibility of your module’s public API.

## Working with interfaces

Sometimes, new features require changes to publicly-exposed interfaces: for example, an interface needs to be extended with new methods. Directly adding to an interface is a breaking change, though—how, then, can we support new methods on a publicly-exposed interface?

The basic idea is to define a new interface with the new method, and then wherever the old interface is used, dynamically check whether the provided type is the older type or the newer type.

Let’s illustrate this with an example from the [`archive/tar`](https://pkg.go.dev/archive/tar?tab=doc) package. [`tar.NewReader`](https://pkg.go.dev/archive/tar?tab=doc#NewReader) accepts an `io.Reader`, but over time the Go team realized that it would be more efficient to skip from one file header to the next if you could call [`Seek`](https://pkg.go.dev/io?tab=doc#Seeker). But, they could not add a `Seek` method to `io.Reader`: that would break all implementers of `io.Reader`.

Another ruled-out option was to change `tar.NewReader` to accept [`io.ReadSeeker`](https://pkg.go.dev/io?tab=doc#ReadSeeker) rather than `io.Reader`, since it supports both `io.Reader` methods and `Seek` (by way of `io.Seeker`). But, as we saw above, changing a function signature is also a breaking change.

So, they decided to keep `tar.NewReader` signature unchanged, but type check for (and support) `io.Seeker` in `tar.Reader` methods:
    
    
    package tar
    
    type Reader struct {
      r io.Reader
    }
    
    func NewReader(r io.Reader) *Reader {
      return &Reader{r: r}
    }
    
    func (r *Reader) Read(b []byte) (int, error) {
      if rs, ok := r.r.(io.Seeker); ok {
        // Use more efficient rs.Seek.
      }
      // Use less efficient r.r.Read.
    }
    

(See [reader.go](https://github.com/golang/go/blob/60f78765022a59725121d3b800268adffe78bde3/src/archive/tar/reader.go#L837) for the actual code.)

When you run into a case where you want to add a method to an existing interface, you may be able to follow this strategy. Start by creating a new interface with your new method, or identify an existing interface with the new method. Next, identify the relevant functions that need to support it, type check for the second interface, and add code that uses it.

This strategy only works when the old interface without the new method can still be supported, limiting the future extensibility of your module.

Where possible, it is better to avoid this class of problem entirely. When designing constructors, for example, prefer to return concrete types. Working with concrete types allows you to add methods in the future without breaking users, unlike interfaces. That property allows your module to be extended more easily in the future.

Tip: if you do need to use an interface but don’t intend for users to implement it, you can add an unexported method. This prevents types defined outside your package from satisfying your interface without embedding, freeing you to add methods later without breaking user implementations. For example, see [`testing.TB`’s `private()` function](https://github.com/golang/go/blob/83b181c68bf332ac7948f145f33d128377a09c42/src/testing/testing.go#L564-L567).
    
    
    // TB is the interface common to T and B.
    type TB interface {
        Error(args ...interface{})
        Errorf(format string, args ...interface{})
        // ...
    
        // A private method to prevent users implementing the
        // interface and so future additions to it will not
        // violate Go 1 compatibility.
        private()
    }
    

This topic is also explored in more detail in Jonathan Amsterdam’s “Detecting Incompatible API Changes” talk ([video](https://www.youtube.com/watch?v=JhdL5AkH-AQ), [slides](https://github.com/gophercon/2019-talks/blob/master/JonathanAmsterdam-DetectingIncompatibleAPIChanges/slides.pdf)).

## Add configuration methods

So far we’ve talked about overt breaking changes, where changing a type or a function would cause users’ code to stop compiling. However, behavior changes can also break users, even if user code continues to compile. For example, many users expect [`json.Decoder`](https://pkg.go.dev/encoding/json?tab=doc#Decoder) to ignore fields in the JSON that are not in the argument struct. When the Go team wanted to return an error in that case, they had to be careful. Doing so without an opt-in mechanism would mean that the many users relying on those methods might start receiving errors where they hadn’t before.

So, rather than changing the behavior for all users, they added a configuration method to the `Decoder` struct: [`Decoder.DisallowUnknownFields`](https://pkg.go.dev/encoding/json?tab=doc#Decoder.DisallowUnknownFields). Calling this method opts a user in to the new behavior, but not doing so preserves the old behavior for existing users.

## Maintaining struct compatibility

We saw above that any change to a function’s signature is a breaking change. The situation is much better with structs. If you have an exported struct type, you can almost always add a field or remove an unexported field without breaking compatibility. When adding a field, make sure that its zero value is meaningful and preserves the old behavior, so that existing code that doesn’t set the field continues to work.

Recall that the authors of the `net` package added `ListenConfig` in Go 1.11 because they thought more options might be forthcoming. Turns out they were right. In Go 1.13, the [`KeepAlive` field](https://pkg.go.dev/net@go1.13?tab=doc#ListenConfig) was added to allow for disabling keep-alive or changing its period. The default value of zero preserves the original behavior of enabling keep-alive with a default period.

There is one subtle way a new field can break user code unexpectedly. If all the field types in a struct are comparable—meaning values of those types can be compared with `==` and `!=` and used as a map key—then the overall struct type is comparable too. In this case, adding a new field of uncomparable type will make the overall struct type non-comparable, breaking any code that compares values of that struct type.

To keep a struct comparable, don’t add non-comparable fields to it. You can write a test for that, or rely on the upcoming [gorelease](https://pkg.go.dev/golang.org/x/exp/cmd/gorelease?tab=doc) tool to catch it.

To prevent comparison in the first place, make sure the struct has a non-comparable field. It may have one already—no slice, map or function type is comparable—but if not, one can be added like so:
    
    
    type Point struct {
            _ [0]func()
            X int
            Y int
    }
    

The `func()` type is not comparable, and the zero-length array takes up no space. We can define a type to clarify our intent:
    
    
    type doNotCompare [0]func()
    
    type Point struct {
            doNotCompare
            X int
            Y int
    }
    

Should you use `doNotCompare` in your structs? If you’ve defined the struct to be used as a pointer—that is, it has pointer methods and perhaps a `NewXXX` constructor function that returns a pointer—then adding a `doNotCompare` field is probably overkill. Users of a pointer type understand that each value of the type is distinct: that if they want to compare two values, they should compare the pointers.

If you are defining a struct intended to be used as a value directly, like our `Point` example, then quite often you want it to be comparable. In the uncommon case that you have a value struct that you don’t want compared, then adding a `doNotCompare` field will give you the freedom to change the struct later without having to worry about breaking comparisons. On the downside, the type won’t be usable as a map key.

## Conclusion

When planning an API from scratch, consider carefully how extensible the API will be to new changes in the future. And when you do need to add new features, remember the rule: add, don’t change or remove, keeping in mind the exceptions—interfaces, function arguments, and return values can’t be added in backwards-compatible ways.

If you need to dramatically change an API, or if an API begins to lose its focus as more features are added, then it may be time for a new major version. But most of the time, making a backwards-compatible change is easy and avoids causing pain for your users.

**Next article:**[Go 1.15 is released](/blog/go1.15)  
**Previous article:**[The Next Step for Generics](/blog/generics-next-step)  
**[Blog Index](/blog/all)**