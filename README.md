[![Build Status](https://travis-ci.com/fxamacker/cbor.svg?branch=master)](https://travis-ci.com/fxamacker/cbor)
[![codecov](https://codecov.io/gh/fxamacker/cbor/branch/master/graph/badge.svg?v=4)](https://codecov.io/gh/fxamacker/cbor)
[![Go Report Card](https://goreportcard.com/badge/github.com/fxamacker/cbor)](https://goreportcard.com/report/github.com/fxamacker/cbor)
[![Release](https://img.shields.io/github/release/fxamacker/cbor.svg?style=flat-square)](https://github.com/fxamacker/cbor/releases)
[![GoDoc](http://img.shields.io/badge/go-documentation-blue.svg?style=flat-square)](http://godoc.org/github.com/fxamacker/cbor)
[![License](http://img.shields.io/badge/license-mit-blue.svg?style=flat-square)](https://raw.githubusercontent.com/fxamacker/cbor/master/LICENSE)

# cbor - CBOR encoding and decoding in Go

CBOR is a concise binary alternative to JSON.  This library makes CBOR easy to use.

This library is designed to be:
* __Easy__ -- idiomatic API like `encoding/json`.
* __Safe and reliable__ -- no `unsafe` pkg, test coverage at 96%, and 10+ hrs of fuzzing with RFC 7049 tests as seed.
* __Standards-compliant__ -- supports [RFC 7049](https://tools.ietf.org/html/rfc7049) and canonical CBOR encodings (both [RFC 7049](https://tools.ietf.org/html/rfc7049#section-3.9) and [CTAP2](https://fidoalliance.org/specs/fido-v2.0-id-20180227/fido-client-to-authenticator-protocol-v2.0-id-20180227.html#ctap2-canonical-cbor-encoding-form)).
* __Small and self-contained__ -- pkg compiles to under 0.5 MB with no external dependencies.

`cbor` balances speed, safety (no `unsafe` pkg) and compiled size.  To keep size small, it doesn't use code generation.  For speed, it caches struct field types, bypasses `reflect` when appropriate, and uses `sync.Pool` to reuse transient objects.  

## Current status

Oct 7, 2019: `cbor` v1.1 is released.  It passed 10+ hours of fuzzing on linux_amd64 with Go 1.12.  

## Size comparison

Program size comparison (linux_amd64, Go 1.12) doing the same CBOR encoding and decoding:
- 2.6 MB program using fxamacker/cbor
- 11.9 MB program using ugorji/go

Library size comparison (linux_amd64, Go 1.12):
- 0.42 MB pkg -- fxamacker/cbor
- 2.9 MB pkg -- ugorji/go without code generation (`go install --tags "notfastpath"`)
- 5.7 MB pkg -- ugorji/go with code generation (default build)

## Features

* Idiomatic API as in `json` package.
* No external dependencies.
* No use of `unsafe` package.
* Tested with [RFC 7049 test vectors](https://tools.ietf.org/html/rfc7049#appendix-A).
* Test coverage at 96%, and fuzzed 10+ hours using [cbor-fuzz](https://github.com/fxamacker/cbor-fuzz).
* Decode slices, maps, and structs in-place.
* Decode into struct with field name case-insensitive match.
* Support canonical CBOR encoding for map/struct.
* Support both "cbor" and "json" keys for struct field format tags.
* Encode anonymous struct fields by `json` package struct fields visibility rules.
* Encode and decode nil slice/map/pointer/interface values correctly.
* Encode and decode indefinite length bytes/string/array/map (["streaming"](https://tools.ietf.org/html/rfc7049#section-2.2)).
* Encode and decode time.Time as RFC 3339 formatted text string or Unix time.
* Support encoding.BinaryMarshaler and encoding.BinaryUnmarshaler interfaces.

## Standards 

This library implements CBOR as specified in [RFC 7049](https://tools.ietf.org/html/rfc7049), with minor [limitations](#limitations).

It also supports canonical CBOR encodings (both [RFC 7049](https://tools.ietf.org/html/rfc7049#section-3.9) and [CTAP2](https://fidoalliance.org/specs/fido-v2.0-id-20180227/fido-client-to-authenticator-protocol-v2.0-id-20180227.html#ctap2-canonical-cbor-encoding-form)).  CTAP2 canonical CBOR encoding is used by [CTAP](https://fidoalliance.org/specs/fido-v2.0-id-20180227/fido-client-to-authenticator-protocol-v2.0-id-20180227.html) and [WebAuthn](https://www.w3.org/TR/webauthn/) in [FIDO2](https://fidoalliance.org/fido2/) framework.

## Limitations

* CBOR tags (type 6) are ignored.  Decoder simply decodes tagged data after ignoring the tags.
* Signed integer values incompatible with Go's int64 are not supported.  So RFC 7049 test vectors incompatible with Go's int64 are skipped. For example, test result -18446744073709551616 can't fit into int64.

## Versions and API changes

This project uses [Semantic Versioning](https://semver.org), so the API is always backwards compatible unless the major version number changes.

## API 

See [API docs](https://godoc.org/github.com/fxamacker/cbor) for more details.

```
package cbor // import "github.com/fxamacker/cbor"

func Marshal(v interface{}, encOpts EncOptions) ([]byte, error)
func Unmarshal(data []byte, v interface{}) error
func Valid(data []byte) (rest []byte, err error)
type Decoder struct{ ... }
    func NewDecoder(r io.Reader) *Decoder
    func (dec *Decoder) Decode(v interface{}) (err error)
    func (dec *Decoder) NumBytesRead() int
type EncOptions struct{ ... }
type Encoder struct{ ... }
    func NewEncoder(w io.Writer, encOpts EncOptions) *Encoder
    func (enc *Encoder) Encode(v interface{}) error
    func (enc *Encoder) StartIndefiniteByteString() error
    func (enc *Encoder) StartIndefiniteTextString() error
    func (enc *Encoder) StartIndefiniteArray() error
    func (enc *Encoder) StartIndefiniteMap() error
    func (enc *Encoder) EndIndefinite() error
type InvalidUnmarshalError struct{ ... }
type SemanticError struct{ ... }
type SyntaxError struct{ ... }
type UnmarshalTypeError struct{ ... }
type UnsupportedTypeError struct{ ... }
```

## Installation 

```
go get github.com/fxamacker/cbor
```

## Usage

See [examples](example_test.go).

Using decoder:

```
// create a decoder
dec := cbor.NewDecoder(reader)

// decode into empty interface
var i interface{}
err = dec.Decode(&i)

// decode into struct 
var stru ExampleStruct
err = dec.Decode(&stru)

// decode into map
var m map[string]string
err = dec.Decode(&m)

// decode into primitives
var f float32
err = dec.Decode(&f)
```

Using encoder:

```
// create an encoder with canonical CBOR encoding enabled
enc := cbor.NewEncoder(writer, cbor.EncOptions{Canonical: true})

// encode struct
err = enc.Encode(stru)

// encode map
err = enc.Encode(m)

// encode primitives
err = enc.Encode(f)
```

Encoding indefinite length array:

```
enc := cbor.NewEncoder(writer, cbor.EncOptions{})

// start indefinite length array encoding
err = enc.StartIndefiniteArray()

// encode array element
err = enc.Encode(1)

// encode array element
err = enc.Encode([]int{2, 3})

// start nested indefinite length array as array element
err = enc.StartIndefiniteArray()

// encode nested array element
err = enc.Encode(4)

// encode nested array element
err = enc.Encode(5)

// close nested indefinite length array
err = enc.EndIndefinite()

// close outer indefinite length array
err = enc.EndIndefinite()
```

## Benchmarks

See [bench_test.go](bench_test.go).

`Unmarshal` benchmarks are made on CBOR data representing the following values:

* Boolean: `true`
* Positive integer: `18446744073709551615`
* Negative integer: `-1000`
* Float: `-4.1`
* Byte string: `h'0102030405060708090a0b0c0d0e0f101112131415161718191a'`
* Text string: `"The quick brown fox jumps over the lazy dog"`
* Array: `[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26]`
* Map: `{"a": "A", "b": "B", "c": "C", "d": "D", "e": "E", "f": "F", "g": "G", "h": "H", "i": "I", "j": "J", "l": "L", "m": "M", "n": "N"}}`

`Marshal` benchmarks are made on Go values representing the same values.

Benchmarks shows that decoding into struct is >56% faster than decoding into map, and encoding struct is >74% faster than encoding map.  

`Unmarshal` Benchmark | Time | Bytes | Allocs 
--- | ---: | ---: | ---:
BenchmarkUnmarshal/CBOR_bool_to_Go_interface_{}-2 | 119 ns/op | 16 B/op | 1 allocs/op
BenchmarkUnmarshal/CBOR_bool_to_Go_bool-2 | 66.8 ns/op | 1 B/op | 1 allocs/op
BenchmarkUnmarshal/CBOR_positive_int_to_Go_interface_{}-2 | 142 ns/op | 24 B/op | 2 allocs/op
BenchmarkUnmarshal/CBOR_positive_int_to_Go_uint64-2 | 71.4 ns/op | 8 B/op | 1 allocs/op
BenchmarkUnmarshal/CBOR_negative_int_to_Go_interface_{}-2 | 140 ns/op | 24 B/op | 2 allocs/op
BenchmarkUnmarshal/CBOR_negative_int_to_Go_int64-2 | 73.9 ns/op | 8 B/op | 1 allocs/op
BenchmarkUnmarshal/CBOR_float_to_Go_interface_{}-2 | 145 ns/op | 24 B/op | 2 allocs/op
BenchmarkUnmarshal/CBOR_float_to_Go_float64-2 | 71.9 ns/op | 8 B/op | 1 allocs/op
BenchmarkUnmarshal/CBOR_bytes_to_Go_interface_{}-2 | 187 ns/op | 80 B/op | 3 allocs/op
BenchmarkUnmarshal/CBOR_bytes_to_Go_[]uint8-2 | 158 ns/op | 64 B/op | 2 allocs/op
BenchmarkUnmarshal/CBOR_text_to_Go_interface_{}-2 | 215 ns/op | 80 B/op | 3 allocs/op
BenchmarkUnmarshal/CBOR_text_to_Go_string-2 | 159 ns/op | 64 B/op | 2 allocs/op
BenchmarkUnmarshal/CBOR_array_to_Go_interface_{}-2 |1122 ns/op | 672 B/op | 29 allocs/op
BenchmarkUnmarshal/CBOR_array_to_Go_[]int-2 | 1025 ns/op | 272 B/op | 3 allocs/op
BenchmarkUnmarshal/CBOR_map_to_Go_interface_{}-2 | 3043 ns/op | 1420 B/op | 30 allocs/op
BenchmarkUnmarshal/CBOR_map_to_Go_map[string]interface_{}-2 | 3857 ns/op | 964 B/op | 19 allocs/op
BenchmarkUnmarshal/CBOR_map_to_Go_map[string]string-2 | 2647 ns/op | 741 B/op | 5 allocs/op
BenchmarkUnmarshal/CBOR_map_to_Go_struct-2 | 1672 ns/op| 208 B/op | 1 allocs/op

`Marshal` Benchmark | Time | Bytes | Allocs 
--- | ---: | ---: | ---:
BenchmarkMarshal/Go_bool_to_CBOR_bool-2 | 75.0 ns/op	| 1 B/op | 1 allocs/op
BenchmarkMarshal/Go_uint64_to_CBOR_positive_int-2 | 86.1 ns/op | 16 B/op | 1 allocs/op
BenchmarkMarshal/Go_int64_to_CBOR_negative_int-2 | 81.4 ns/op | 3 B/op | 1 allocs/op
BenchmarkMarshal/Go_float64_to_CBOR_float-2 | 85.4 ns/op	| 16 B/op | 1 allocs/op
BenchmarkMarshal/Go_[]uint8_to_CBOR_bytes-2 | 119 ns/op | 32 B/op	| 1 allocs/op
BenchmarkMarshal/Go_string_to_CBOR_text-2 | 105 ns/op | 48 B/op | 1 allocs/op
BenchmarkMarshal/Go_[]int_to_CBOR_array-2 | 561 ns/op | 32 B/op	| 1 allocs/op
BenchmarkMarshal/Go_map[string]string_to_CBOR_map-2 | 3122 ns/op | 576 B/op | 28 allocs/op
BenchmarkMarshal/Go_struct_to_CBOR_map-2 | 785 ns/op | 64 B/op | 1 allocs/op

## License 

Copyright (c) 2019 [Faye Amacker](https://github.com/fxamacker)

Licensed under [MIT License](LICENSE)
