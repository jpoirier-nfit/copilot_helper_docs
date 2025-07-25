---
source_url: https://go.dev/cmd/cgo/
title: cgo command - cmd/cgo - Go Packages
crawl_date: 2025-07-25T12:08:27.583950
watsonmd_version: 0.1.0
---

[ ![Go](/static/shared/logo/go-blue.svg) ](https://go.dev/)

# cgo

command standard library ![](/static/shared/icon/content_copy_gm_grey_24dp.svg)

[ Version:  go1.24.5 ](?tab=versions)

Opens a new window with list of versions in this module. 

Latest Latest  ![Warning](/static/shared/icon/alert_gm_grey_24dp.svg)

This package is not in the latest version of its module.

[ Go to latest ](/cmd/cgo) Published: Jul 8, 2025  License: [BSD-3-Clause](/cmd/cgo?tab=licenses)

Opens a new window with license information. 

[ Imports: 39 ](/cmd/cgo?tab=imports)

Opens a new window with list of imports. 

[ Imported by: 0 ](/cmd/cgo?tab=importedby)

Opens a new window with list of known importers. 

Main Versions  Licenses  Imports  Imported By 

## Details

  * ![checked](/static/shared/icon/check_circle_gm_grey_24dp.svg) Valid [go.mod](https://cs.opensource.google/go/go/+/go1.24.5:src/go.mod) file ![](/static/shared/icon/help_gm_grey_24dp.svg)

The Go module system was introduced in Go 1.11 and is the official dependency management solution for Go. 

  * ![checked](/static/shared/icon/check_circle_gm_grey_24dp.svg) Redistributable license ![](/static/shared/icon/help_gm_grey_24dp.svg)

Redistributable licenses place minimal restrictions on how software can be used, modified, and redistributed. 

  * ![checked](/static/shared/icon/check_circle_gm_grey_24dp.svg) Tagged version ![](/static/shared/icon/help_gm_grey_24dp.svg)

Modules with tagged versions give importers more predictable builds.

  * ![checked](/static/shared/icon/check_circle_gm_grey_24dp.svg) Stable version ![](/static/shared/icon/help_gm_grey_24dp.svg)

When a project reaches major version v1 it is considered stable.

  * [Learn more about best practices](/about#best-practices)



## Repository

[ cs.opensource.google/go/go ](https://cs.opensource.google/go/go "https://cs.opensource.google/go/go")

## Links

  * [ ![](/static/shared/icon/security_grey_24dp.svg) Report a Vulnerability ](https://go.dev/security/policy "Report security issues in the Go standard library and sub-repositories")



##  ![](/static/shared/icon/code_gm_grey_24dp.svg) Documentation ¶

### Overview ¶

  * Using cgo with the go command
  * Go references to C
  * C references to Go
  * Passing pointers
  * Optimizing calls of C code
  * Special cases
  * Using cgo directly



Cgo enables the creation of Go packages that call C code. 

#### Using cgo with the go command ¶

To use cgo write normal Go code that imports a pseudo-package "C". The Go code can then refer to types such as C.size_t, variables such as C.stdout, or functions such as C.putchar. 

If the import of "C" is immediately preceded by a comment, that comment, called the preamble, is used as a header when compiling the C parts of the package. For example: 
    
    
    // #include <stdio.h>
    // #include <errno.h>
    import "C"
    

The preamble may contain any C code, including function and variable declarations and definitions. These may then be referred to from Go code as though they were defined in the package "C". All names declared in the preamble may be used, even if they start with a lower-case letter. Exception: static variables in the preamble may not be referenced from Go code; static functions are permitted. 

See $GOROOT/cmd/cgo/internal/teststdio and $GOROOT/misc/cgo/gmp for examples. See "C? Go? Cgo!" for an introduction to using cgo: <https://golang.org/doc/articles/c_go_cgo.html>. 

CFLAGS, CPPFLAGS, CXXFLAGS, FFLAGS and LDFLAGS may be defined with pseudo #cgo directives within these comments to tweak the behavior of the C, C++ or Fortran compiler. Values defined in multiple directives are concatenated together. The directive can include a list of build constraints limiting its effect to systems satisfying one of the constraints (see <https://golang.org/pkg/go/build/#hdr-Build_Constraints> for details about the constraint syntax). For example: 
    
    
    // #cgo CFLAGS: -DPNG_DEBUG=1
    // #cgo amd64 386 CFLAGS: -DX86=1
    // #cgo LDFLAGS: -lpng
    // #include <png.h>
    import "C"
    

Alternatively, CPPFLAGS and LDFLAGS may be obtained via the pkg-config tool using a '#cgo pkg-config:' directive followed by the package names. For example: 
    
    
    // #cgo pkg-config: png cairo
    // #include <png.h>
    import "C"
    

The default pkg-config tool may be changed by setting the PKG_CONFIG environment variable. 

For security reasons, only a limited set of flags are allowed, notably -D, -U, -I, and -l. To allow additional flags, set CGO_CFLAGS_ALLOW to a regular expression matching the new flags. To disallow flags that would otherwise be allowed, set CGO_CFLAGS_DISALLOW to a regular expression matching arguments that must be disallowed. In both cases the regular expression must match a full argument: to allow -mfoo=bar, use CGO_CFLAGS_ALLOW='-mfoo.*', not just CGO_CFLAGS_ALLOW='-mfoo'. Similarly named variables control the allowed CPPFLAGS, CXXFLAGS, FFLAGS, and LDFLAGS. 

Also for security reasons, only a limited set of characters are permitted, notably alphanumeric characters and a few symbols, such as '.', that will not be interpreted in unexpected ways. Attempts to use forbidden characters will get a "malformed #cgo argument" error. 

When building, the CGO_CFLAGS, CGO_CPPFLAGS, CGO_CXXFLAGS, CGO_FFLAGS and CGO_LDFLAGS environment variables are added to the flags derived from these directives. Package-specific flags should be set using the directives, not the environment variables, so that builds work in unmodified environments. Flags obtained from environment variables are not subject to the security limitations described above. 

All the cgo CPPFLAGS and CFLAGS directives in a package are concatenated and used to compile C files in that package. All the CPPFLAGS and CXXFLAGS directives in a package are concatenated and used to compile C++ files in that package. All the CPPFLAGS and FFLAGS directives in a package are concatenated and used to compile Fortran files in that package. All the LDFLAGS directives in any package in the program are concatenated and used at link time. All the pkg-config directives are concatenated and sent to pkg-config simultaneously to add to each appropriate set of command-line flags. 

When the cgo directives are parsed, any occurrence of the string ${SRCDIR} will be replaced by the absolute path to the directory containing the source file. This allows pre-compiled static libraries to be included in the package directory and linked properly. For example if package foo is in the directory /go/src/foo: 
    
    
    // #cgo LDFLAGS: -L${SRCDIR}/libs -lfoo
    

Will be expanded to: 
    
    
    // #cgo LDFLAGS: -L/go/src/foo/libs -lfoo
    

When the Go tool sees that one or more Go files use the special import "C", it will look for other non-Go files in the directory and compile them as part of the Go package. Any .c, .s, .S or .sx files will be compiled with the C compiler. Any .cc, .cpp, or .cxx files will be compiled with the C++ compiler. Any .f, .F, .for or .f90 files will be compiled with the fortran compiler. Any .h, .hh, .hpp, or .hxx files will not be compiled separately, but, if these header files are changed, the package (including its non-Go source files) will be recompiled. Note that changes to files in other directories do not cause the package to be recompiled, so all non-Go source code for the package should be stored in the package directory, not in subdirectories. The default C and C++ compilers may be changed by the CC and CXX environment variables, respectively; those environment variables may include command line options. 

The cgo tool will always invoke the C compiler with the source file's directory in the include path; i.e. -I${SRCDIR} is always implied. This means that if a header file foo/bar.h exists both in the source directory and also in the system include directory (or some other place specified by a -I flag), then "#include <foo/bar.h>" will always find the local version in preference to any other version. 

The cgo tool is enabled by default for native builds on systems where it is expected to work. It is disabled by default when cross-compiling as well as when the CC environment variable is unset and the default C compiler (typically gcc or clang) cannot be found on the system PATH. You can override the default by setting the CGO_ENABLED environment variable when running the go tool: set it to 1 to enable the use of cgo, and to 0 to disable it. The go tool will set the build constraint "cgo" if cgo is enabled. The special import "C" implies the "cgo" build constraint, as though the file also said "//go:build cgo". Therefore, if cgo is disabled, files that import "C" will not be built by the go tool. (For more about build constraints see <https://golang.org/pkg/go/build/#hdr-Build_Constraints>). 

When cross-compiling, you must specify a C cross-compiler for cgo to use. You can do this by setting the generic CC_FOR_TARGET or the more specific CC_FOR_${GOOS}_${GOARCH} (for example, CC_FOR_linux_arm) environment variable when building the toolchain using make.bash, or you can set the CC environment variable any time you run the go tool. 

The CXX_FOR_TARGET, CXX_FOR_${GOOS}_${GOARCH}, and CXX environment variables work in a similar way for C++ code. 

#### Go references to C ¶

Within the Go file, C's struct field names that are keywords in Go can be accessed by prefixing them with an underscore: if x points at a C struct with a field named "type", x._type accesses the field. C struct fields that cannot be expressed in Go, such as bit fields or misaligned data, are omitted in the Go struct, replaced by appropriate padding to reach the next field or the end of the struct. 

The standard C numeric types are available under the names C.char, C.schar (signed char), C.uchar (unsigned char), C.short, C.ushort (unsigned short), C.int, C.uint (unsigned int), C.long, C.ulong (unsigned long), C.longlong (long long), C.ulonglong (unsigned long long), C.float, C.double, C.complexfloat (complex float), and C.complexdouble (complex double). The C type void* is represented by Go's unsafe.Pointer. The C types __int128_t and __uint128_t are represented by [16]byte. 

A few special C types which would normally be represented by a pointer type in Go are instead represented by a uintptr. See the Special cases section below. 

To access a struct, union, or enum type directly, prefix it with struct_, union_, or enum_, as in C.struct_stat. The size of any C type T is available as C.sizeof_T, as in C.sizeof_struct_stat. These special prefixes means that there is no way to directly reference a C identifier that starts with "struct_", "union_", "enum_", or "sizeof_", such as a function named "struct_function". A workaround is to use a "#define" in the preamble, as in "#define c_struct_function struct_function" and then in the Go code refer to "C.c_struct_function". 

A C function may be declared in the Go file with a parameter type of the special name _GoString_. This function may be called with an ordinary Go string value. The string length, and a pointer to the string contents, may be accessed by calling the C functions 
    
    
    size_t _GoStringLen(_GoString_ s);
    const char *_GoStringPtr(_GoString_ s);
    

These functions are only available in the preamble, not in other C files. The C code must not modify the contents of the pointer returned by _GoStringPtr. Note that the string contents may not have a trailing NUL byte. 

As Go doesn't have support for C's union type in the general case, C's union types are represented as a Go byte array with the same length. 

Go structs cannot embed fields with C types. 

Go code cannot refer to zero-sized fields that occur at the end of non-empty C structs. To get the address of such a field (which is the only operation you can do with a zero-sized field) you must take the address of the struct and add the size of the struct. 

Cgo translates C types into equivalent unexported Go types. Because the translations are unexported, a Go package should not expose C types in its exported API: a C type used in one Go package is different from the same C type used in another. 

Any C function (even void functions) may be called in a multiple assignment context to retrieve both the return value (if any) and the C errno variable as an error (use _ to skip the result value if the function returns void). For example: 
    
    
    n, err = C.sqrt(-1)
    _, err := C.voidFunc()
    var n, err = C.sqrt(1)
    

Note that the C errno value may be non-zero, and thus the err result may be non-nil, even if the function call is successful. Unlike normal Go conventions, you should first check whether the call succeeded before checking the error result. For example: 
    
    
    n, err := C.setenv(key, value, 1)
    if n != 0 {
    	// we know the call failed, so it is now valid to use err
    	return err
    }
    

Calling C function pointers is currently not supported, however you can declare Go variables which hold C function pointers and pass them back and forth between Go and C. C code may call function pointers received from Go. For example: 
    
    
    package main
    
    // typedef int (*intFunc) ();
    //
    // int
    // bridge_int_func(intFunc f)
    // {
    //		return f();
    // }
    //
    // int fortytwo()
    // {
    //	    return 42;
    // }
    import "C"
    import "fmt"
    
    func main() {
    	f := C.intFunc(C.fortytwo)
    	fmt.Println(int(C.bridge_int_func(f)))
    	// Output: 42
    }
    

In C, a function argument written as a fixed size array actually requires a pointer to the first element of the array. C compilers are aware of this calling convention and adjust the call accordingly, but Go cannot. In Go, you must pass the pointer to the first element explicitly: C.f(&C.x[0]). 

Calling variadic C functions is not supported. It is possible to circumvent this by using a C function wrapper. For example: 
    
    
    package main
    
    // #include <stdio.h>
    // #include <stdlib.h>
    //
    // static void myprint(char* s) {
    //   printf("%s\n", s);
    // }
    import "C"
    import "unsafe"
    
    func main() {
    	cs := C.CString("Hello from stdio")
    	C.myprint(cs)
    	C.free(unsafe.Pointer(cs))
    }
    

A few special functions convert between Go and C types by making copies of the data. In pseudo-Go definitions: 
    
    
    // Go string to C string
    // The C string is allocated in the C heap using malloc.
    // It is the caller's responsibility to arrange for it to be
    // freed, such as by calling C.free (be sure to include stdlib.h
    // if C.free is needed).
    func C.CString(string) *C.char
    
    // Go []byte slice to C array
    // The C array is allocated in the C heap using malloc.
    // It is the caller's responsibility to arrange for it to be
    // freed, such as by calling C.free (be sure to include stdlib.h
    // if C.free is needed).
    func C.CBytes([]byte) unsafe.Pointer
    
    // C string to Go string
    func C.GoString(*C.char) string
    
    // C data with explicit length to Go string
    func C.GoStringN(*C.char, C.int) string
    
    // C data with explicit length to Go []byte
    func C.GoBytes(unsafe.Pointer, C.int) []byte
    

As a special case, C.malloc does not call the C library malloc directly but instead calls a Go helper function that wraps the C library malloc but guarantees never to return nil. If C's malloc indicates out of memory, the helper function crashes the program, like when Go itself runs out of memory. Because C.malloc cannot fail, it has no two-result form that returns errno. 

#### C references to Go ¶

Go functions can be exported for use by C code in the following way: 
    
    
    //export MyFunction
    func MyFunction(arg1, arg2 int, arg3 string) int64 {...}
    
    //export MyFunction2
    func MyFunction2(arg1, arg2 int, arg3 string) (int64, *C.char) {...}
    

They will be available in the C code as: 
    
    
    extern GoInt64 MyFunction(int arg1, int arg2, GoString arg3);
    extern struct MyFunction2_return MyFunction2(int arg1, int arg2, GoString arg3);
    

found in the _cgo_export.h generated header, after any preambles copied from the cgo input files. Functions with multiple return values are mapped to functions returning a struct. 

Not all Go types can be mapped to C types in a useful way. Go struct types are not supported; use a C struct type. Go array types are not supported; use a C pointer. 

Go functions that take arguments of type string may be called with the C type _GoString_, described above. The _GoString_ type will be automatically defined in the preamble. Note that there is no way for C code to create a value of this type; this is only useful for passing string values from Go to C and back to Go. 

Using //export in a file places a restriction on the preamble: since it is copied into two different C output files, it must not contain any definitions, only declarations. If a file contains both definitions and declarations, then the two output files will produce duplicate symbols and the linker will fail. To avoid this, definitions must be placed in preambles in other files, or in C source files. 

#### Passing pointers ¶

Go is a garbage collected language, and the garbage collector needs to know the location of every pointer to Go memory. Because of this, there are restrictions on passing pointers between Go and C. 

In this section the term Go pointer means a pointer to memory allocated by Go (such as by using the & operator or calling the predefined new function) and the term C pointer means a pointer to memory allocated by C (such as by a call to C.malloc). Whether a pointer is a Go pointer or a C pointer is a dynamic property determined by how the memory was allocated; it has nothing to do with the type of the pointer. 

Note that values of some Go types, other than the type's zero value, always include Go pointers. This is true of interface, channel, map, and function types. A pointer type may hold a Go pointer or a C pointer. Array, slice, string, and struct types may or may not include Go pointers, depending on their type and how they are constructed. All the discussion below about Go pointers applies not just to pointer types, but also to other types that include Go pointers. 

All Go pointers passed to C must point to pinned Go memory. Go pointers passed as function arguments to C functions have the memory they point to implicitly pinned for the duration of the call. Go memory reachable from these function arguments must be pinned as long as the C code has access to it. Whether Go memory is pinned is a dynamic property of that memory region; it has nothing to do with the type of the pointer. 

Go values created by calling new, by taking the address of a composite literal, or by taking the address of a local variable may also have their memory pinned using [runtime.Pinner](/runtime#Pinner). This type may be used to manage the duration of the memory's pinned status, potentially beyond the duration of a C function call. Memory may be pinned more than once and must be unpinned exactly the same number of times it has been pinned. 

Go code may pass a Go pointer to C provided the memory to which it points does not contain any Go pointers to memory that is unpinned. When passing a pointer to a field in a struct, the Go memory in question is the memory occupied by the field, not the entire struct. When passing a pointer to an element in an array or slice, the Go memory in question is the entire array or the entire backing array of the slice. 

C code may keep a copy of a Go pointer only as long as the memory it points to is pinned. 

C code may not keep a copy of a Go pointer after the call returns, unless the memory it points to is pinned with [runtime.Pinner](/runtime#Pinner) and the Pinner is not unpinned while the Go pointer is stored in C memory. This implies that C code may not keep a copy of a string, slice, channel, and so forth, because they cannot be pinned with [runtime.Pinner](/runtime#Pinner). 

The _GoString_ type also may not be pinned with [runtime.Pinner](/runtime#Pinner). Because it includes a Go pointer, the memory it points to is only pinned for the duration of the call; _GoString_ values may not be retained by C code. 

A Go function called by C code may return a Go pointer to pinned memory (which implies that it may not return a string, slice, channel, and so forth). A Go function called by C code may take C pointers as arguments, and it may store non-pointer data, C pointers, or Go pointers to pinned memory through those pointers. It may not store a Go pointer to unpinned memory in memory pointed to by a C pointer (which again, implies that it may not store a string, slice, channel, and so forth). A Go function called by C code may take a Go pointer but it must preserve the property that the Go memory to which it points (and the Go memory to which that memory points, and so on) is pinned. 

These rules are checked dynamically at runtime. The checking is controlled by the cgocheck setting of the GODEBUG environment variable. The default setting is GODEBUG=cgocheck=1, which implements reasonably cheap dynamic checks. These checks may be disabled entirely using GODEBUG=cgocheck=0. Complete checking of pointer handling, at some cost in run time, is available by setting GOEXPERIMENT=cgocheck2 at build time. 

It is possible to defeat this enforcement by using the unsafe package, and of course there is nothing stopping the C code from doing anything it likes. However, programs that break these rules are likely to fail in unexpected and unpredictable ways. 

The runtime/cgo.Handle type can be used to safely pass Go values between Go and C. See the runtime/cgo package documentation for details. 

Note: the current implementation has a bug. While Go code is permitted to write nil or a C pointer (but not a Go pointer) to C memory, the current implementation may sometimes cause a runtime error if the contents of the C memory appear to be a Go pointer. Therefore, avoid passing uninitialized C memory to Go code if the Go code is going to store pointer values in it. Zero out the memory in C before passing it to Go. 

#### Optimizing calls of C code ¶

When passing a Go pointer to a C function the compiler normally ensures that the Go object lives on the heap. If the C function does not keep a copy of the Go pointer, and never passes the Go pointer back to Go code, then this is unnecessary. The #cgo noescape directive may be used to tell the compiler that no Go pointers escape via the named C function. If the noescape directive is used and the C function does not handle the pointer safely, the program may crash or see memory corruption. 

For example: 
    
    
    // #cgo noescape cFunctionName
    

When a Go function calls a C function, it prepares for the C function to call back to a Go function. The #cgo nocallback directive may be used to tell the compiler that these preparations are not necessary. If the nocallback directive is used and the C function does call back into Go code, the program will panic. 

For example: 
    
    
    // #cgo nocallback cFunctionName
    

#### Special cases ¶

A few special C types which would normally be represented by a pointer type in Go are instead represented by a uintptr. Those include: 

1\. The *Ref types on Darwin, rooted at CoreFoundation's CFTypeRef type. 

2\. The object types from Java's JNI interface: 
    
    
    jobject
    jclass
    jthrowable
    jstring
    jarray
    jbooleanArray
    jbyteArray
    jcharArray
    jshortArray
    jintArray
    jlongArray
    jfloatArray
    jdoubleArray
    jobjectArray
    jweak
    

3\. The EGLDisplay and EGLConfig types from the EGL API. 

These types are uintptr on the Go side because they would otherwise confuse the Go garbage collector; they are sometimes not really pointers but data structures encoded in a pointer type. All operations on these types must happen in C. The proper constant to initialize an empty such reference is 0, not nil. 

These special cases were introduced in Go 1.10. For auto-updating code from Go 1.9 and earlier, use the cftype or jni rewrites in the Go fix tool: 
    
    
    go tool fix -r cftype <pkg>
    go tool fix -r jni <pkg>
    

It will replace nil with 0 in the appropriate places. 

The EGLDisplay case was introduced in Go 1.12. Use the egl rewrite to auto-update code from Go 1.11 and earlier: 
    
    
    go tool fix -r egl <pkg>
    

The EGLConfig case was introduced in Go 1.15. Use the eglconf rewrite to auto-update code from Go 1.14 and earlier: 
    
    
    go tool fix -r eglconf <pkg>
    

#### Using cgo directly ¶

Usage: 
    
    
    go tool cgo [cgo options] [-- compiler options] gofiles...
    

Cgo transforms the specified input Go source files into several output Go and C source files. 

The compiler options are passed through uninterpreted when invoking the C compiler to compile the C parts of the package. 

The following options are available when running cgo directly: 
    
    
    -V
    	Print cgo version and exit.
    -debug-define
    	Debugging option. Print #defines.
    -debug-gcc
    	Debugging option. Trace C compiler execution and output.
    -dynimport file
    	Write list of symbols imported by file. Write to
    	-dynout argument or to standard output. Used by go
    	build when building a cgo package.
    -dynlinker
    	Write dynamic linker as part of -dynimport output.
    -dynout file
    	Write -dynimport output to file.
    -dynpackage package
    	Set Go package for -dynimport output.
    -exportheader file
    	If there are any exported functions, write the
    	generated export declarations to file.
    	C code can #include this to see the declarations.
    -gccgo
    	Generate output for the gccgo compiler rather than the
    	gc compiler.
    -gccgoprefix prefix
    	The -fgo-prefix option to be used with gccgo.
    -gccgopkgpath path
    	The -fgo-pkgpath option to be used with gccgo.
    -gccgo_define_cgoincomplete
    	Define cgo.Incomplete locally rather than importing it from
    	the "runtime/cgo" package. Used for old gccgo versions.
    -godefs
    	Write out input file in Go syntax replacing C package
    	names with real values. Used to generate files in the
    	syscall package when bootstrapping a new target.
    -importpath string
    	The import path for the Go package. Optional; used for
    	nicer comments in the generated files.
    -import_runtime_cgo
    	If set (which it is by default) import runtime/cgo in
    	generated output.
    -import_syscall
    	If set (which it is by default) import syscall in
    	generated output.
    -ldflags flags
    	Flags to pass to the C linker. The cmd/go tool uses
    	this to pass in the flags in the CGO_LDFLAGS variable.
    -objdir directory
    	Put all generated files in directory.
    -srcdir directory
    	Find the Go input files, listed on the command line,
    	in directory.
    -trimpath rewrites
    	Apply trims and rewrites to source file paths.
    

##  ![](/static/shared/icon/insert_drive_file_gm_grey_24dp.svg) Source Files ¶

[View all Source files](https://cs.opensource.google/go/go/+/go1.24.5:src/cmd/cgo)

  * [ast.go](https://cs.opensource.google/go/go/+/go1.24.5:src/cmd/cgo/ast.go "ast.go")
  * [ast_go118.go](https://cs.opensource.google/go/go/+/go1.24.5:src/cmd/cgo/ast_go118.go "ast_go118.go")
  * [doc.go](https://cs.opensource.google/go/go/+/go1.24.5:src/cmd/cgo/doc.go "doc.go")
  * [gcc.go](https://cs.opensource.google/go/go/+/go1.24.5:src/cmd/cgo/gcc.go "gcc.go")
  * [godefs.go](https://cs.opensource.google/go/go/+/go1.24.5:src/cmd/cgo/godefs.go "godefs.go")
  * [main.go](https://cs.opensource.google/go/go/+/go1.24.5:src/cmd/cgo/main.go "main.go")
  * [out.go](https://cs.opensource.google/go/go/+/go1.24.5:src/cmd/cgo/out.go "out.go")
  * [util.go](https://cs.opensource.google/go/go/+/go1.24.5:src/cmd/cgo/util.go "util.go")



##  ![](/static/shared/icon/folder_gm_grey_24dp.svg) Directories ¶

Show internal  Expand all 

Path | Synopsis  
---|---  
![](/static/shared/icon/arrow_right_gm_grey_24dp.svg) internal |   
[cgotest](/cmd/cgo/internal/cgotest@go1.24.5) |  |  [test](/cmd/cgo/internal/test@go1.24.5) Test that pthread_cancel works as expected (NPTL uses SIGRTMIN to implement thread cancellation) See https://golang.org/issue/6997 | Test that pthread_cancel works as expected (NPTL uses SIGRTMIN to implement thread cancellation) See https://golang.org/issue/6997 |  [test/gcc68255](/cmd/cgo/internal/test/gcc68255@go1.24.5) |  |  [test/issue23555a](/cmd/cgo/internal/test/issue23555a@go1.24.5) |  |  [test/issue23555b](/cmd/cgo/internal/test/issue23555b@go1.24.5) |  |  [test/issue24161arg](/cmd/cgo/internal/test/issue24161arg@go1.24.5) |  |  [test/issue24161e0](/cmd/cgo/internal/test/issue24161e0@go1.24.5) |  |  [test/issue24161e1](/cmd/cgo/internal/test/issue24161e1@go1.24.5) |  |  [test/issue24161e2](/cmd/cgo/internal/test/issue24161e2@go1.24.5) |  |  [test/issue24161res](/cmd/cgo/internal/test/issue24161res@go1.24.5) |  |  [test/issue26213](/cmd/cgo/internal/test/issue26213@go1.24.5) |  |  [test/issue26430](/cmd/cgo/internal/test/issue26430@go1.24.5) |  |  [test/issue26743](/cmd/cgo/internal/test/issue26743@go1.24.5) |  |  [test/issue27054](/cmd/cgo/internal/test/issue27054@go1.24.5) |  |  [test/issue27340](/cmd/cgo/internal/test/issue27340@go1.24.5) |  |  [test/issue29563](/cmd/cgo/internal/test/issue29563@go1.24.5) |  |  [test/issue30527](/cmd/cgo/internal/test/issue30527@go1.24.5) |  |  [test/issue41761a](/cmd/cgo/internal/test/issue41761a@go1.24.5) |  |  [test/issue43639](/cmd/cgo/internal/test/issue43639@go1.24.5) |  |  [test/issue52611a](/cmd/cgo/internal/test/issue52611a@go1.24.5) |  |  [test/issue52611b](/cmd/cgo/internal/test/issue52611b@go1.24.5) |  |  [test/issue8756](/cmd/cgo/internal/test/issue8756@go1.24.5) |  |  [test/issue8828](/cmd/cgo/internal/test/issue8828@go1.24.5) |  |  [test/issue9026](/cmd/cgo/internal/test/issue9026@go1.24.5) |  |  [test/issue9400](/cmd/cgo/internal/test/issue9400@go1.24.5) |  |  [test/issue9510a](/cmd/cgo/internal/test/issue9510a@go1.24.5) |  |  [test/issue9510b](/cmd/cgo/internal/test/issue9510b@go1.24.5) |  |  [testnocgo](/cmd/cgo/internal/testnocgo@go1.24.5) |  |  [testtls](/cmd/cgo/internal/testtls@go1.24.5) |   
  
Click to show internal directories. 

Click to hide internal directories.