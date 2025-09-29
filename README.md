
> [!CAUTION]
> # `flow-record` is leaving github
> 
> ## Why?
>
> We believe that all men (and women, and all human between and above) are created equal. In this mindset, it does not make sense to judge people based on their birthplace, or their language, color, religion, or whatsoever.
> 
> We believe that who you are is made up of what you do. If you are caring towards other people, then that's you are. If you do harm to other people, then that's who you are.
> 
> We're concerned of what is currently happening in the United States. We don't like it when a government thinks it is above the law. We don't like it when a government doesn't serve the people, but sees people as a threat. But that's politics.
> 
> Github is part of Microsoft, and Microsoft is supporting this government. For example, Microsoft blocked the mail accounts of ICC members because of political reasons. We don't want to get our accounts blocked or deleted arbitrarily. Therefore, we're going to not support Microsoft in any way. That's why we'll move all our repositories away from github.
> 
> We had a good time. Cheers.
>
> ## Where?
> 
> The new place-to-be for the `flow-record` is <https://codeberg.org/janstarke/flow-record>.
>
> 
<img width="100%" src="https://raw.githubusercontent.com/janstarke/flow-record/main/docs/img/flow-record-header.png"></img> 

[![Crates.io](https://img.shields.io/crates/v/flow-record)](https://crates.io/crates/flow-record)
![Crates.io](https://img.shields.io/crates/l/flow-record)
![Crates.io (latest)](https://img.shields.io/crates/dv/flow-record)

# flow-record
Library for the creation of DFIR timelines, to be used by [`rdump`](https://docs.dissect.tools/en/latest/tools/rdump.html)

# Record flow format

Basically, the [record format](https://github.com/fox-it/flow.record) uses [MsgPack](https://github.com/msgpack/msgpack). A record stream is a sequence of tuples, each containing of a 4 byte size field and a msgpack encoded [record pack](#record-packs) (see below).

The very first of those tuples is a special case; it is some kind of a header.

```text
                                ┌────────[record size] bytes───────┐ 
                               /                                    \
┌──────────────────────────────┬────────────────────────────────────┐
│record size (32bit big endian)│     msgpack encoded content        │
└──────────────────────────────┴────────────────────────────────────┘
```

# Header

The header is formed by the serialized version of the string `RECORDSTREAM\n`, encoded as `bin8`:

```text
   ┌───────────────msgpack type: bin8
   │                                 
   │    ┌──────────length: 13 bytes  
   │    │                            
   │    │    ┌─────13 bytes of data  
   ▼    ▼    ▼                       
┌────┬────┬──────────────┐           
│0xc4│0x0d│RECORDSTREAM\n│           
└────┴────┴──────────────┘           
```

In the following description I omit the fact that every distinct record and every descriptor must be preceded by its length.

# Record packs

All data in the record format are specified as a *record pack*, which is simply a tuple (a *fixarray* of length 2) consisting of an record pack type and the record pack data.

```text
   ┌──────────────────────────── msgpack type ext8/ext16/ext32
   │    ┌─────────────────────── length of content            
   │    │    ┌────────────────── type id must be 0x0e         
   │    │    │     ┌──────────── array of length 2            
   │    │    │     │    ┌─────── record pack type             
   ▼    ▼    ▼     │    │    ┌── record pack data                      
┌────┬────┬────┬───┼────┼────┼──────────────────────────      
│    │    │    │   ▼    ▼    ▼                                
│    │    │    │┌────┬────┬──────────────────                 
│0xc7│    │0x0e││0x92│    │                                   
│    │    │    │└────┴────┴──────────────────                 
│    │    │    │                                              
└────┴────┴────┴────────────────────────────────────────      
```

The following record pack types are known:

|Object ID|Raw value|Description|
|-|-|-|
|RecordPackTypeRecord | `0x1` | |
|RecordPackTypeDescriptor | `0x2` | a [record descriptor](#descriptor) |
|RecordPackTypeFieldtype | `0x3` | |
|RecordPackTypeDatetime | `0x10` | |
|RecordPackTypeVarint | `0x11` | |
|RecordPackTypeGroupedrecord | `0x12` | |

## Descriptor

Every record must have some certain type, which must be specified using a *record descriptor* first. A record descriptor is a [record pack](#record-packs) of type `RecordPackTypeDescriptor`, which is wrapped as an msgpack `ext8` (depending on its size). The msgpack type id is `0x0e`.

Consider the following type:

```rust
struct test_csv_test1 {
       field11: String,
       field12: String,
       field13: String
}
```

which will the following msgpack encoding:

| raw value | explanation |
|-|-|
| `0xc7` | This is an ext8 record |
| `0x43` | length of the containing data |
| `0x0e` | marker for `rdump` that this contains an [object](#objects)

The record pack itself will be the msgpack encoded equivalent of the following data:

```json
[
	2,
	[
		"test_csv_test1",
		[
			[
				"string",
				"field11"
			],
			[
				"string",
				"field12"
			],
			[
				"string",
				"field13"
			]
		]
	]
]
```

It is important to note that every field is encoded as a tuple where the first entry is the datatype, and the second is the field name. The following datatypes are supported:

| Datatype | Mapped from Rust type | Explanation |
|-|-|-|
|`boolean`| `bool`
|`command`|
|`dynamic`|
|`datetime`| `DateTime<TZ: TimeZone>` | UNIX timestamp, encoding as integer in msgpack|
|`filesize`|`flow-record-common::Filesize`|
|`uint16`| `u8`, `u16`|
|`uint32`| `u32`, `u64`|
|`float`|`f32`, `f64`|
|`string`|`String`|
|`stringlist`|
|`dictlist`|
|`unix_file_mode`| `String` | the `chmod` formatted string will be converted to u16 internally
|`varint`| `i8`,`i16`, `i32`, `i64` |
|`wstring`|
|`net.ipv4.Address`|
|`net.ipv4.Subnet`|
|`net.tcp.Port`|
|`net.udp.Port`|
|`uri`|
|`digest`|
|`bytes`|`Vec<u8>`|
|`record`|
|`net.ipaddress`|
|`net.ipnetwork`|
|`path`|`flow_record_common::types::Path`|

### Identifier

A record descriptor is identified by

- the name of the record type
- a hash, which equals the first 32 bit of a SHA256-Hash of the record type name and the names and types (in that order) of the record fields. For example, the above struct would have the following input for the hash function:
  `test_csv_test1field11stringfield12stringfield13string`, which would result in the hash `12a9d8d90aa34e5068dbf6692b82baf6fff0143eeaa84d7b2a9c92021f7747c2`. Here we take the first 4 bytes `12a9d8d9`, interpret them as byte endian integer `313120985` and use this as hash.

Every remaining record can refer to a record descriptor using the name and hash of it.

## Record data

A record contains a reference to the record descriptor and a list of values, in the order of fields like specified in the descriptor.

```text
        ┌───────────────────────────────────────────────────────────── record pack type 1           
        │                     ┌─────────────────────────────────────── name of the descriptor       
        │                     │                   ┌─────────────────── hash of the descriptor       
        │                     │                   │       ┌─────────── array of values              
        ▼                     │                   │       │   ┌─────── value of the first data field
┌────┬────┬───────────────────┼───────────────────┼───────┼───┼───────────────────────────          
│    │    │                   │                   │       │   │                                     
│    │    │┌────┬─────────────┼───────────────────┼───┬───┼───┼───────────────────────────          
│    │    ││    │             ▼                   ▼   │   ▼   ▼                                     
│    │    ││    │┌────┬────┬─────────────┬────┬──────┐│┌────┬─────────┬─────────┬─────────          
│0x92│0x01││0x92││0x92│0xa?│<struct name>│0xce│<hash>│││0x9?│<field 1>│<field 2>│...                
│    │    ││    │└────┴────┴─────────────┴────┴──────┘│└────┴─────────┴─────────┴─────────          
│    │    ││    │                                     │                                             
│    │    │└────┴─────────────────────────────────────┴───────────────────────────────────          
│    │    │                                                                                         
└────┴────┴───────────────────────────────────────────────────────────────────────────────          
```
License: GPL-3.0
