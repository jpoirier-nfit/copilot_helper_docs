---
source_url: https://go.dev/blog/race-detector
title: Introducing the Go Race Detector - The Go Programming Language
crawl_date: 2025-07-25T12:08:27.608085
watsonmd_version: 0.1.0
---

# [The Go Blog](/blog/)

# Introducing the Go Race Detector

Dmitry Vyukov and Andrew Gerrand  
26 June 2013 

## Introduction

[Race conditions](http://en.wikipedia.org/wiki/Race_condition) are among the most insidious and elusive programming errors. They typically cause erratic and mysterious failures, often long after the code has been deployed to production. While Go’s concurrency mechanisms make it easy to write clean concurrent code, they don’t prevent race conditions. Care, diligence, and testing are required. And tools can help.

We’re happy to announce that Go 1.1 includes a [race detector](/doc/articles/race_detector.html), a new tool for finding race conditions in Go code. It is currently available for Linux, OS X, and Windows systems with 64-bit x86 processors.

The race detector is based on the C/C++ [ThreadSanitizer runtime library](https://github.com/google/sanitizers), which has been used to detect many errors in Google’s internal code base and in [Chromium](http://www.chromium.org/). The technology was integrated with Go in September 2012; since then it has detected [42 races](https://github.com/golang/go/issues?utf8=%E2%9C%93&q=ThreadSanitizer) in the standard library. It is now part of our continuous build process, where it continues to catch race conditions as they arise.

## How it works

The race detector is integrated with the go tool chain. When the `-race` command-line flag is set, the compiler instruments all memory accesses with code that records when and how the memory was accessed, while the runtime library watches for unsynchronized accesses to shared variables. When such “racy” behavior is detected, a warning is printed. (See [this article](https://github.com/google/sanitizers/wiki/ThreadSanitizerAlgorithm) for the details of the algorithm.)

Because of its design, the race detector can detect race conditions only when they are actually triggered by running code, which means it’s important to run race-enabled binaries under realistic workloads. However, race-enabled binaries can use ten times the CPU and memory, so it is impractical to enable the race detector all the time. One way out of this dilemma is to run some tests with the race detector enabled. Load tests and integration tests are good candidates, since they tend to exercise concurrent parts of the code. Another approach using production workloads is to deploy a single race-enabled instance within a pool of running servers.

## Using the race detector

The race detector is fully integrated with the Go tool chain. To build your code with the race detector enabled, just add the `-race` flag to the command line:
    
    
    $ go test -race mypkg    // test the package
    $ go run -race mysrc.go  // compile and run the program
    $ go build -race mycmd   // build the command
    $ go install -race mypkg // install the package
    

To try out the race detector for yourself, copy this example program into `racy.go`:
    
    
    package main
    
    import "fmt"
    
    func main() {
        done := make(chan bool)
        m := make(map[string]string)
        m["name"] = "world"
        go func() {
            m["name"] = "data race"
            done <- true
        }()
        fmt.Println("Hello,", m["name"])
        <-done
    }
    

Then run it with the race detector enabled:
    
    
    $ go run -race racy.go
    

## Examples

Here are two examples of real issues caught by the race detector.

### Example 1: Timer.Reset

The first example is a simplified version of an actual bug found by the race detector. It uses a timer to print a message after a random duration between 0 and 1 second. It does so repeatedly for five seconds. It uses [`time.AfterFunc`](/pkg/time/#AfterFunc) to create a [`Timer`](/pkg/time/#Timer) for the first message and then uses the [`Reset`](/pkg/time/#Timer.Reset) method to schedule the next message, re-using the `Timer` each time.
    
    
    package main
    
    import (
        "fmt"
        "math/rand"
        "time"
    )
    
    
    
    
    
    10  func main() {
    11      start := time.Now()
    12      var t *time.Timer
    13      t = time.AfterFunc(randomDuration(), func() {
    14          fmt.Println(time.Now().Sub(start))
    15          t.Reset(randomDuration())
    16      })
    17      time.Sleep(5 * time.Second)
    18  }
    19  
    20  func randomDuration() time.Duration {
    21      return time.Duration(rand.Int63n(1e9))
    22  }
    23  
    

This looks like reasonable code, but under certain circumstances it fails in a surprising way:
    
    
    panic: runtime error: invalid memory address or nil pointer dereference
    [signal 0xb code=0x1 addr=0x8 pc=0x41e38a]
    
    goroutine 4 [running]:
    time.stopTimer(0x8, 0x12fe6b35d9472d96)
        src/pkg/runtime/ztime_linux_amd64.c:35 +0x25
    time.(*Timer).Reset(0x0, 0x4e5904f, 0x1)
        src/pkg/time/sleep.go:81 +0x42
    main.func·001()
        race.go:14 +0xe3
    created by time.goFunc
        src/pkg/time/sleep.go:122 +0x48
    

What’s going on here? Running the program with the race detector enabled is more illuminating:
    
    
    ==================
    WARNING: DATA RACE
    Read by goroutine 5:
      main.func·001()
         race.go:16 +0x169
    
    Previous write by goroutine 1:
      main.main()
          race.go:14 +0x174
    
    Goroutine 5 (running) created at:
      time.goFunc()
          src/pkg/time/sleep.go:122 +0x56
      timerproc()
         src/pkg/runtime/ztime_linux_amd64.c:181 +0x189
    ==================
    

The race detector shows the problem: an unsynchronized read and write of the variable `t` from different goroutines. If the initial timer duration is very small, the timer function may fire before the main goroutine has assigned a value to `t` and so the call to `t.Reset` is made with a nil `t`.

To fix the race condition we change the code to read and write the variable `t` only from the main goroutine:
    
    
    package main
    
    import (
        "fmt"
        "math/rand"
        "time"
    )
    
    
    
    
    
    10  func main() {
    11      start := time.Now()
    12      reset := make(chan bool)
    13      var t *time.Timer
    14      t = time.AfterFunc(randomDuration(), func() {
    15          fmt.Println(time.Now().Sub(start))
    16          reset <- true
    17      })
    18      for time.Since(start) < 5*time.Second {
    19          <-reset
    20          t.Reset(randomDuration())
    21      }
    22  }
    23  
    
    
    
    func randomDuration() time.Duration {
        return time.Duration(rand.Int63n(1e9))
    }
    
    

Here the main goroutine is wholly responsible for setting and resetting the `Timer` `t` and a new reset channel communicates the need to reset the timer in a thread-safe way.

A simpler but less efficient approach is to [avoid reusing timers](/play/p/kuWTrY0pS4).

### Example 2: ioutil.Discard

The second example is more subtle.

The `ioutil` package’s [`Discard`](/pkg/io/ioutil/#Discard) object implements [`io.Writer`](/pkg/io/#Writer), but discards all the data written to it. Think of it like `/dev/null`: a place to send data that you need to read but don’t want to store. It is commonly used with [`io.Copy`](/pkg/io/#Copy) to drain a reader, like this:
    
    
    io.Copy(ioutil.Discard, reader)
    

Back in July 2011 the Go team noticed that using `Discard` in this way was inefficient: the `Copy` function allocates an internal 32 kB buffer each time it is called, but when used with `Discard` the buffer is unnecessary since we’re just throwing the read data away. We thought that this idiomatic use of `Copy` and `Discard` should not be so costly.

The fix was simple. If the given `Writer` implements a `ReadFrom` method, a `Copy` call like this:
    
    
    io.Copy(writer, reader)
    

is delegated to this potentially more efficient call:
    
    
    writer.ReadFrom(reader)
    

We [added a ReadFrom method](/cl/4817041) to Discard’s underlying type, which has an internal buffer that is shared between all its users. We knew this was theoretically a race condition, but since all writes to the buffer should be thrown away we didn’t think it was important.

When the race detector was implemented it immediately [flagged this code](/issue/3970) as racy. Again, we considered that the code might be problematic, but decided that the race condition wasn’t “real”. To avoid the “false positive” in our build we implemented [a non-racy version](/cl/6624059) that is enabled only when the race detector is running.

But a few months later [Brad](https://bradfitz.com/) encountered a [frustrating and strange bug](/issue/4589). After a few days of debugging, he narrowed it down to a real race condition caused by `ioutil.Discard`.

Here is the known-racy code in `io/ioutil`, where `Discard` is a `devNull` that shares a single buffer between all of its users.
    
    
    var blackHole [4096]byte // shared buffer
    
    func (devNull) ReadFrom(r io.Reader) (n int64, err error) {
        readSize := 0
        for {
            readSize, err = r.Read(blackHole[:])
            n += int64(readSize)
            if err != nil {
                if err == io.EOF {
                    return n, nil
                }
                return
            }
        }
    }
    

Brad’s program includes a `trackDigestReader` type, which wraps an `io.Reader` and records the hash digest of what it reads.
    
    
    type trackDigestReader struct {
        r io.Reader
        h hash.Hash
    }
    
    func (t trackDigestReader) Read(p []byte) (n int, err error) {
        n, err = t.r.Read(p)
        t.h.Write(p[:n])
        return
    }
    

For example, it could be used to compute the SHA-1 hash of a file while reading it:
    
    
    tdr := trackDigestReader{r: file, h: sha1.New()}
    io.Copy(writer, tdr)
    fmt.Printf("File hash: %x", tdr.h.Sum(nil))
    

In some cases there would be nowhere to write the data—but still a need to hash the file—and so `Discard` would be used:
    
    
    io.Copy(ioutil.Discard, tdr)
    

But in this case the `blackHole` buffer isn’t just a black hole; it is a legitimate place to store the data between reading it from the source `io.Reader` and writing it to the `hash.Hash`. With multiple goroutines hashing files simultaneously, each sharing the same `blackHole` buffer, the race condition manifested itself by corrupting the data between reading and hashing. No errors or panics occurred, but the hashes were wrong. Nasty!
    
    
    func (t trackDigestReader) Read(p []byte) (n int, err error) {
        // the buffer p is blackHole
        n, err = t.r.Read(p)
        // p may be corrupted by another goroutine here,
        // between the Read above and the Write below
        t.h.Write(p[:n])
        return
    }
    

The bug was finally [fixed](/cl/7011047) by giving a unique buffer to each use of `ioutil.Discard`, eliminating the race condition on the shared buffer.

## Conclusions

The race detector is a powerful tool for checking the correctness of concurrent programs. It will not issue false positives, so take its warnings seriously. But it is only as good as your tests; you must make sure they thoroughly exercise the concurrent properties of your code so that the race detector can do its job.

What are you waiting for? Run `"go test -race"` on your code today!

**Next article:**[The first Go program](/blog/first-go-program)  
**Previous article:**[Go and the Google Cloud Platform](/blog/io2013-talks-cloud)  
**[Blog Index](/blog/all)**