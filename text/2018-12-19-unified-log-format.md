# Summary

This RFC stipulates a unified log format called "TiDB Log Format" to be used
across all TiDB software, i.e. TiDB, TiKV, and PD.

# Motivation

A unified log format makes it easy to collect (i.e. Fluentd) logs into a central
storage (i.e. ElasticSearch) and allows users to query logs in a structural way
later.

# Detailed Design

A TiDB Log Format file contains a sequence of *lines* containing Unicode
characters in UTF-8 encoding terminated by either the sequence LF or CRLF. Each
line contains a *Log Header Section*, a *Log Fields Section* and a
*Log Message Section*, concatenated by one whitespace character U+0020 (SPACE).

## Log Header Section

The Log Header Section is in the following format:

```
[date_time] [LEVEL] [source_file:line_number]
```

- `date_time`: The human readable time that the log line is generated in the
  local time zone, in the format `YYYY/MM/dd HH:mm:ss.SSS ZZ`.

   Sample: `2018/12/15 14:20:11.015 +08:00`

- `LEVEL`: The log level in upper case. Available levels are `FATAL`, `ERROR`,
  `WARN`, `INFO` and `DEBUG`.

   Sample: `WARN`

- `source_file`: The source file name that generates the log line. Only
  characters matching the regular express `[a-zA-Z0-9\.-_]` are permitted and
  other characters should be removed.

   Sample: `endpoint.rs`

- `line_number`: The source file line number that generates the log line.
  Only digits are permitted.

   Sample: `151`

For cases that source file and source line number are unknown,
`source_file:line_numer` should be `<unknown>` instead.

Log Header Section sample:

```
[2018/12/15 14:20:11.015 +08:00] [INFO] [kv.rs:145]
[2013/01/05 00:01:15.000 -07:00] [ERROR] [<unknown>]
```

## Log Fields Section

The Log Fields Section contains zero or more *Log Field(s)*, with each one
concatenated by one whitespace character U+0020 (SPACE). The number and the
content of Log Fields are left to the application. Additionally, Log Field
content and orders are not required to be identical for different log lines.

Each Log Field is in the following format:

```
[field_key=field_value]
```

Log Field key and value must be strings. If types other than string is provided,
it must be converted to string.

When one of the following Unicode characters exists in the field key or field
value, field key or field value should be JSON string encoded:

- U+0000 (NULL) ~ U+0020 (SPACE)
- U+0022 (QUOTATION MARK)
- U+003D (EQUALS SIGN)
- U+005B (LEFT SQUARE BRACKET)
- U+005D (RIGHT SQUARE BRACKET)

Log Field sample:

```
[region_id=1]
["user name"=foo]
[sql="SELECT * FROM TABLE\nWHERE ID=\"abc\""]
[duration=1.345s]
[client=192.168.0.123:12345]
[txn_id=123000102231]
```

Log Fields Section sample:

```
[region_id=1] [peer_id=14] [duration=1.345s] [sql="insert into t values (\"]This should not break log parsing!\")"]
```

## Log Message Section

The Log Message Section contains an additional log message, which is usually
invariable at the output place. The following Unicode characters are replaced
by U+0020 (SPACE) from the log message to prevent line break:

- U+000A (LINE FEED (LF))
- U+000D (CARRIAGE RETURN (CR))

In addition, the log message must not start with U+005B (LEFT SQUARE BRACKET)
in order to be differentiated from the Log Fields Section. This specification
suggests to prepend a tab character U+0009 (CHARACTER TABULATION) when there is
a leading U+005B character.

## Samples

### Log line without Log Fields

```
[2018/12/15 14:20:11.015 +08:00] [INFO] [tikv-server.rs:13] TiKV Started
```

### Log line with unknown source and simple Log Fields

```
[2013/01/05 00:01:15.000 -07:00] [WARN] [<unknown>] [ddl_job_id=1] [duration=1.3s] DDL Job finished
```

### Log line with JSON encoded Log Fields

```
[2018/12/15 14:20:11.015 +08:00] [WARN] [session.go:1234] [sql="SELECT * FROM TABLE\nWHERE ID=\"abc\""] [duration=1.345s] [client=192.168.0.123:12345] [txn_id=123000102231] Slow query
```

### Log line with JSON encoded Log Fields

```
[2018/12/15 14:20:11.015 +08:00] [FATAL] [panic_hook.rs:45] [stack="   0: std::sys::imp::backtrace::tracing::imp::unwind_backtrace\n             at /checkout/src/libstd/sys/unix/backtrace/tracing/gcc_s.rs:49\n   1: std::sys_common::backtrace::_print\n             at /checkout/src/libstd/sys_common/backtrace.rs:71\n   2: std::panicking::default_hook::{{closure}}\n             at /checkout/src/libstd/sys_common/backtrace.rs:60\n             at /checkout/src/libstd/panicking.rs:381"] [message="thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99"] TiKV panic
```

# Drawbacks

- We need to modify existing log outputs as well as support specifying
  structural fields.
- The decoding process can be complex because we need to find out the end of
  JSON string, which requires correctly parsing JSON string itself.

# Alternatives

We can use fixed fields as an alternative. It can be parsed easily compared to
this RFC, but is not flexible enough.

# Unresolved questions

The decoding process is not provided in this RFC.
