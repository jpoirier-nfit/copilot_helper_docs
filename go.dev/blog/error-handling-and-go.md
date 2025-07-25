---
source_url: https://go.dev/blog/error-handling-and-go
title: Error handling and Go - The Go Programming Language
crawl_date: 2025-07-25T12:08:26.561449
watsonmd_version: 0.1.0
---

# [The Go Blog](/blog/)

# Error handling and Go

Andrew Gerrand  
12 July 2011 

## Introduction

If you have written any Go code you have probably encountered the built-in `error` type. Go code uses `error` values to indicate an abnormal state. For example, the `os.Open` function returns a non-nil `error` value when it fails to open a file.
    
    
    func Open(name string) (file *File, err error)
    

The following code uses `os.Open` to open a file. If an error occurs it calls `log.Fatal` to print the error message and stop.
    
    
    f, err := os.Open("filename.ext")
    if err != nil {
        log.Fatal(err)
    }
    // do something with the open *File f
    

You can get a lot done in Go knowing just this about the `error` type, but in this article we’ll take a closer look at `error` and discuss some good practices for error handling in Go.

## The error type

The `error` type is an interface type. An `error` variable represents any value that can describe itself as a string. Here is the interface’s declaration:
    
    
    type error interface {
        Error() string
    }
    

The `error` type, as with all built in types, is [predeclared](/doc/go_spec.html#Predeclared_identifiers) in the [universe block](/doc/go_spec.html#Blocks).

The most commonly-used `error` implementation is the [errors](/pkg/errors/) package’s unexported `errorString` type.
    
    
    // errorString is a trivial implementation of error.
    type errorString struct {
        s string
    }
    
    func (e *errorString) Error() string {
        return e.s
    }
    

You can construct one of these values with the `errors.New` function. It takes a string that it converts to an `errors.errorString` and returns as an `error` value.
    
    
    // New returns an error that formats as the given text.
    func New(text string) error {
        return &errorString{text}
    }
    

Here’s how you might use `errors.New`:
    
    
    func Sqrt(f float64) (float64, error) {
        if f < 0 {
            return 0, errors.New("math: square root of negative number")
        }
        // implementation
    }
    

A caller passing a negative argument to `Sqrt` receives a non-nil `error` value (whose concrete representation is an `errors.errorString` value). The caller can access the error string (“math: square root of…”) by calling the `error`’s `Error` method, or by just printing it:
    
    
    f, err := Sqrt(-1)
    if err != nil {
        fmt.Println(err)
    }
    

The [fmt](/pkg/fmt/) package formats an `error` value by calling its `Error() string` method.

It is the error implementation’s responsibility to summarize the context. The error returned by `os.Open` formats as “open /etc/passwd: permission denied,” not just “permission denied.” The error returned by our `Sqrt` is missing information about the invalid argument.

To add that information, a useful function is the `fmt` package’s `Errorf`. It formats a string according to `Printf`’s rules and returns it as an `error` created by `errors.New`.
    
    
    if f < 0 {
        return 0, fmt.Errorf("math: square root of negative number %g", f)
    }
    

In many cases `fmt.Errorf` is good enough, but since `error` is an interface, you can use arbitrary data structures as error values, to allow callers to inspect the details of the error.

For instance, our hypothetical callers might want to recover the invalid argument passed to `Sqrt`. We can enable that by defining a new error implementation instead of using `errors.errorString`:
    
    
    type NegativeSqrtError float64
    
    func (f NegativeSqrtError) Error() string {
        return fmt.Sprintf("math: square root of negative number %g", float64(f))
    }
    

A sophisticated caller can then use a [type assertion](/doc/go_spec.html#Type_assertions) to check for a `NegativeSqrtError` and handle it specially, while callers that just pass the error to `fmt.Println` or `log.Fatal` will see no change in behavior.

As another example, the [json](/pkg/encoding/json/) package specifies a `SyntaxError` type that the `json.Decode` function returns when it encounters a syntax error parsing a JSON blob.
    
    
    type SyntaxError struct {
        msg    string // description of error
        Offset int64  // error occurred after reading Offset bytes
    }
    
    func (e *SyntaxError) Error() string { return e.msg }
    

The `Offset` field isn’t even shown in the default formatting of the error, but callers can use it to add file and line information to their error messages:
    
    
    if err := dec.Decode(&val); err != nil {
        if serr, ok := err.(*json.SyntaxError); ok {
            line, col := findLine(f, serr.Offset)
            return fmt.Errorf("%s:%d:%d: %v", f.Name(), line, col, err)
        }
        return err
    }
    

(This is a slightly simplified version of some [actual code](https://github.com/camlistore/go4/blob/03efcb870d84809319ea509714dd6d19a1498483/jsonconfig/eval.go#L123-L135) from the [Camlistore](http://camlistore.org) project.)

The `error` interface requires only a `Error` method; specific error implementations might have additional methods. For instance, the [net](/pkg/net/) package returns errors of type `error`, following the usual convention, but some of the error implementations have additional methods defined by the `net.Error` interface:
    
    
    package net
    
    type Error interface {
        error
        Timeout() bool   // Is the error a timeout?
        Temporary() bool // Is the error temporary?
    }
    

Client code can test for a `net.Error` with a type assertion and then distinguish transient network errors from permanent ones. For instance, a web crawler might sleep and retry when it encounters a temporary error and give up otherwise.
    
    
    if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
        time.Sleep(1e9)
        continue
    }
    if err != nil {
        log.Fatal(err)
    }
    

## Simplifying repetitive error handling

In Go, error handling is important. The language’s design and conventions encourage you to explicitly check for errors where they occur (as distinct from the convention in other languages of throwing exceptions and sometimes catching them). In some cases this makes Go code verbose, but fortunately there are some techniques you can use to minimize repetitive error handling.

Consider an [App Engine](https://cloud.google.com/appengine/docs/go/) application with an HTTP handler that retrieves a record from the datastore and formats it with a template.
    
    
    func init() {
        http.HandleFunc("/view", viewRecord)
    }
    
    func viewRecord(w http.ResponseWriter, r *http.Request) {
        c := appengine.NewContext(r)
        key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
        record := new(Record)
        if err := datastore.Get(c, key, record); err != nil {
            http.Error(w, err.Error(), 500)
            return
        }
        if err := viewTemplate.Execute(w, record); err != nil {
            http.Error(w, err.Error(), 500)
        }
    }
    

This function handles errors returned by the `datastore.Get` function and `viewTemplate`’s `Execute` method. In both cases, it presents a simple error message to the user with the HTTP status code 500 (“Internal Server Error”). This looks like a manageable amount of code, but add some more HTTP handlers and you quickly end up with many copies of identical error handling code.

To reduce the repetition we can define our own HTTP `appHandler` type that includes an `error` return value:
    
    
    type appHandler func(http.ResponseWriter, *http.Request) error
    

Then we can change our `viewRecord` function to return errors:
    
    
    func viewRecord(w http.ResponseWriter, r *http.Request) error {
        c := appengine.NewContext(r)
        key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
        record := new(Record)
        if err := datastore.Get(c, key, record); err != nil {
            return err
        }
        return viewTemplate.Execute(w, record)
    }
    

This is simpler than the original version, but the [http](/pkg/net/http/) package doesn’t understand functions that return `error`. To fix this we can implement the `http.Handler` interface’s `ServeHTTP` method on `appHandler`:
    
    
    func (fn appHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
        if err := fn(w, r); err != nil {
            http.Error(w, err.Error(), 500)
        }
    }
    

The `ServeHTTP` method calls the `appHandler` function and displays the returned error (if any) to the user. Notice that the method’s receiver, `fn`, is a function. (Go can do that!) The method invokes the function by calling the receiver in the expression `fn(w, r)`.

Now when registering `viewRecord` with the http package we use the `Handle` function (instead of `HandleFunc`) as `appHandler` is an `http.Handler` (not an `http.HandlerFunc`).
    
    
    func init() {
        http.Handle("/view", appHandler(viewRecord))
    }
    

With this basic error handling infrastructure in place, we can make it more user friendly. Rather than just displaying the error string, it would be better to give the user a simple error message with an appropriate HTTP status code, while logging the full error to the App Engine developer console for debugging purposes.

To do this we create an `appError` struct containing an `error` and some other fields:
    
    
    type appError struct {
        Error   error
        Message string
        Code    int
    }
    

Next we modify the appHandler type to return `*appError` values:
    
    
    type appHandler func(http.ResponseWriter, *http.Request) *appError
    

(It’s usually a mistake to pass back the concrete type of an error rather than `error`, for reasons discussed in [the Go FAQ](/doc/go_faq.html#nil_error), but it’s the right thing to do here because `ServeHTTP` is the only place that sees the value and uses its contents.)

And make `appHandler`’s `ServeHTTP` method display the `appError`’s `Message` to the user with the correct HTTP status `Code` and log the full `Error` to the developer console:
    
    
    func (fn appHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
        if e := fn(w, r); e != nil { // e is *appError, not os.Error.
            c := appengine.NewContext(r)
            c.Errorf("%v", e.Error)
            http.Error(w, e.Message, e.Code)
        }
    }
    

Finally, we update `viewRecord` to the new function signature and have it return more context when it encounters an error:
    
    
    func viewRecord(w http.ResponseWriter, r *http.Request) *appError {
        c := appengine.NewContext(r)
        key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
        record := new(Record)
        if err := datastore.Get(c, key, record); err != nil {
            return &appError{err, "Record not found", 404}
        }
        if err := viewTemplate.Execute(w, record); err != nil {
            return &appError{err, "Can't display record", 500}
        }
        return nil
    }
    

This version of `viewRecord` is the same length as the original, but now each of those lines has specific meaning and we are providing a friendlier user experience.

It doesn’t end there; we can further improve the error handling in our application. Some ideas:

  * give the error handler a pretty HTML template,

  * make debugging easier by writing the stack trace to the HTTP response when the user is an administrator,

  * write a constructor function for `appError` that stores the stack trace for easier debugging,

  * recover from panics inside the `appHandler`, logging the error to the console as “Critical,” while telling the user “a serious error has occurred.” This is a nice touch to avoid exposing the user to inscrutable error messages caused by programming errors. See the [Defer, Panic, and Recover](/doc/articles/defer_panic_recover.html) article for more details.




## Conclusion

Proper error handling is an essential requirement of good software. By employing the techniques described in this post you should be able to write more reliable and succinct Go code.

**Next article:**[Go for App Engine is now generally available](/blog/appengine-ga)  
**Previous article:**[First Class Functions in Go](/blog/functions-codewalk)  
**[Blog Index](/blog/all)**