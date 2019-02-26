# TiKV Rust Client Command Line Tool

## Summary

Use the TiKV Rust Client to build a simple command line tool for inspection and anaylsis.

This tool will stand as an analogue to `mycli` or `mysql-client` for users.

## Motivation

Currently `tikv-ctl` offers some ways to interact with a TiKV cluster, it
focuses mostly on the *management* of a TiKV cluster instead of a *client* for
the cluster.

With the `tikv-client` crate users will find a way to interact with their TiKV
cluster from their application. Some users will find it desirable to be able to
inspect their cluster from a command line tool, or otherwise outside of their
application.

Building a simple command line tool to execute commands such as `get`, `set`,
`delete`, and `scan` would mean users do not need to worry about creating such
tools for themselves.

It also provides us with a useful tool during the development cycle, as it will
be very easy to issue one-off commands to TiKV.

Further, we mentioned in the Rust Client RFC
(https://github.com/tikv/rfcs/pull/7) we mentioned we wanted to accomplish this.

## Detailed design

The client will be created using the `tikv-client` crate and its already existing
functionality. It will utilize `clap` to provide a simple wrapper around the
crate API.

### Install

We will build this tool in the `tikv-cli` repository. It will be published.
Users will be able to install it with `cargo install tikv-cli`. (See
*Alternatives* for another option.)

### Configuration

The tool will retrieve its configuration from one of the following places, in
order of precedence (greatest to least):

* The command line options, such as `--pd 127.0.0.1:2379`.
* A local configuration file `./.tikv-cli.toml`
* The global configuration file `${XDG_CONFIG_HOME}/tikv-cli.toml`.
* The default options (connect to localhost, no TLS).

The options are:

```toml
[connection]
# A list of `pd` endpoints.
pd_endpoints = [
    "127.0.0.1:2379",
    "127.0.0.2:2379",
]

[tls]
# The path of the CA file for TLS.
ca_path = "./rsa.ca"
# The path of the certificate file for TLS.
cert_path = "./rsa.cert"
# The path of the key file for TLS.
key_path = "./rsa.key"

[output]
# If the ouput JSON should be minified.
minify = false
# Level of logging to use. Available: error. warn, info, debug, trace
logging = "info"
```

### Input/Output

By default input is by positional arguments in `bash`/`sh` string format.

By default the tool will output in prettyprinted JSON. For single entry results,
it will return a string (including `"`), for key-value pairs it will use a JSON
object `{ key: "", value: "" }`, for multiple results it will return a JSON
array. Errors will use an `{ error: "" }` object. Opting out of prettyprint will
be via the `--minify` flag.

JSON was chosen because it is an efficient, pervasive, human-readable, modern
standard that has many tools (eg `jq`) to use for post processing if needed.

All standard (non-logging) output will be on `stdout`.

It will feature configurable logging levels with `slog` which will output to `stderr`.

The tool will exit zero on successful requests, and non-zero on unsuccessful
requests. In the case of REPL mode, it shall always exit zero.

### Interacting with it

Users will interact with the tool through the command line. It will support both
a one-off mode or a REPL (Read-eval-print-loop) style mode (See alternatives).

Example of the one-off raw mode:

```bash
$ tikv-cli raw set "cncf" "linux foundation"
Finished in 0.001s.
$ tikv-cli raw set "tikv" "rust"
Finished in 0.001s.
$ tikv-cli raw get "cncf"
"linux foundation"
Finished in 0.002s.
$ tikv-cli raw scan "b"..
[
    { key: "cncf", value: "linux foundation" },
    { key: "tikv", value: "rust" },
]
Finished in 0.003s.
```

Example of REPL raw mode:

```bash
$ tikv-cli raw
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

REPL transactional mode, notice how the `>>` communicates that the user is in a
transaction. Also notice how the transaction timestamp is outputted in the
request complete messages.

```bash
$ tikv-cli txn
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
> commit
Finished in 0.001s. (txn ts: 1551216224838)
```

One-off transactional mode is a bit different, as we need some way to maintain
timestamp across commands. The user may choose how to do this, the reccomended
way is via ENV variables, but they must do this. (See alternatives)

```bash
$ TS=$(tikv-cli txn begin)
$ tikv-cli txn ${TS} set "cncf" "linux foundation"
Finished in 0.001s. (txn ts: 1551216224838)
$ tikv-cli txn ${TS} set "tikv" "rust"
Finished in 0.001s. (txn ts: 1551216224838)
$ tikv-cli txn ${TS} get "cncf"
"linux foundation"
Finished in 0.002s. (txn ts: 1551216224838)
$ tikv-cli txn ${TS} scan "b"..
[
    { key: "cncf", value: "linux foundation" },
    { key: "tikv", value: "rust" },
]
Finished in 0.003s. (txn ts: 1551216224838)
$ tikv-cli txn ${TS} commit
Finished in 0.004s. (txn ts: 1551216224838)
```

## Drawbacks

We may not wish to need to maintain this tool, or we may wish for the community
to maintain it.

## Alternatives

* *(Installation/Repo)* We could use the `tikv-client` repository and provide
  either an opt-out or an opt-in binary from `src/bin/tikv-cli.rs`. If we chose
  an opt-in functionality here users mgiht end up with an awkard installation
  call.
* (*Use `tikv-cli`*), we could use `tikv-cli` for this, but it creates as fuzzy
  distinction between management tools and clients.
* (*No REPL, or only REPL*). We may chose to not have a REPL mode, or we may chose
  to *only* have a REPL mode.
* (*A different timestamp handling for one-off mode*). We may find the way we
  handle timestamps in transactional one-off mode to be unwieldly. Perhaps there
  is a better way.

## Unresolved questions

We must decide if there are other alternatives we want to favor.
