---
source_url: https://go.dev/blog/json-and-go
title: JSON and Go - The Go Programming Language
crawl_date: 2025-07-25T12:08:26.575450
watsonmd_version: 0.1.0
---

# [The Go Blog](/blog/)

# JSON and Go

Andrew Gerrand  
25 January 2011 

## Introduction

JSON (JavaScript Object Notation) is a simple data interchange format. Syntactically it resembles the objects and lists of JavaScript. It is most commonly used for communication between web back-ends and JavaScript programs running in the browser, but it is used in many other places, too. Its home page, [json.org](http://json.org), provides a wonderfully clear and concise definition of the standard.

With the [json package](/pkg/encoding/json/) it’s a snap to read and write JSON data from your Go programs.

## Encoding

To encode JSON data we use the [`Marshal`](/pkg/encoding/json/#Marshal) function.
    
    
    func Marshal(v interface{}) ([]byte, error)
    

Given the Go data structure, `Message`,
    
    
    type Message struct {
        Name string
        Body string
        Time int64
    }
    

and an instance of `Message`
    
    
    m := Message{"Alice", "Hello", 1294706395881547000}
    

we can marshal a JSON-encoded version of m using `json.Marshal`:
    
    
    b, err := json.Marshal(m)
    

If all is well, `err` will be `nil` and `b` will be a `[]byte` containing this JSON data:
    
    
    b == []byte(`{"Name":"Alice","Body":"Hello","Time":1294706395881547000}`)
    

Only data structures that can be represented as valid JSON will be encoded:

  * JSON objects only support strings as keys; to encode a Go map type it must be of the form `map[string]T` (where `T` is any Go type supported by the json package).

  * Channel, complex, and function types cannot be encoded.

  * Cyclic data structures are not supported; they will cause `Marshal` to go into an infinite loop.

  * Pointers will be encoded as the values they point to (or ’null’ if the pointer is `nil`).




The json package only accesses the exported fields of struct types (those that begin with an uppercase letter). Therefore only the exported fields of a struct will be present in the JSON output.

## Decoding

To decode JSON data we use the [`Unmarshal`](/pkg/encoding/json/#Unmarshal) function.
    
    
    func Unmarshal(data []byte, v interface{}) error
    

We must first create a place where the decoded data will be stored
    
    
    var m Message
    

and call `json.Unmarshal`, passing it a `[]byte` of JSON data and a pointer to `m`
    
    
    err := json.Unmarshal(b, &m)
    

If `b` contains valid JSON that fits in `m`, after the call `err` will be `nil` and the data from `b` will have been stored in the struct `m`, as if by an assignment like:
    
    
    m = Message{
        Name: "Alice",
        Body: "Hello",
        Time: 1294706395881547000,
    }
    

How does `Unmarshal` identify the fields in which to store the decoded data? For a given JSON key `"Foo"`, `Unmarshal` will look through the destination struct’s fields to find (in order of preference):

  * An exported field with a tag of `"Foo"` (see the [Go spec](/ref/spec#Struct_types) for more on struct tags),

  * An exported field named `"Foo"`, or

  * An exported field named `"FOO"` or `"FoO"` or some other case-insensitive match of `"Foo"`.




What happens when the structure of the JSON data doesn’t exactly match the Go type?
    
    
    b := []byte(`{"Name":"Bob","Food":"Pickle"}`)
    var m Message
    err := json.Unmarshal(b, &m)
    

`Unmarshal` will decode only the fields that it can find in the destination type. In this case, only the Name field of m will be populated, and the Food field will be ignored. This behavior is particularly useful when you wish to pick only a few specific fields out of a large JSON blob. It also means that any unexported fields in the destination struct will be unaffected by `Unmarshal`.

But what if you don’t know the structure of your JSON data beforehand?

## Generic JSON with interface

The `interface{}` (empty interface) type describes an interface with zero methods. Every Go type implements at least zero methods and therefore satisfies the empty interface.

The empty interface serves as a general container type:
    
    
    var i interface{}
    i = "a string"
    i = 2011
    i = 2.777
    

A type assertion accesses the underlying concrete type:
    
    
    r := i.(float64)
    fmt.Println("the circle's area", math.Pi*r*r)
    

Or, if the underlying type is unknown, a type switch determines the type:
    
    
    switch v := i.(type) {
    case int:
        fmt.Println("twice i is", v*2)
    case float64:
        fmt.Println("the reciprocal of i is", 1/v)
    case string:
        h := len(v) / 2
        fmt.Println("i swapped by halves is", v[h:]+v[:h])
    default:
        // i isn't one of the types above
    }
    

The json package uses `map[string]interface{}` and `[]interface{}` values to store arbitrary JSON objects and arrays; it will happily unmarshal any valid JSON blob into a plain `interface{}` value. The default concrete Go types are:

  * `bool` for JSON booleans,

  * `float64` for JSON numbers,

  * `string` for JSON strings, and

  * `nil` for JSON null.




## Decoding arbitrary data

Consider this JSON data, stored in the variable `b`:
    
    
    b := []byte(`{"Name":"Wednesday","Age":6,"Parents":["Gomez","Morticia"]}`)
    

Without knowing this data’s structure, we can decode it into an `interface{}` value with `Unmarshal`:
    
    
    var f interface{}
    err := json.Unmarshal(b, &f)
    

At this point the Go value in `f` would be a map whose keys are strings and whose values are themselves stored as empty interface values:
    
    
    f = map[string]interface{}{
        "Name": "Wednesday",
        "Age":  6,
        "Parents": []interface{}{
            "Gomez",
            "Morticia",
        },
    }
    

To access this data we can use a type assertion to access `f`’s underlying `map[string]interface{}`:
    
    
    m := f.(map[string]interface{})
    

We can then iterate through the map with a range statement and use a type switch to access its values as their concrete types:
    
    
    for k, v := range m {
        switch vv := v.(type) {
        case string:
            fmt.Println(k, "is string", vv)
        case float64:
            fmt.Println(k, "is float64", vv)
        case []interface{}:
            fmt.Println(k, "is an array:")
            for i, u := range vv {
                fmt.Println(i, u)
            }
        default:
            fmt.Println(k, "is of a type I don't know how to handle")
        }
    }
    

In this way you can work with unknown JSON data while still enjoying the benefits of type safety.

## Reference Types

Let’s define a Go type to contain the data from the previous example:
    
    
    type FamilyMember struct {
        Name    string
        Age     int
        Parents []string
    }
    
    var m FamilyMember
    err := json.Unmarshal(b, &m)
    

Unmarshaling that data into a `FamilyMember` value works as expected, but if we look closely we can see a remarkable thing has happened. With the var statement we allocated a `FamilyMember` struct, and then provided a pointer to that value to `Unmarshal`, but at that time the `Parents` field was a `nil` slice value. To populate the `Parents` field, `Unmarshal` allocated a new slice behind the scenes. This is typical of how `Unmarshal` works with the supported reference types (pointers, slices, and maps).

Consider unmarshaling into this data structure:
    
    
    type Foo struct {
        Bar *Bar
    }
    

If there were a `Bar` field in the JSON object, `Unmarshal` would allocate a new `Bar` and populate it. If not, `Bar` would be left as a `nil` pointer.

From this a useful pattern arises: if you have an application that receives a few distinct message types, you might define “receiver” structure like
    
    
    type IncomingMessage struct {
        Cmd *Command
        Msg *Message
    }
    

and the sending party can populate the `Cmd` field and/or the `Msg` field of the top-level JSON object, depending on the type of message they want to communicate. `Unmarshal`, when decoding the JSON into an `IncomingMessage` struct, will only allocate the data structures present in the JSON data. To know which messages to process, the programmer need simply test that either `Cmd` or `Msg` is not `nil`.

## Streaming Encoders and Decoders

The json package provides `Decoder` and `Encoder` types to support the common operation of reading and writing streams of JSON data. The `NewDecoder` and `NewEncoder` functions wrap the [`io.Reader`](/pkg/io/#Reader) and [`io.Writer`](/pkg/io/#Writer) interface types.
    
    
    func NewDecoder(r io.Reader) *Decoder
    func NewEncoder(w io.Writer) *Encoder
    

Here’s an example program that reads a series of JSON objects from standard input, removes all but the `Name` field from each object, and then writes the objects to standard output:
    
    
    package main
    
    import (
        "encoding/json"
        "log"
        "os"
    )
    
    func main() {
        dec := json.NewDecoder(os.Stdin)
        enc := json.NewEncoder(os.Stdout)
        for {
            var v map[string]interface{}
            if err := dec.Decode(&v); err != nil {
                log.Println(err)
                return
            }
            for k := range v {
                if k != "Name" {
                    delete(v, k)
                }
            }
            if err := enc.Encode(&v); err != nil {
                log.Println(err)
            }
        }
    }
    

Due to the ubiquity of Readers and Writers, these `Encoder` and `Decoder` types can be used in a broad range of scenarios, such as reading and writing to HTTP connections, WebSockets, or files.

## References

For more information see the [json package documentation](/pkg/encoding/json/). For an example usage of json see the source files of the [jsonrpc package](/pkg/net/rpc/jsonrpc/).

**Next article:**[Go becomes more stable](/blog/stable-releases)  
**Previous article:**[Go Slices: usage and internals](/blog/slices-intro)  
**[Blog Index](/blog/all)**