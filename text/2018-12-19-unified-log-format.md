# Unified Log Format

## Summary

This RFC stipulates a unified log format called "TiDB Log Format" to be used
across all TiDB software, i.e. TiDB, TiKV, and PD.

## Motivation

A unified log format makes it easy to collect (i.e. Fluentd) logs into a central
storage (i.e. ElasticSearch) and allows users to query logs in a structural way
later.

## Detailed Design

A TiDB Log Format file contains a sequence of *lines* containing UTF-8
characters terminated by either the sequence LF or CRLF. Each line contains a
*Log Header Section*, a *Log Message Section* and a *Log Fields Section*,
concatenated by one whitespace character U+0020 (SPACE).

### Log Header Section

The Log Header Section is in the following format:

```text
[date_time] [LEVEL] [source_file:line_number]
```

- `date_time`: The human readable time that the log line is generated in the
  local time zone in the following format using the [UTS #35] symbol table:

  ```text
  yyyy/MM/dd HH:mm:ss.SSS ZZZZZ
  ```

   Sample: `2018/12/15 14:20:11.015 +08:00`

- `LEVEL`: The log level in upper case. Available levels are `FATAL`, `ERROR`,
  `WARN`, `INFO` and `DEBUG`.

   Sample: `WARN`

- `source_file`: The source file name that generates the log line. Only
  UTF-8 characters matching the regular expression `[a-zA-Z0-9\.-_]` are
  permitted and other characters should be removed.

   Sample: `endpoint.rs`

- `line_number`: The source file line number that generates the log line.
  Only digits are permitted.

   Sample: `151`

For cases that source file and source line number are unknown,
`source_file:line_numer` should be `<unknown>` instead.

Log Header Section sample:

```text
[2018/12/15 14:20:11.015 +08:00] [INFO] [kv.rs:145]
```

```text
[2013/01/05 00:01:15.000 -07:00] [ERROR] [<unknown>]
```

### Log Message Section

The Log Message Section contains a customized message describing the log line in
the following format:

```text
[message]
```

Message must be valid a UTF-8 string and follows the same encoding rule for
field key and field value (see Log Fields Section).

Log Message Section sample:

```text
[my_custom_message]
```

```text
["Slow Query"]
```

### Log Fields Section

The Log Fields Section contains zero or more *Log Field(s)*, with each one
concatenated by one whitespace character U+0020 (SPACE). The number and the
content of Log Fields are left to the application. Additionally, Log Field
content and orders are not required to be identical for different log lines.

Each Log Field is in the following format:

```text
[field_key=field_value]
```

Log Field key and value must be valid UTF-8 strings.

- If types other than string is provided, it must be converted to string.
- If string in other encoding is provided, it must be converted to UTF-8.

When one of the following UTF-8 characters exists in the field key or field
value, field key or field value should be JSON string encoded:

- U+0000 (NULL) ~ U+0020 (SPACE)
- U+0022 (QUOTATION MARK)
- U+003D (EQUALS SIGN)
- U+005B (LEFT SQUARE BRACKET)
- U+005D (RIGHT SQUARE BRACKET)

Log Field sample:

```text
[region_id=1]
```

```text
["user name"=foo]
```

```text
[sql="SELECT * FROM TABLE\nWHERE ID=\"abc\""]
```

```text
[duration=1.345s]
```

```text
[client=192.168.0.123:12345]
```

Log Fields Section sample:

```text
[region_id=1] [peer_id=14] [duration=1.345s] [sql="insert into t values (\"]This should not break log parsing!\")"]
```

### Samples

#### Sample 1

- There is a space in the message thus message is encoded when printing.
- No log fields.

```text
[2018/12/15 14:20:11.015 +08:00] [INFO] [tikv-server.rs:13] ["TiKV Started"]
```

#### Sample 2

- Unknown source.
- There are some fields but all of them don't need to be encoded.

```text
[2013/01/05 00:01:15.000 -07:00] [WARN] [<unknown>] [DDL_Finished] [ddl_job_id=1] [duration=1.3s]
```

#### Sample 3

- Some fields are encoded but some are not.

```text
[2018/12/15 14:20:11.015 +08:00] [WARN] [session.go:1234] ["Slow query"] [sql="SELECT * FROM TABLE\nWHERE ID=\"abc\""] [duration=1.345s] [client=192.168.0.123:12345] [txn_id=123000102231]
```

```text
[2018/12/15 14:20:11.015 +08:00] [FATAL] [panic_hook.rs:45] ["TiKV panic"] [stack="   0: std::sys::imp::backtrace::tracing::imp::unwind_backtrace\n             at /checkout/src/libstd/sys/unix/backtrace/tracing/gcc_s.rs:49\n   1: std::sys_common::backtrace::_print\n             at /checkout/src/libstd/sys_common/backtrace.rs:71\n   2: std::panicking::default_hook::{{closure}}\n             at /checkout/src/libstd/sys_common/backtrace.rs:60\n             at /checkout/src/libstd/panicking.rs:381"] [error="thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99"]
```

### Note for Non-UTF-8 Characters

Similar to JSON specification, this RFC stipulates that all output characters
must be valid UTF-8 character sequences. Non-UTF-8 characters are not supported.
It's up to the logging framework or framework users to determine what to do when
there are non-UTF-8 characters. This RFC only provides some advice for different
scenarios here.

#### Logging Framework

The logging framework is recommended to accept only UTF-8 characters at the
interface level, e.g. use `str` in Rust language. In this way, the
responsibility of avoiding non-UTF-8 characters is left to the framework user.

For languages that do not provide such facilities, it is recommended that the
logging framework should replace invalid UTF-8 character sequences with
U+FFFD (REPLACEMENT CHARACTER), which looks like this: �. Notice that this is a
lossy operation.

#### Framework Users: Binary Fields

Some fields are just likely to contain invalid UTF-8 characters, for example,
the region start key and region end key. In such scenario, the application is
recommended to do customized encoding before passing the field to the logging
framework. For example, there is already an RFC that required keys to be encoded
in hex format when outputting to logs.

#### Framework Users: Unknown User-Input Fields

Some field content comes from user input, e.g. the SQL expression. The logging
framework user may not be able to ensure that the field content is a valid UTF-8
string. In such scenario, this RFC provides two candidate solutions:

- Lossy: Replacing invalid UTF-8 character sequences with U+FFFD
  (REPLACEMENT CHARACTER)

- Lossless: Performing customized escaping (e.g. Golang quoting) that converts
  invalid UTF-8 character sequences to something else but also allows
  converting back.

### Note for Large Fields

This RFC does not limit the key or value length of fields. However usually long
content (e.g. > 1KB) is not friendly for log storing or log parsing and is
likely to cause issues. It's up to logging framework users to decide whether or
not these long content should be avoided.

## Drawbacks

- We need to modify existing log outputs as well as support specifying
  structural fields.
- The decoding process can be complex because we need to find out the end of
  JSON string, which requires correctly parsing JSON string itself.

## Alternatives

We can use fixed fields as an alternative. It can be parsed easily compared to
this RFC, but is not flexible enough.

## Unresolved questions

The decoding process is not provided in this RFC.

[UTS #35]: http://www.unicode.org/reports/tr35/tr35-31/tr35-dates.html#Date_Field_Symbol_Table
