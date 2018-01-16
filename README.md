# Mashpack

## Motivation

Mashpack is based on a combination of [MessagePack](https://github.com/msgpack/msgpack)
for packing JSON-like objects into binary and [HTTP/2 HPACK](https://http2.github.io/http2-spec/compression.html)
for encoding [variable-length integers](https://http2.github.io/http2-spec/compression.html#rfc.section.5.1)
in order to achieve superior JSON-object compression.

Variable length integers are less efficient at representing numbers compared
to a packed uint32 if both are 32 bits long but when it comes to representing
integers within 0x00000000-0xFFFFFFFF the variable length integer will only use
the space required to represent the integer and therefore optimize for smaller
integers.

Using this method also allows Mashpack to break away from requiring
multiple sizes for each data type compared to MessagePack. (`ARRAY` in Mashpack
is capable of representing MessagePack's `fixarray`, `array16` and `array32`)

Variable length integers also remove the requirement that all collections cannot
be longer than 2\*\*32-1 which is a constraint inherant to MessagePack's collection types.

This repository also serves as a "reference-implementation" of the specification in Python. The specification and
implementation are currently works in progress, feel free to contribute with a pull request!

## Specification

### Data Types Overview

| data type | prefix      | prefix length |
|-----------|-------------|---------------|
| ARRAY     | `b100`      | 3             |
| INT       | `b101`      | 3             |
| MAP       | `b110`      | 3             |
| STR       | `b111`      | 3             |
| RESERVED  | `b0000`     | 4             |
| MAPSTR    | `b0001`     | 4             |
| ARRAYT    | `b0010`     | 4             |
| MAPT      | `b0011`     | 4             |
| MATRIX    | `b0100`     | 4             |
| NEGINT    | `b0101`     | 4             |
| BIN       | `b0110`     | 4             |
| NULL      | `b01110000` | 8             |
| TIMESTAMP | `b01110001` | 8             |
| FLOAT32   | `b01110010` | 8             |
| FLOAT64   | `b01110011` | 8             |
| FALSE     | `b01110100` | 8             |
| TRUE      | `b01110101` | 8             |
| RESERVED  | `b01110110` | 8             |
| RESERVED  | `b01110111` | 8             |
| EXT       | `b01111XXX` | 8             |

### Variable-Length Integers

A variable length integer is encoded on one or more bytes in the following format:
Start with a "prefix" which the number of bits to offset into the first byte.
A prefix of zero means to use the whole byte. A prefix of 3 means to use the least
significant 5 bits. Of these bits there are N-1 data bits and 1 stop bit.  If the value
is 1 in the most significant bit (after accounting for prefix) then the integer is complete
otherwise there is another byte to consume. All data is shifted by 7 bits and the next byte
is added and processed similarly.

Within this specification a variable-length integer is labeled as `varint(prefix)`.

Here's what the integer 1283 would look like represented with a prefix of 3:

```
+-----+---------+------------+
| xxx | [0]0011 | [1]0000101 |
+-----+---------+------------+

where
- xxx is the prefix bits that aren't a part of the integer.
- [N] are the stop bits. Notice the first stop bit is 0 indicating continue
  and the second is 1 indicating stop.
- 1283 == (b0000101 << 4) + b0011 == b00001010000011
```

### Arrays (`ARRAY`, `ARRAYT`)

```
ARRAY:
+-----+-----------+============================+
| 100 | varint(3) | N mashpack-encoded objects |
+-----+-----------+============================+

ARRAYT:
+------+-----------+=======================+=========================================+
| 0010 | varint(4) | mashpack object w/ type | N-1 mashpack-encoded objects w/o type |
+------+-----------+=======================+=========================================+
```

### Integers (`INT` and `NEGINT`)

```
INT:
+-----+-----------+
| 101 | varint(3) |
+-----+-----------+

NEGINT:
+------+-----------+
| 0101 | varint(4) |
+------+-----------+
```

### Maps (`MAP`, `MAPT`, and `MAPSTR`)

```
MAP:
+-----+-----------+==============================+
| 110 | varint(3) | N*2 mashpack-encoded objects |
+-----+-----------+==============================+

MAPT:
+------+-----------+============================+=====================================+
| 0011 | varint(4) | 2 mashpack objects w/ type | N-2 mshpck-encoded objects w/o type |
+------+-----------+============================+=====================================+

MAPSTR:
+-----++-----------+=====================+============================+
| 0001 | varint(4) | N NUL-separated STR | N mashpack-encoded objects |
+-----++-----------+=====================+============================+
```

### Strings (`STR` and `BIN`)

```
STR:
+-----+-----------+===========================+
| 111 | varint(3) | N bytes of unicode string |
+-----+-----------+===========================+

BIN:
+------+-----------+========================+
| 0110 | varint(4) | N bytes of byte string |
+------+-----------+========================+
```

### Matrix (`MATRIX`)

```
MATRIX:
+------+-----------+---+-----------+==============================+
| 0100 | varint(4) | X | varint(7) | N*M mashpack-encoded objects |
+------+-----------+---+-----------+==============================+
```

### Bools (`TRUE` and `FALSE`)

```
FALSE:
+----------+
| 01110100 |
+----------+

TRUE:
+----------+
| 01110100 |
+----------+
```

### Floats (`FLOAT32` and `FLOAT64`)

```
FLOAT32:
+----------+=================================+
| 01110010 | IEEE 754 single precision float |
+----------+=================================+

FLOAT64:
+----------+=================================+
| 01110011 | IEEE 754 double precision float |
+----------+=================================+
```

### Null (`NULL`)

```
NULL:
+----------+
| 01110000 |
+----------+
```

### Extensions (`EXT`)

```
EXT:
+-------+------------+
| 01111 | varint(5)  |
+-------+------------+
```

## Future Improvements

- `MAPT`, `ARRAYT` for 'typed map' and 'typed array' which the first element
  declares it's type information and then the rest of the elements follow the
  same type information. This allows for getting rid of the type prefix on
  items that are extensible.
  
  This is especially useful for large arrays of integers or floats as it removes
  lots of header information.
  
  Example:
  
  ```
  Encoding [0.0] * 4 comparison between ARRAY and ARRAYT(INT)

  ARRAY
  b100|10100[b01110011[FLOAT],b01110011[FLOAT],b01110011[FLOAT],b01110011[FLOAT]] (37 bytes)
  
  ARRAYT(INT)
  b0010|1100[b01110011[FLOAT],[FLOAT],[FLOAT],[FLOAT]] (34 bytes)
  ```

- `MATRIX` which is where a list of lists can be condensed into a single list
  and only encode the lengths of the two dimensions. This can be done if the system
  detects `ARRAY[ARRAY]` and all inner `ARRAY`s are the same length.
  
  There's also great potential in encoding large arrays as instead of `VARINT(3)` for
  encoding the length of the arrays the length is encoded in `VARINT(0)` which can
  store much larger values. Potential for use in graphics, data science, and machine learning.
  
  Example:
  ```
  Encoding [[1], [1], [1]] comparison between ARRAY[ARRAY] and MATRIX

  ARRAY[ARRAY]
  b100|10011[b100|10001[b101|10001],b100|10001[b101|10001],b100|10001[b101|10001]] (7 bytes)
  
  MATRIX
  b0111|0000,10000011,10000001[b101|10001,b101|10001,b101|10001] (6 bytes)
  ```

- `MAPSTR` which is a map where all keys are `STR` without `0x00` bytes. This is a common
  scenario when encoding JSON and can lead to increased packing of keys. Instead of packing
  key-value, key-value the structure is packed key\x00key\x00value,value. Keys are also packed
  without header or length information as it is known they are `STR` and their length is determined
  with `0x00` bytes.
  
  Example:
  ```
  Encoding {'foo': 1, 'bar': 2, 'foooooooooooobar': 3} comparison between MAP and MAPSTR
  
  MAP:
  b110|10011{b111|10011"foo":b101|10001,b111|10011,"bar":b101|10010,b111|00000b10000001,"foooooooooooobar":b101|10011} (30 bytes)
  
  MAPSTR:
  b0001|1011{"foo"b00000000"bar"b00000000"foooooooooooobar"b00000000,b101|10001,b101|10010,b101|10011} (29 bytes)
  ```
  
- Maybe need to investigate a "splicing" method for `MAPSTR` where all `STR` keys are inter-spliced
  and then the length of the entire splicing is encoded as a singular length? Encoding would become
  more efficient for similar key lengths and many keys. Use `NUL` values when the string is a suitable
  length for that splice.
  
  This would enhance it's efficiency when dealing with `MAP[STR,ANY]` with keys that aren't necessarily long
  as the implementation above only benefits from long keys.
  
  Example:
  ```
  Encoding {'foo': 1, 'bar': 2} comparison between MAP and MAPSTR2
  
  MAP:
  b110|10011{b111|10011"foo":b101|10001,b111|10011,"bar":b101|10010} (11 bytes)
  
  MAPSTR2:
  b0001|1010{b111|10110"fboaor":b101|10001,b101|10010} (10 bytes)
  ```

- Handling of special cases such as `ARRAYT[NONE/TRUE/FALSE]` which would literally map as an `ARRAYT`
  header with length information and the first type and then no other data. This also goes for `MAP`
  types with fixed-width no-data types.
  
- Figure out a good datatype for `b0000` because it's in a valuable location having 4 prefix bytes.

- User-defined `EXT` types. b01110000 to b01110111 are reserved for use by Mashpack.

- Implement a pure-Python version of the specification as well as a Cython implementation.

## Comparison to JSON and MsgPack

TODO

## Implementations

- [Python](https://github.com/SethMichaelLarson/mshpack) (WIP)

## License

Apache-2.0
