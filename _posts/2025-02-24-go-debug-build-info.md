---
title: 'Build information in Go'
tags: [Go]
---

## Build information in Go

 - [Build info of the currently running Go binary](#build-info-of-the-currently-running-go-binary)
 - [Build info structure](#build-info-structure)
 - [Build info of Go binary file](#build-info-of-go-binary-file)
 - [An example](#an-example)

---

---

Go offers a few simple methods to handle with build information header embedded into each Go binary built with
support of Go modules. All those methods can be found in the 
[runtime/debug](https://pkg.go.dev/runtime/debug@go1.24.0#BuildInfo) package.

---

### Build info of the currently running Go binary

Imagine, that you need to publish the version of the currently executed Go binary.
This requirement might come from need to publish application version via RestAPI or to include this
precious information in help of your small commandline tool. It is as simple as running 
`func ReadBuildInfo() (info *BuildInfo, ok bool)` function from above-mentioned package
[runtime/debug](https://pkg.go.dev/runtime/debug@go1.24.0#BuildInfo).

As one can see, it returns a pointer for `BuildInfo` structure, which is described by the following
section.

---

### Build info structure

There follows the code snippet with definition of the `BuildInfo` structure and it's  

```go
type BuildInfo struct {
	// GoVersion is the version of the Go toolchain that built the binary
	// (for example, "go1.19.2").
	GoVersion string

	// Path is the package path of the main package for the binary
	// (for example, "golang.org/x/tools/cmd/stringer").
	Path string

	// Main describes the module that contains the main package for the binary.
	Main Module

	// Deps describes all the dependency modules, both direct and indirect,
	// that contributed packages to the build of this binary.
	Deps []*Module

	// Settings describes the build settings used to build the binary.
	Settings []BuildSetting
}

type Module struct {
	Path    string  // module path
	Version string  // module version
	Sum     string  // checksum
	Replace *Module // replaced by this module
}

type BuildSetting struct {
	// Key and Value describe the build setting.
	// Key must not contain an equals sign, space, tab, or newline.
	// Value must not contain newlines ('\n').
	Key, Value string
}
```

It contains a lot of potentially useful pieced of information, e.g.:
 - `GoVersion` tells the exact version of Go used to build that binary;
 - `Path` returns the name defined within the module's `go.mod` file;
 - `Main.Vesion` returns the version extracted from your git commit tag;
 - `Settings` slice should contain a flag with key of `vcs.modified` (since Go v1.24) and some more useful items.

---

### Build info of Go binary file

One might need to retrieve build information embedded into any Go binary file at the given file path
(denoted by the only parameter of that function). One might use 
[`debug/buildinfo.Read`](https://pkg.go.dev/debug/buildinfo@go1.24.0#Read) function and proces its
content as needed afterward.

As the official documentation states: _Most information is only available for binaries built with module support._

---

### An example

If one calls the following function:

```go
func printBuildInfo() error {
	const (
		width  = 20
		format = "%-[1]*[2]s: %[3]v\n"
	)

	bi, ok := debug.ReadBuildInfo()
	if !ok {
		return errors.New("print build info failed (probably built without module support)")
	}

	fmt.Printf(format, width, "Go version", bi.GoVersion)
	fmt.Printf(format, width, "main path", bi.Main.Path)
	fmt.Printf(format, width, "main version", bi.Main.Version)
	fmt.Printf(format, width, "main sum", bi.Main.Sum)
	for _, set := range bi.Settings {
		fmt.Printf(format, width, "set "+set.Key, set.Value)
	}

	return nil
}
```

one might get output on stdout that is similar to the following one:

```
Go version          : go1.24.0
main path           : github.com/Radek-Pysny/xyz
main version        : v1.2.0+dirty
main sum            : 
set -buildmode      : exe
set -compiler       : gc
set CGO_ENABLED     : 1
set CGO_CFLAGS      : 
set CGO_CPPFLAGS    : 
set CGO_CXXFLAGS    : 
set CGO_LDFLAGS     : 
set GOARCH          : amd64
set GOOS            : linux
set GOAMD64         : v1
set vcs             : git
set vcs.revision    : 774f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f00
set vcs.time        : 2025-02-24T08:46:17Z
set vcs.modified    : true
```

The important part to note, is visible on lines with prefix `main version` and `set vcs.modified`.
The `+dirty` suffix and `true` value are both presenting that the build was done with some not yet
commited modification on top of the commit with ID of `774f4f4f...` that was tagged by `v1.2.0`.

This might be a really important piece of information.
