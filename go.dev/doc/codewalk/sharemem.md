---
source_url: https://go.dev/doc/codewalk/sharemem
title: Codewalk: Share Memory By Communicating - The Go Programming Language
crawl_date: 2025-07-25T12:08:26.516896
watsonmd_version: 0.1.0
---

# Codewalk: Share Memory By Communicating

[ ![Pop Out Code](/doc/codewalk/popout.png) ]() doc/codewalk/urlpoll.go

code on left • right code width 70% filepaths shown • hidden

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=0&hi=0#mark)

Introduction

Go's approach to concurrency differs from the traditional use of threads and shared memory. Philosophically, it can be summarized:   
  
_Don't communicate by sharing memory; share memory by communicating._   
  
Channels allow you to pass references to data structures between goroutines. If you consider this as passing around ownership of the data (the ability to read and write it), they become a powerful and expressive synchronization mechanism.   
  
In this codewalk we will look at a simple program that polls a list of URLs, checking their HTTP response codes and periodically printing their state. 

doc/codewalk/urlpoll.go

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=26&hi=30#mark)

State type

The State type represents the state of a URL.   
  
The Pollers send State values to the StateMonitor, which maintains a map of the current state of each URL. 

doc/codewalk/urlpoll.go:26,30

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=60&hi=64#mark)

Resource type

A Resource represents the state of a URL to be polled: the URL itself and the number of errors encountered since the last successful poll.   
  
When the program starts, it allocates one Resource for each URL. The main goroutine and the Poller goroutines send the Resources to each other on channels. 

doc/codewalk/urlpoll.go:60,64

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=86&hi=92#mark)

Poller function

Each Poller receives Resource pointers from an input channel. In this program, the convention is that sending a Resource pointer on a channel passes ownership of the underlying data from the sender to the receiver. Because of this convention, we know that no two goroutines will access this Resource at the same time. This means we don't have to worry about locking to prevent concurrent access to these data structures.   
  
The Poller processes the Resource by calling its Poll method.   
  
It sends a State value to the status channel, to inform the StateMonitor of the result of the Poll.   
  
Finally, it sends the Resource pointer to the out channel. This can be interpreted as the Poller saying "I'm done with this Resource" and returning ownership of it to the main goroutine.   
  
Several goroutines run Pollers, processing Resources in parallel. 

doc/codewalk/urlpoll.go:86,92

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=66&hi=77#mark)

The Poll method

The Poll method (of the Resource type) performs an HTTP HEAD request for the Resource's URL and returns the HTTP response's status code. If an error occurs, Poll logs the message to standard error and returns the error string instead. 

doc/codewalk/urlpoll.go:66,77

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=94&hi=116#mark)

main function

The main function starts the Poller and StateMonitor goroutines and then loops passing completed Resources back to the pending channel after appropriate delays. 

doc/codewalk/urlpoll.go:94,116

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=95&hi=96#mark)

Creating channels

First, main makes two channels of *Resource, pending and complete.   
  
Inside main, a new goroutine sends one Resource per URL to pending and the main goroutine receives completed Resources from complete.   
  
The pending and complete channels are passed to each of the Poller goroutines, within which they are known as in and out. 

doc/codewalk/urlpoll.go:95,96

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=98&hi=99#mark)

Initializing StateMonitor

StateMonitor will initialize and launch a goroutine that stores the state of each Resource. We will look at this function in detail later.   
  
For now, the important thing to note is that it returns a channel of State, which is saved as status and passed to the Poller goroutines. 

doc/codewalk/urlpoll.go:98,99

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=101&hi=104#mark)

Launching Poller goroutines

Now that it has the necessary channels, main launches a number of Poller goroutines, passing the channels as arguments. The channels provide the means of communication between the main, Poller, and StateMonitor goroutines. 

doc/codewalk/urlpoll.go:101,104

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=106&hi=111#mark)

Send Resources to pending

To add the initial work to the system, main starts a new goroutine that allocates and sends one Resource per URL to pending.   
  
The new goroutine is necessary because unbuffered channel sends and receives are synchronous. That means these channel sends will block until the Pollers are ready to read from pending.   
  
Were these sends performed in the main goroutine with fewer Pollers than channel sends, the program would reach a deadlock situation, because main would not yet be receiving from complete.   
  
Exercise for the reader: modify this part of the program to read a list of URLs from a file. (You may want to move this goroutine into its own named function.) 

doc/codewalk/urlpoll.go:106,111

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=113&hi=115#mark)

Main Event Loop

When a Poller is done with a Resource, it sends it on the complete channel. This loop receives those Resource pointers from complete. For each received Resource, it starts a new goroutine calling the Resource's Sleep method. Using a new goroutine for each ensures that the sleeps can happen in parallel.   
  
Note that any single Resource pointer may only be sent on either pending or complete at any one time. This ensures that a Resource is either being handled by a Poller goroutine or sleeping, but never both simultaneously. In this way, we share our Resource data by communicating. 

doc/codewalk/urlpoll.go:113,115

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=79&hi=84#mark)

The Sleep method

Sleep calls time.Sleep to pause before sending the Resource to done. The pause will either be of a fixed length (pollInterval) plus an additional delay proportional to the number of sequential errors (r.errCount).   
  
This is an example of a typical Go idiom: a function intended to run inside a goroutine takes a channel, upon which it sends its return value (or other indication of completed state). 

doc/codewalk/urlpoll.go:79,84

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=32&hi=50#mark)

StateMonitor

The StateMonitor receives State values on a channel and periodically outputs the state of all Resources being polled by the program. 

doc/codewalk/urlpoll.go:32,50

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=36&hi=36#mark)

The updates channel

The variable updates is a channel of State, on which the Poller goroutines send State values.   
  
This channel is returned by the function. 

doc/codewalk/urlpoll.go:36

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=37&hi=37#mark)

The urlStatus map

The variable urlStatus is a map of URLs to their most recent status. 

doc/codewalk/urlpoll.go:37

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=38&hi=38#mark)

The Ticker object

A time.Ticker is an object that repeatedly sends a value on a channel at a specified interval.   
  
In this case, ticker triggers the printing of the current state to standard output every updateInterval nanoseconds. 

doc/codewalk/urlpoll.go:38

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=39&hi=48#mark)

The StateMonitor goroutine

StateMonitor will loop forever, selecting on two channels: ticker.C and update. The select statement blocks until one of its communications is ready to proceed.   
  
When StateMonitor receives a tick from ticker.C, it calls logState to print the current state. When it receives a State update from updates, it records the new status in the urlStatus map.   
  
Notice that this goroutine owns the urlStatus data structure, ensuring that it can only be accessed sequentially. This prevents memory corruption issues that might arise from parallel reads and/or writes to a shared map. 

doc/codewalk/urlpoll.go:39,48

[](/doc/codewalk/?fileprint=/doc%2fcodewalk%2furlpoll.go&lo=0&hi=0#mark)

Conclusion

In this codewalk we have explored a simple example of using Go's concurrency primitives to share memory through communication.   
  
This should provide a starting point from which to explore the ways in which goroutines and channels can be used to write expressive and concise concurrent programs. 

doc/codewalk/urlpoll.go

previous step • next step