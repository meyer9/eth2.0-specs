# [WIP] SimpleSerialize (SSZ) Spec

This is the **work in progress** document to describe `simpleserialize`, the
current selected serialization method for Ethereum 2.0 using the Beacon Chain.

This document specifies the general information for serializing and
deserializing objects and data types.

## ToC

* [About](#about)
* [Terminology](#terminology)
* [Constants](#constants)
* [Overview](#overview)
   + [Serialize/Encode](#serializeencode)
      - [uint: 8/16/24/32/64/256](#uint-816243264256)
      - [Address](#address)
      - [Hash](#hash)
         * [Hash32](#hash32)
         * [Hash96](#hash96)
         * [Hash97](#hash97)
      - [Bytes](#bytes)
      - [List/Vectors](#listvectors)
      - [Container (TODO)](#container)
   + [Deserialize/Decode](#deserializedecode)
      - [uint: 8/16/24/32/64/256](#uint-816243264256-1)
      - [Address](#address-1)
      - [Hash](#hash-1)
         * [Hash32](#hash32-1)
         * [Hash96](#hash96-1)
         * [Hash97](#hash97-1)
      - [Bytes](#bytes-1)
      - [List/Vectors](#listvectors-1)
      - [Container (TODO)](#container-1)
* [Implementations](#implementations)

## About

`SimpleSerialize` was first proposed by Vitalik Buterin as the serialization
protocol for use in the Ethereum 2.0 Beacon Chain.

The core feature of `ssz` is the simplicity of the serialization with low
overhead.

## Terminology

| Term         | Definition                                                                                     |
|:-------------|:-----------------------------------------------------------------------------------------------|
| `big`        | Big Endian                                                                                     |
| `byte_order` | Specifies [endianness:](https://en.wikipedia.org/wiki/Endianness) Big Endian or Little Endian. |
| `len`        | Length/Number of Bytes.                                                                        |
| `to_bytes`   | Convert to bytes. Should take parameters ``size`` and ``byte_order``.                          |
| `from_bytes` | Convert from bytes to object. Should take ``bytes`` and ``byte_order``.                        |
| `value`      | The value to serialize.                                                                        |
| `rawbytes`   | Raw serialized bytes.                                                                          |

## Constants

| Constant       | Value | Definition                                                                            |
|:---------------|:-----:|:--------------------------------------------------------------------------------------|
| `LENGTH_BYTES` |   4   | Number of bytes used for the length added before a variable-length serialized object. |


## Overview

### Serialize/Encode

#### uint: 8/16/24/32/64/256

Convert directly to bytes the size of the int. (e.g. ``uint16 = 2 bytes``)

All integers are serialized as **big endian**.

| Check to perform       | Code                  |
|:-----------------------|:----------------------|
| Size is a byte integer | ``int_size % 8 == 0`` |

```python
assert(int_size % 8 == 0)
buffer_size = int_size / 8
return value.to_bytes(buffer_size, 'big')
```

#### bool

Convert directly to a single 0x00 or 0x01 byte.

| Check to perform  | Code                       |
|:------------------|:---------------------------|
| Value is boolean  | ``value in (True, False)`` |

```python
assert(value in (True, False))
return b'\x01' if value is True else b'\x00'
```


#### Address

The address should already come as a hash/byte format. Ensure that length is
**20**.

| Check to perform       | Code                 |
|:-----------------------|:---------------------|
| Length is correct (20) | ``len(value) == 20`` |

```python
assert( len(value) == 20 )
return value
```

#### Hash

| Hash Type | Usage                                           |
|:---------:|:------------------------------------------------|
|  `hash32` | Hash size of ``keccak`` or `blake2b[0.. < 32]`. |
|  `hash96` | BLS Public Key Size.                            |
|  `hash97` | BLS Public Key Size with recovery bit.          |


| Checks to perform                   | Code                 |
|:-----------------------------------|:---------------------|
| Length is correct (32) if `hash32` | ``len(value) == 32`` |
| Length is correct (96) if `hash96` | ``len(value) == 96`` |
| Length is correct (97) if `hash97` | ``len(value) == 97`` |


**Example all together**

```python
if (type(value) == 'hash32'):
   assert(len(value) == 32)
elif (type(value) == 'hash96'):
   assert(len(value) == 96)
elif (type(value) == 'hash97'):
   assert(len(value) == 97)
else:
   raise TypeError('Invalid hash type supplied')

return value
```

##### Hash32

Ensure 32 byte length and return the bytes.

```python
assert(len(value) == 32)
return value
```

##### Hash96

Ensure 96 byte length and return the bytes.

```python
assert(len(value) == 96)
return value
```

##### Hash97

Ensure 97 byte length and return the bytes.

```python
assert(len(value) == 97)
return value
```

#### Bytes

For general `byte` type:
1. Get the length/number of bytes; Encode into a `4-byte` integer.
2. Append the value to the length and return: ``[ length_bytes ] + [
   value_bytes ]``

| Check to perform                     | Code                   |
|:-------------------------------------|:-----------------------|
| Length of bytes can fit into 4 bytes | ``len(value) < 2**32`` |

```python
assert(len(value) < 2**32)
byte_length = (len(value)).to_bytes(LENGTH_BYTES, 'big')
return byte_length + value
```

#### List/Vectors

| Check to perform                            | Code                        |
|:--------------------------------------------|:----------------------------|
| Length of serialized list fits into 4 bytes | ``len(serialized) < 2**32`` |


1. Get the number of raw bytes to serialize: it is ``len(list) * sizeof(element)``.
   * Encode that as a `4-byte` **big endian** `uint32`.
2. Append the elements in a packed manner.

* *Note on efficiency*: consider using a container that does not need to iterate over all elements to get its length. For example Python lists, C++ vectors or Rust Vec.

**Example in Python**

```python

serialized_list_string = b''

for item in value:
   serialized_list_string += serialize(item)

assert(len(serialized_list_string) < 2**32)

serialized_len = (len(serialized_list_string).to_bytes(LENGTH_BYTES, 'big'))

return serialized_len + serialized_list_string
```

#### Container

```
########################################
                 TODO
########################################
```


### Deserialize/Decode

The decoding requires knowledge of the type of the item to be decoded. When
performing decoding on an entire serialized string, it also requires knowledge
of the order in which the objects have been serialized.

Note: Each return will provide ``deserialized_object, new_index`` keeping track
of the new index.

At each step, the following checks should be made:

| Check to perform         | Check                                                      |
|:-------------------------|:-----------------------------------------------------------|
| Ensure sufficient length | ``length(rawbytes) >= current_index + deserialize_length`` |

#### uint: 8/16/24/32/64/256

Convert directly from bytes into integer utilising the number of bytes the same
size as the integer length. (e.g. ``uint16 == 2 bytes``)

All integers are interpreted as **big endian**.

```python
assert(len(rawbytes) >= current_index + int_size)
byte_length = int_size / 8
new_index = current_index + int_size
return int.from_bytes(rawbytes[current_index:current_index+int_size], 'big'), new_index
```

#### Bool

Return True if 0x01, False if 0x00.

```python
assert rawbytes in (b'\x00', b'\x01')
return True if rawbytes == b'\x01' else False
```

#### Address

Return the 20 bytes.

```python
assert(len(rawbytes) >= current_index + 20)
new_index = current_index + 20
return rawbytes[current_index:current_index+20], new_index
```

#### Hash

##### Hash32

Return the 32 bytes.

```python
assert(len(rawbytes) >= current_index + 32)
new_index = current_index + 32
return rawbytes[current_index:current_index+32], new_index
```

##### Hash96

Return the 96 bytes.

```python
assert(len(rawbytes) >= current_index + 96)
new_index = current_index + 96
return rawbytes[current_index:current_index+96], new_index
```

##### Hash97

Return the 97 bytes.

```python
assert(len(rawbytes) >= current_index + 97)
new_index = current_index + 97
return rawbytes[current_index:current_index+97], new_index
```


#### Bytes

Get the length of the bytes, return the bytes.

| Check to perform                                  | code                                             |
|:--------------------------------------------------|:-------------------------------------------------|
| rawbytes has enough left for length               | ``len(rawbytes) > current_index + LENGTH_BYTES`` |
| bytes to return not greater than serialized bytes | ``len(rawbytes) > bytes_end ``                   |

```python
assert(len(rawbytes) > current_index + LENGTH_BYTES)
bytes_length = int.from_bytes(rawbytes[current_index:current_index + LENGTH_BYTES], 'big')

bytes_start = current_index + LENGTH_BYTES
bytes_end = bytes_start + bytes_length
new_index = bytes_end

assert(len(rawbytes) >= bytes_end)

return rawbytes[bytes_start:bytes_end], new_index
```

#### List/Vectors

Deserialize each object in the list.
1. Get the length of the serialized list.
2. Loop through deserializing each item in the list until you reach the
entire length of the list.


| Check to perform                          | code                                                            |
|:------------------------------------------|:----------------------------------------------------------------|
| rawbytes has enough left for length       | ``len(rawbytes) > current_index + LENGTH_BYTES``                |
| list is not greater than serialized bytes | ``len(rawbytes) > current_index + LENGTH_BYTES + total_length`` |

```python
assert(len(rawbytes) > current_index + LENGTH_BYTES)
total_length = int.from_bytes(rawbytes[current_index:current_index + LENGTH_BYTES], 'big')
new_index = current_index + LENGTH_BYTES + total_length
assert(len(rawbytes) >= new_index)
item_index = current_index + LENGTH_BYTES
deserialized_list = []

while item_index < new_index:
   object, item_index = deserialize(rawbytes, item_index, item_type)
   deserialized_list.append(object)

return deserialized_list, new_index
```

#### Container

```
########################################
                 TODO
########################################
```

## Implementations

| Language | Implementation                                                                                                                                                     | Description                                              |
|:--------:|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
|  Python  | [ https://github.com/ethereum/beacon_chain/blob/master/ssz/ssz.py ](https://github.com/ethereum/beacon_chain/blob/master/ssz/ssz.py)                               | Beacon chain reference implementation written in Python. |
|   Rust   | [ https://github.com/sigp/lighthouse/tree/master/beacon_chain/utils/ssz ](https://github.com/sigp/lighthouse/tree/master/beacon_chain/utils/ssz)                                                         | Lighthouse (Rust Ethereum 2.0 Node) maintained SSZ.      |
|    Nim   | [ https://github.com/status-im/nim-beacon-chain/blob/master/beacon_chain/ssz.nim ](https://github.com/status-im/nim-beacon-chain/blob/master/beacon_chain/ssz.nim) | Nim Implementation maintained SSZ.                       |
|   Rust   | [ https://github.com/paritytech/shasper/tree/master/util/ssz ](https://github.com/paritytech/shasper/tree/master/util/ssz)                                         | Shasper implementation of SSZ maintained by ParityTech.  |
|   Javascript   | [ https://github.com/ChainSafeSystems/ssz-js/blob/master/src/index.js ](https://github.com/ChainSafeSystems/ssz-js/blob/master/src/index.js)                                         | Javascript Implementation maintained SSZ |
