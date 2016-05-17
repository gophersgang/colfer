# Colfer [![Build Status](https://travis-ci.org/pascaldekloe/colfer.svg?branch=master)](https://travis-ci.org/pascaldekloe/colfer)

Colfer is a schema-based binary data format optimized for speed and size.

The project's compiler `colf(1)` generates source code from schema definitions
to marshal and unmarshall data structures.

This is free and unencumbered software released into the [public domain](http://creativecommons.org/publicdomain/zero/1.0).
The format is inspired by Proto**col** Buf**fer**.


#### Features

* Simple and straightforward in use
* Support for: Go, Java and ECMAScript/JavaScript
* No dependencies other than the core library
* Both faster and smaller than: Protocol Buffers, FlatBuffers and MessagePack
* The generated code is human-readable
* Compiler runs on [many platforms](https://golang.org/doc/install/source#environment)
* Configurable data limits with sane defaults (memory protection)
* Maximum of 127 fields per data structure
* No support for enumerations

#### TODO's

* RMI
* Arrays for non structs.



## Use

Download a [prebuilt compiler](https://github.com/pascaldekloe/colfer/releases)
or run `go get -u github.com/pascaldekloe/colfer/cmd/colf` to make one yourself.

```
NAME
	colf — compile Colfer schemas

SYNOPSIS
	colf [-b <dir>] [-p <path>] <language> [<file> ...]

DESCRIPTION
	Generates source code for the given language. The options are: Go,
	Java and ECMAScript.
	The file operands are processed in command-line order. If file is
	absent, colf reads all ".colf" files in the working directory.

  -b string
	Use a specific destination base directory. (default ".")
  -p string
	Adds a package prefix. Use slash as a separator when nesting.

BUGS
	Report bugs at https://github.com/pascaldekloe/colfer/issues

SEE ALSO
	protoc(1)
```


It is recommended to commit the generated source code to the respective version
control.
Maven developers may [disagree](https://github.com/pascaldekloe/colfer/wiki/Java).



## Schema

Data structures are defined per package in `.colf` files. The format is quite
self explanatory.

```
package example

type member struct {
	id     int64
	name   text
	joined timestamp
	avatar binary
	allies []member
}
```


The following table shows how Colfer data types are applied per language.

| Colfer	| ECMAScript		| Go		| Java		|
|:--------------|:----------------------|:--------------|:--------------|
| bool		| Boolean		| bool		| boolean	|
| uint32	| Number		| uint32	| int		|
| uint64	| Number †		| uint64	| long		|
| int32		| Number		| int32		| int		|
| int64		| Number †		| int64		| long		|
| float32	| Number		| float32	| float		|
| float64	| Number		| float64	| double	|
| timestamp	| Date + Number	‡	| time.Time	| java.time.Instant |
| text		| String		| string	| java.lang.String |
| binary	| Uint8Array		| []byte	| byte[]	|

* † range limited to (-2⁵³ + 1, 2⁵³ - 1)
* ‡ range limited to (`1970-01-01T00:00:00.000000000Z`, `287396-10-12T08:59:00.991999999`)



## Compatibility

Name changes do not affect the serialization format. Deprecated fields can be
renamed to clearly discourage its use.

The following changes are backward compatible.
* Adding new fields to the end of Colfer structs
* Raising the bit size (which is actually just a limit) of signed integers



## Performance

```
% go test -bench .
PASS
BenchmarkMarshal-4               	10000000	       128 ns/op	      52 B/op	       1 allocs/op
BenchmarkMarshalProtoBuf-4       	10000000	       138 ns/op	      52 B/op	       1 allocs/op
BenchmarkMarshalFlatBuf-4        	 1000000	      1355 ns/op	     472 B/op	      12 allocs/op
BenchmarkUnmarshal-4             	10000000	       157 ns/op	      84 B/op	       2 allocs/op
BenchmarkUnmarshalProtoBuf-4     	10000000	       199 ns/op	      84 B/op	       2 allocs/op
BenchmarkUnmarshalFlatBuf-4      	 5000000	       265 ns/op	      84 B/op	       2 allocs/op
BenchmarkMarshalReuse-4          	30000000	        53.5 ns/op	       0 B/op	       0 allocs/op
BenchmarkMarshalProtoBufReuse-4  	20000000	        66.1 ns/op	       0 B/op	       0 allocs/op
BenchmarkMarshalFlatBufReuse-4   	 5000000	       390 ns/op	       0 B/op	       0 allocs/op
BenchmarkUnmarshalReuse-4        	20000000	        94.3 ns/op	      20 B/op	       1 allocs/op
BenchmarkUnmarshalProtoBufReuse-4	10000000	       145 ns/op	      20 B/op	       1 allocs/op
BenchmarkUnmarshalFlatBufReuse-4 	10000000	       211 ns/op	      20 B/op	       1 allocs/op
ok  	github.com/pascaldekloe/colfer	21.177s
```

For Java the numbers look even better.

```
Running testdata.bench.bench
20M unmarshals avg 77ns
20M marshals avg 60ns
20M marshals with buffer reuse avg 39ns
```


## Format

Data structures consist of zero or more field *value definitions* followed by a
termination byte `0x7f`. Only those fields with a value other than the *zero
value* may be serialized. Fields appear in order as stated by the schema.

The zero value for booleans is `false`, integers: `0`, floating points: `0.0`,
timestamps: `1970-01-01T00:00:00.000000000Z`, text & binary: the empty
string, nested data structures: `null` and an empty list for data structure
lists.

Data is represented in a big-endian manner. The format relies on *varints* also
known as a
[variable-length quantity](https://en.wikipedia.org/wiki/Variable-length_quantity).


#### Value Definiton

Each definition starts with an 8-bit header. The 7 least significant bits
identify the field by its (0-based position) index in the schema. The most
significant bit is used as a *flag*.

Boolean occurrences set the value to `true`.

Integers are encoded as varints. The header flag indicates negative for signed
types and fixed size for unsigned types. The tenth byte for 64-bit integers is
skipped for encoding since its value is fixed to `0x01`.

Floating points are encoded conform IEEE 754.

Timestamps are encoded as a 32-bit unsigned integer for the number of seconds
that have elapsed since 00:00:00 UTC, Thursday, 1 January 1970, not counting
leap seconds. When the header flag is set then the number of seconds is encoded
as a 64-bit two's complement integer. In both cases the value is followed with
32 bits for the nanosecond fraction. Note that the first two bits are not in use
(reserved).

The data for text and binaries is prefixed with a varint byte size declaration.
Text is encoded as UTF-8.

Object arrays are also prefixed with a varint size declaration.
