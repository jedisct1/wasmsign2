[![GitHub CI](https://github.com/wasm-signatures/wasmsign2/workflows/CI/badge.svg)](https://github.com/wasm-signatures/wasmsign2/actions)
[![docs.rs](https://docs.rs/wasmsign2/badge.svg)](https://docs.rs/wasmsign2/)
[![crates.io](https://img.shields.io/crates/v/wasmsign2.svg)](https://crates.io/crates/wasmsign2)

# ![Wasmsign2](https://raw.github.com/wasm-signatures/wasmsign2/master/logo.png)

A tool to add and verify digital signatures to/from WASM binaries.

- [!Wasmsign2](#)
  - [WASM signatures](#wasm-signatures)
  - [Installation](#installation)
  - [Usage](#usage)
  - [Inspecting a module](#inspecting-a-module)
  - [Creating a key pair](#creating-a-key-pair)
  - [Signing a WebAssembly module](#signing-a-webassembly-module)
  - [Verifying a WebAssembly module](#verifying-a-webassembly-module)
  - [Verifying a WebAssembly module against multiple public keys](#verifying-a-webassembly-module-against-multiple-public-keys)
  - [Detaching a signature from a module](#detaching-a-signature-from-a-module)
  - [Embedding a detached signature in a module](#embedding-a-detached-signature-in-a-module)
  - [Partial verification](#partial-verification)

## WASM signatures

Unlike typical desktop and mobile applications, WebAssembly binaries do not embed any kind of digital signatures to verify that they come from a trusted source, and haven't been tampered with.

Wasmsign2 takes an existing WebAssembly module, computes a signature for its content, and stores the signature in a custom section.

The resulting binary remains a standalone, valid WebAssembly module, but signatures can be verified prior to executing it.

Wasmsign2 is a proof of concept implementation of the [WebAssembly modules signatures](https://github.com/wasm-signatures/design) proposal.

The proposal, and this implementation, support domain-specific features such as:

- The ability to have multiple signatures for a single module, with a compact representation
- The ability to sign a module which was already signed with different keys
- The ability to extend an existing module additional custom sections, without breaking existing signatures
- The ability to verify multiple subsets of a module's sections with a single signature
- The ability to turn an embedded signature into a detached one, and the other way round.

## Installation

`wasmsign2` is a Rust crate, that can be used in other applications.

See the [API documentation](https://docs.rs/wasmsign2) for details.

It is also a CLI tool to perform common operations, whose usage is summarized below.

The tool requires the Rust compiler, and can be installed with the following command:

```sh
cargo install wasmsign2
```

## Usage

```text
USAGE:
    wasmsign2 [FLAGS] [SUBCOMMAND]

FLAGS:
    -d               Debug information
    -h, --help       Prints help information
    -V, --version    Prints version information
    -v               Verbose output

SUBCOMMANDS:
    attach
    detach
    help             Prints this message or the help of the given subcommand(s)
    keygen
    show
    sign
    split
    verify
    verify_matrix
```

## Inspecting a module

```text
wasmsign2 show --input-file <input_file>
```

Example:

```sh
wasmsign2 show -i z.wasm
```

The `-v` switch prints additional details about signature data.

## Creating a key pair

```text
wasmsign2 keygen --public-key <public_key_file> --secret-key <secret_key_file>

-K, --public-key <public_key_file>    Public key file
-k, --secret-key <secret_key_file>    Secret key file
```

Example:

```sh
wasmsign2 keygen --public-key key.public --secret-key key.secret
```

## Signing a WebAssembly module

```text
wasmsign2 sign [OPTIONS] --input-file <input_file> --output-file <output_file> --secret-key <secret_key_file>

-i, --input-file <input_file>            Input file
-o, --output-file <output_file>          Output file
-K, --public-key <public_key_file>       Public key file
-k, --secret-key <secret_key_file>       Secret key file
-S, --signature-file <signature_file>    Signature file
```

Example:

```sh
wasmsign2 sign -i z.wasm -o z2.wasm -k secret.key
```

The public key is optional. It is only used to include a key identifier into the signature in order to speed up signature verification when a module includes multiple signatures made with different keys.

By default, signatures are assumed to be embedded in modules. Detached signatures can be provided with the optional `--signature-file` argument.

A module that was already signed can be signed with other keys, and can then be verified by any of the corresponding public keys.

## Verifying a WebAssembly module

```text
wasmsign2 verify [OPTIONS] --input-file <input_file> --public-key <public_key_file>

-i, --input-file <input_file>            Input file
-K, --public-key <public_key_file>       Public key file
-S, --signature-file <signature_file>    Signature file
-s, --split <regex>                      custom section names to be verified
```

Example:

```sh
wasmsign2 verify -i z2.wasm -K public.key
```

The optional `-s/--split` parameter is documented in the "partial verification" section down below.

## Verifying a WebAssembly module against multiple public keys

```text
wasmsign2 verify_matrix --input-file <input_file> --public-keys <public_key_files>...

-i, --input-file <input_file>              Input file
-K, --public-keys <public_key_files>...    Public key files
-s, --split <regex>                        custom section names to be verified
```

The command verifies a module's signatures against multiple keys simultaneously, and reports the set of public keys for which a valid signature was found.

The optional `-s/--split` parameter is documented in the "partial verification" section down below.

Example:

```sh
wasmsign2 verify_matrix -i z2.wasm -K public.key -K public.key2
```

## Detaching a signature from a module

```text
wasmsign2 detach --input-file <input_file> --output-file <output_file> --signature-file <signature_file>

-i, --input-file <input_file>            Input file
-o, --output-file <output_file>          Output file
-S, --signature-file <signature_file>    Signature file
```

The command extracts and removes the signature from a module, and stores it in a distinct file.

Example:

```sh
wasmsign2 detach -i z2.wasm -o z3.wasm -S signature
```

## Embedding a detached signature in a module

```text
wasmsign2 attach --input-file <input_file> --output-file <output_file> --signature-file <signature_file>

-i, --input-file <input_file>            Input file
-o, --output-file <output_file>          Output file
-S, --signature-file <signature_file>    Signature file
```

The command embeds a detached signature into a module.

Example:

```sh
wasmsign2 attach -i z2.wasm -o z3.wasm -S signature
```

## Partial verification

A signature can verify an entire module, but also one or more subsets of it.

This requires "cutting points" to be defined before the signature process. It is impossible to verify a signature beyond cutting point boudaries.

Cutting points can be added to a module with the `split` command:

```text
wasmsign2 split [OPTIONS] --input-file <input_file> --output-file <output_file>

-i, --input-file <input_file>      Input file
-o, --output-file <output_file>    Output file
-s, --split <regex>                custom section names to be signed
```

This adds cutting points so that it is possible to verify only the subset of custom sections whose name matches the regular expression, in addition to standard sections.

This command can be repeated, to add new cutting points to a module that was already prepared for partial verification.

Example:

```sh
wasmsign2 split -i z2.wasm -o z3.wasm -s '^.debug_'
```

The above command makes it possible to verify only the custom sections whose name starts with `.debug_`, even though the entire module was signed.

In order to do partial verification, the `--split` parameter is also available in the verification commands:

```sh
wasmsign2 verify -i z3.wasm -K public.key -s '^.debug_'
```

```sh
wasmsign2 verify_matrix -i z3.wasm -K public.key -K public.key2 -s '.debug_'
```
