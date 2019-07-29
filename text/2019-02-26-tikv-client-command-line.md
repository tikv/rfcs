# TiKV Rust Client Command Line Tool

## Summary

Use the TiKV Rust Client to build a simple command line tool for inspection and analysis.

This tool will stand as an analogue to `mycli` or `mysql-client` for users.

## Motivation

While `tikv-ctl` offers some ways to interact with a TiKV cluster, it
focuses mostly on the *management* of a TiKV cluster instead of a *client* for
the cluster.

With the `tikv-client` Rust crate, users will find a way to interact with their TiKV
cluster from their application. Some users will find it desirable to be able to
inspect their cluster from a command line tool, or otherwise outside of their
application.

Building a simple command line tool to execute commands such as `get`, `set`,
`delete`, or `scan` means users do not need to worry about creating such
tools for themselves.

It also provides us with a useful tool during the development cycle, as it will
be very easy to issue one-off commands to TiKV.

Plus, this tool is one of the goals we mentioned in the Rust Client RFC
(https://github.com/tikv/rfcs/pull/7).

## Detailed design

The client will be created using the `tikv-client` crate and existing
functionality. `clap` and `serde` will be used to provide a simple wrapper
around the crate API.

### Install

This tool will be built in the `client-rust` repository. It will be published.
Users will be able to install it with
`cargo install tikv-client --features cli`. (See *Alternatives* for another option.)

### Configuration

The tool will retrieve its configuration from one of the following places, in
order of precedence (greatest to least):

* Command line flags, such as `--pd 127.0.0.1:2379`.
* A local configuration file `./.tikv-client.toml`
* A global configuration file `${XDG_CONFIG_HOME}/tikv-client.toml` (or
  comparable cross platform options)
* Default options (connect to localhost, no TLS).

The options are:

```toml
[connection]
# A list of `pd` endpoints.
# Default: "127.0.0.1:2379"
pd_endpoints = [
    "127.0.0.1:2379",
    "127.0.0.2:2379",
]

[tls]
# The path of the CA file for TLS.
# Default: None
ca_path = "./rsa.ca"
# The path of the certificate file for TLS.
# Default: None
cert_path = "./rsa.cert"
# The path of the key file for TLS.
# Default: None
key_path = "./rsa.key"

[client]
# Level of logging to use.
# Available: critical, error, warning, info, debug, trace
# Default: info
logging = "info"
# If the ouput JSON should be minified.
# Default: false
minify = false
# The mode the client should be in.
# All clients connected to a keyspace should use the same mode.
# Available: raw, transaction
# Default: transaction
mode = "transaction"
# If the tool should output on stderr the duration of commands.
# Default: true
output_durations = true
# Control the encoding.
# TiKV has a Unified Key Format which is used in cases where a more human readable format is
# required for non-UTF-8 data.
# Avaiable: utf-8, ukf, protobuffer, hex
key_encoding = "utf-8"
value_encoding = "utf-8"
```

On the command line, these options have the same name, but with `_` replaced
with `-` for idiomatic reasons.

### Input/Output

By default input is by positional arguments in `bash`/`sh` string format.

By default the tool will output in prettyprinted JSON. For single entry results,
it will return a string (including `"`), for key-value pairs it will use a JSON
object `{ key: "", value: "" }`, for multiple results it will return a JSON
array. Errors will use an `{ error: "" }` object. Opting out of prettyprint will
be via the `--minify` flag.

JSON is chosen because it is an efficient, pervasive, human-readable, modern
standard that has many tools (eg `jq`) to use for post processing if needed.

All standard (non-logging) output will be on `stdout`.

It will feature configurable logging levels with `slog` which will output to `stderr`.

The tool will exit with exit code zero on successful requests, and non-zero on unsuccessful
requests. In the case of REPL mode, it shall exit zero unless the connection is
found to be lost.

TiKV is often used in cases where a key or value is not a UTF-8 compatible
string, or may print undesirable control characters to the console. We provide the
`key_encoding` and `value_encoding` configuration options to control this.

For most users, their natural first attempt with the client will be to use UTF-8
strings for both keys and values. In order to prevent confusion, we default to
UTF-8 for both and make the options for `*-encoding` discoverable.

The `key_encoding` and `value_encoding` options support `utf-8`, `ukf` (TiKV's
Unified Key format), `protobuffer`, or `hex`. (There is a potential use of
values to store keys.)

### Interacting with it

Users will interact with the tool through a command line interface. It will
support both one-off mode or REPL (Read-eval-print-loop) style mode
(See [alternatives](#Alternatives)).

TiKV has two APIs, *raw* and *transactional*, both are supported by this tool.
Learn about the differences [here](https://tikv.org/docs/architecture/#apis).

Example of the one-off raw mode:

```bash
$ tikv-client --mode raw set "cncf" "linux foundation"
Finished in 0.001s.
$ tikv-client --mode raw set "tikv" "rust"
Finished in 0.001s.
$ tikv-client --mode raw get "cncf"
"linux foundation"
Finished in 0.002s.
$ tikv-client --mode raw scan "b" ..
[
    { key: "cncf", value: "linux foundation" },
    { key: "tikv", value: "rust" },
]
Finished in 0.003s.
```

Example of REPL raw mode:

```bash
$ tikv-client --mode raw
> set "cncf" "linux foundation"
Finished in 0.001s.
> set "tikv" "rust"
Finished in 0.001s.
> get "cncf"
"linux foundation"
Finished in 0.001s.
> scan "b"..
[
    { key: "cncf", value: "linux foundation" },
    { key: "tikv", value: "rust" },
]
Finished in 0.003s.
```

In the REPL of the *(default)* transactional mode below, the `>>` communicates
that the user is in a transaction. Also notice how the transaction timestamp is
outputted in the request complete messages.

```bash
$ tikv-client
> begin
Finished in 0.000s. (txn ts: 1551216224838)
>>set "cncf" "linux foundation"
Finished in 0.001s. (txn ts: 1551216224838)
>> set "tikv" "rust"
Finished in 0.001s. (txn ts: 1551216224838)
>> get "cncf"
"linux foundation"
Finished in 0.001s. (txn ts: 1551216224838)
>> scan "b"..
[
    { key: "cncf", value: "linux foundation" },
    { key: "tikv", value: "rust" },
]
Finished in 0.003s. (txn ts: 1551216224838)
>> commit
Finished in 0.001s. (txn ts: 1551216224838, commit ts: 1551216224838)
```

One-off transactional mode is a bit different, a series of commands can be
provided as a list. This is similar to an 'Auto-commit mode' where `begin` and
`commit` are inserted implictly at the start and end of the command. If any
commands fail, the entire transaction is rolled back and the program exits with
a non-zero exit code.

In most scripting uses, we expect users will newline separate commands.

The output is a JSON array, with each index having the result of the respective
command. In the case of command such as `set` which have no output, a `null` is
used.

```bash
$ tikv-client \
    set "cncf" "linux foundation" \
    set "tikv" "rust" \
    get "cncf" \
    scan "b"..
[
    null,
    null,
    "linux foundation",
    [
        { key: "cncf", value: "linux foundation" },
        { key: "tikv", value: "rust" },
    ],
]
Finished in 0.003s.
```

## Future Work

We plan to eventually spin off this command line interface into an independent
project as it matures. For the time being it will provide a very convenient way
to use and test the Rust client.

## Drawbacks

This will increase our maintenance burden, so it may be better to leave this up
to independent community efforts.

## Alternatives

* (*Use `tikv-cli`*), we could use `tikv-cli` for this, but it creates as fuzzy
  distinction between management tools and clients.
* (*No REPL, or only REPL*). We may chose to not have a REPL mode, or we may chose
  to *only* have a REPL mode.

## Unresolved questions

We must decide if there are other alternatives we want to favor.
