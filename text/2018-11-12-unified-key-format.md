# Summary

This RFC proposes a new unified and human readable format for keys. All TiDB software stack
components (e.g. TiDB, TiKV, pd-ctl, tikv-ctl, ...) *must* follow this new format when printing out
keys and accepting keys from user input like command line parameters.

# Motivation

Currently we have multiple human readable formats for keys:

- Hex format, like `74000000000000001C5F7200000000000000FA`

- Protobuffer format, like `t\000\000\000\000\000\000\000\034_r\000\000\000\000\000\000\000\372`

- Go format, like `t\x00\x00\x00\x00\x00\x00\x00\x1c_r\x00\x00\x00\x00\x00\x00\x00\xfa`

Defects:

- Bad interoperability: These different formats are pain points when working with different tools
  together. For example, pd-ctl produces keys in Go format while tikv-ctl accepts keys in
  Protobuffer format.

- Hard to implement: Actually there are some attempts to support different kind of formats, like
  https://github.com/tikv/tikv/pull/3182 and https://github.com/pingcap/pd/pull/474. However it
  turns out that these implementations won't work in real world scenario. For example, it is
  not sufficient to decode the Golang format by only dealing with hex escapes `\x..` [\[1\]][1]
  and decode the Protobuffer format by only dealing with oct escapes `\...` [\[2\]][2].

- Hard to recognize different parts manually: For hex format, it is hard to extract different parts
  from it manually.

- Hard to be used in shells: For Protobuffer format and Go format, because there are escape
  characters U+005C (\\) in the output, we need to do an escape (convert `\` to `\\`) when passing
  it as a command line parameter in shells. As a result, current documentations are suggesting
  wrong usages [\[3\]][3] that *happens* to work under specially
  crafted code, or won't work [\[4\]][4] actually.

- Contains ambiguous multi-bytes: For Go format, it directly outputs Unicode character if there is
  a valid unicode sequence. For example, hex `00E6B58B01FF` will be printed as `\x00测\x01\xff` in
  Go format. This may result in ambiguous bytes represents in different environments when such
  characters are printed.

To overcome above issues, we need a new human readable format and use it among all TiDB software
stack components, which provides these feature:

- Easy to recognize different parts after encoding by human.

- Easy to implement encoding and decoding. Better to have a lot of existing libraries.

- No escape character U+005C (\\) after encoding so that it can be easily used in shells.

- No multi-byte characters after encoding to avoid encoding issues.

# Detailed Design

This RFC proposes to use hex format like `74000000000000001C5F7200000000000000FA` as unified a human
readable format for keys:

- Each byte must be encoded to a 2 digit hex in upper case.

- Encoded hex string must be concatenated together without any other characters.

- Decoder must support hex string in either lower-case or upper-case.

Although hex format has some defects as discussed in the motivation section above, it is still
preferred over other formats (see alternatives section) with great advantages in simplicity and
uniformity. In addition, it is very easy to be implemented in all languages and does not introduce
escape character U+005C (\\).

In hex format, we are not easy to recognize special parts like `r`, `_t` and `_i`. This can be
solved by other utility functions that further operates on this key format, or just memorizing
the hex representation of these magic bytes.

# Drawbacks

- We need to memorize hex representation of magic bytes, or implement additional helper tools.

- It is not compatible with current formats and we need to write format converters.

# Alternatives

Other binary-to-text encodings like Base64, Percent-Encoding and punycode are investigated. For
Base64 and punycode, it is harder to read magic bytes manually. For Percent-Encoding, its
implementations are varied and only very few of them stick to the Percent-Encoding RFC.

# Unresolved questions

None.

[1]: https://golang.org/ref/spec#Rune_literals
[2]: https://github.com/pingcap/pd/pull/1298/files#diff-ff78a54cb96e131d51e4628c92f70184R246
[3]: https://github.com/pingcap/docs/blob/e81f3225803d37ed4b23f3257dfa48fda38a22f4/tools/tikv-control.md#view-mvcc-of-a-given-key
[4]: https://github.com/pingcap/docs/blob/578c4cbb88e17ad55d0b6a99a1158710425f72fb/tools/pd-control.md#region-key---formatrawpbprotoprotobuf-key
