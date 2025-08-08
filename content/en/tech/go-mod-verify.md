---
author: "Harshanu"
title: "How go verifies checksums of downloaded modules?"
date: 2023-11-30
description: "A Guide to help the go developer know how go verifies checksums of downloade modules"
tags: ["go", "golang", "checksum", "base64", "fingerprint", "sha", "sha256sum", "module"]
thumbnail: https://photos.harshanu.space/api/v1/t/6912fb85590d93cd40d452915c80112ba9e5c3a7/2zwabhu7/fit_1280
---

## Introduction
Checksums play a crucial role in ensuring the integrity of data by providing a unique fingerprint for a given set of information. Whether you're a developer, system administrator, or just a curious individual, understanding how to calculate checksums manually can be a valuable skill. In this guide, we'll explore the concept of checksums and walk through the steps of manually calculating them & compare with go.sum file.

## What is a Checksum?
A checksum is a value derived from the content of a file or data, typically generated using a specific algorithm. Its primary purpose is to verify the integrity of the data by comparing the checksum generated before and after data transfer or storage. If the checksums match, the data is assumed to be intact; otherwise, it may have been corrupted.

## Common Checksum Algorithms
Several checksum algorithms are commonly used, each with its strengths and weaknesses. Some of the widely used algorithms include:

1. **MD5 (Message Digest Algorithm 5)**: Generates a 128-bit hash value, commonly expressed as a 32-character hexadecimal number.

2. **SHA-1 (Secure Hash Algorithm 1)**: Produces a 160-bit hash value, often represented as a 40-character hexadecimal number. Note that SHA-1 is considered insecure for cryptographic purposes.

3. **SHA-256 (Secure Hash Algorithm 256-bit)**: A member of the SHA-2 family, it produces a 256-bit hash value, usually represented as a 64-character hexadecimal number.

## How go computes the checksums
Go uses `SHA-2` algorithm to calculate checksums of downloaded modules. It gets triggered during various triggers such as `go mod tidy`, `go mod verify`, `go mod vendor` just to name few. According to the documentation [here](https://cs.opensource.google/go/x/mod/+/master:sumdb/dirhash/hash.go), below actions are performed ::

{{< highlight go >}}
// Hash1 is "h1:" followed by the base64-encoded SHA-256 hash of a summary
// prepared as if by the Unix command:
//
//	sha256sum $(find . -type f | sort) | sha256sum
//
// More precisely, the hashed summary contains a single line for each file in the list,
// ordered by sort.Strings applied to the file names, where each line consists of
// the hexadecimal SHA-256 hash of the file content,
// two spaces (U+0020), the file name, and a newline (U+000A).
//
// File names with newlines (U+000A) are disallowed.

{{< /highlight >}}

The above documentation is very clear. But it took more than a day to verify manually. This has been answered by me on [StackOverflow](https://stackoverflow.com/a/77579470/3405980). 

A typical `go.sum` file looks like below ::
{{< highlight go >}}
github.com/gorilla/websocket v1.5.1 h1:gmztn0JnHVt9JZquRuzLw3g4wouNVzKL15iLr/zn/QY=
github.com/gorilla/websocket v1.5.1/go.mod h1:x3kM2JMyaluk02fnUJpQuwD2dCS5NDG2ZHL0uE0tcaY=
golang.org/x/crypto v0.16.0 h1:mMMrFzRSCF0GvB7Ne27XVtVAaXLrPmgPC7/v0tkwHaY=
golang.org/x/crypto v0.16.0/go.mod h1:gCAAfMLgwOJRpTjQ2zCCt2OcSfYMTeZVSRtQlPC7Nq4=
golang.org/x/net v0.17.0 h1:pVaXccu2ozPjCXewfr1S7xza/zcXTity9cCdXQYSjIM=
golang.org/x/net v0.17.0/go.mod h1:NxSsAGuq816PNPmqtQdLE42eU2Fs7NoRIZrHJAlaCOE=
golang.org/x/sys v0.15.0 h1:h48lPFYpsTvQJZF4EKyI4aLHaev3CxivZmv7yZig9pc=
golang.org/x/sys v0.15.0/go.mod h1:/VUhepiaJMQUp4+oa/7Zr1D23ma6VTLIYjOOTFZPUcA=
{{< /highlight >}}

The above format is a concatenated form of `<module-path> <module-version> <h1:checksum>`. go also calculates checksum of `go.mod` file. The `checksum` here is `base64` encoded form of `SHA-2` hash of module directory.

## How to calculate go checksums manually?
The way it works is ::

* Calculate the `sha256sum` of each file content in the module directory
* format the above calculated sha2sum followed by two spaces, filename and the new line `(fmt.Fprintf(h, "%x  %s\n", hf.Sum(nil), file)`. It's taken from here
* Now calculate the `sha256sum` for all the files together
* Convert above hex output into binary mode with `xxd` (on *unix systems, if you do programmatically, base64 could be encoded directly from byte array)
* Encode it with `base64`
* Compare the output with checksum in `go.sum` file

For `go.mod` file it's relatively easy to calculate with below bash command ::
```shell
harsha$ sha256sum $(find go.mod -type f | sort) | sha256sum | xxd -r -p | base64
oPkhp1MJrh7nUepCBck5+mAzfO9JrbApNNgaTdGDITg=
```

This matches the checksum defined in `go.sum` file using `golang.org/x/sys` module `v0.12.0` version.

Only thing one needs to be aware is the path of the `go.mod` shouldn't be passed as it can also influence the `sha256sum` calculation in 2nd iteration.

For module directory verification, the process is almost same except the file name as go verifies it bit differently compared to `go.mod`. From here, to calculate the hash of directory followed by prefix of the module path like `mod.Path+"@"+mod.Version` which translates to `golang.org/x/sys@v0.12.0` for `sys` module.

```shell
harsha $ sha256sum $(find /Users/harsha/go/pkg/mod/golang.org/x/sys@v0.12.0 -type f | sort ) | sed 's#/Users/harsha/go/pkg/mod/##' | sha256sum | xxd -r -p | base64       
CM0HF96J0hcLAwsHPJZjfdNzs0gftsLfgKt57wWHJ0o=
```

The `sed` here trims filepath from absolute path `/Users/harsha/go/pkg/mod/golang.org/x/sys@v0.12.0` to `golang.org/x/sys@v0.12.0` (mod path)

I could verify the same with `golang` too by targeting `HashDir` function [here](https://cs.opensource.google/go/x/mod/+/refs/tags/v0.12.0:sumdb/dirhash/hash.go;l=70-79)
```shell
func TestHashDir2(t *testing.T) {
  out, err := HashDir("/Users/hmusanalli/go/pkg/mod/golang.org/x/sys@v0.12.0", "golang.org/x/sys@v0.12.0", Hash1)
  if err != nil {
    t.Fatalf("HashDir: %v", err)
  }

  want := "h1:CM0HF96J0hcLAwsHPJZjfdNzs0gftsLfgKt57wWHJ0o="
  if out != want {
    t.Errorf("HashDir(...) = %s, want %s", out, want)
  }
}
```

## Conclusion
Calculating checksums manually provides a deeper understanding of data integrity and can be a useful skill in various scenarios. Whether you're verifying file downloads, implementing data validation in programming, or exploring the foundations of cryptographic hashing, understanding the manual checksum calculation process is a valuable addition to your skill set.