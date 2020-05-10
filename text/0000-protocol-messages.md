+ Feature name: `protocol-messages`
+ Start date: 2020-04-27
+ RFC PR: [iotaledger/bee-rfcs#00](https://github.com/iotaledger/bee-rfcs/pull/00)
+ Bee issue: [iotaledger/bee#00](https://github.com/iotaledger/bee/issues/00)

# Summary

This RFC introduces the IOTA protocol messages.

# Motivation

Both [IRI](https://github.com/iotaledger/iri) nodes and [HORNET](https://github.com/gohornet/hornet) nodes operate on
the IOTA networks with the TLV protocol presented in this RFC. To be able to take part in the networks, Bee nodes need
to implement the exact same protocol; however, IRI does not implement all the messages and as such Bee nodes and IRI
nodes will not be able to communicate.

# Detailed design

## Trait

The `Message` trait provides serialization/deserialization primitives but does not deal with protocol metadata at all
and is not expected to be used standalone.

```rust
/// A trait describing the behavior of a message.
///
/// This trait is protocol agnostic and only provides serialization and deserialization to and from bytes buffers.
/// It should not be used as is but rather be paired with a higher layer - like a type-length-value encoding - and as
/// such does not provide any safety check on inputs/outputs.
pub(crate) trait Message {
    /// The unique identifier of the message within the protocol.
    const ID: u8;

    /// Returns the size range of the message as it can be compressed.
    fn size_range() -> Range<usize>;

    /// Deserializes a bytes buffer into a message.
    ///
    /// # Arguments
    ///
    /// * `bytes`   -   The bytes buffer to deserialize from.
    ///
    /// # Panics
    ///
    /// Panics if the provided buffer has an invalid size.
    /// The size of the buffer should be within the range returned by the `size_range` method.
    fn from_bytes(bytes: &[u8]) -> Self;

    /// Returns the size of the message.
    fn size(&self) -> usize;

    /// Serializes a message into a bytes buffer.
    ///
    /// # Arguments
    ///
    /// * `bytes`   -   The bytes buffer to serialize into.
    ///
    /// # Panics
    ///
    /// Panics if the provided buffer has an invalid size.
    /// The size of the buffer should be equal to the one returned by the `size` method.
    fn into_bytes(self, bytes: &mut [u8]);
}
```

## Type-length-value protocol

The [type-length-value](https://en.wikipedia.org/wiki/Type-length-value) module is a safe layer on top of the messages.
It allows serialization/deserialization to/from bytes ready to be sent/received to/from a transport layer.

### Header

```rust
/// A header for the type-length-value encoding.
pub(crate) struct Header {
    /// Type of the message.
    pub(crate) message_type: u8,
    /// Length of the message.
    pub(crate) message_length: u16,
}
```

### Methods

```rust
/// Since the following methods have very common names, `from_bytes` and `into_bytes`, the sole purpose of this struct
/// is to give them a proper namespace to avoid confusion.
pub(crate) struct Tlv {}

impl Tlv {
    /// Deserializes a TLV header and a bytes buffer into a message.
    ///
    /// # Arguments
    ///
    /// * `header`  -   The TLV header to deserialize from.
    /// * `bytes`   -   The bytes buffer to deserialize from.
    ///
    /// # Errors
    ///
    /// * The advertised message type does not match the required message type.
    /// * The advertised message length does not match the buffer length.
    /// * The buffer length is not within the allowed size range of the required message type.
    pub(crate) fn from_bytes<M: Message>(header: &Header, bytes: &[u8]) -> Result<M, TlvError> {
        ...
    }

    /// Serializes a TLV header and a message into a bytes buffer.
    ///
    /// # Arguments
    ///
    /// * `message` -   The message to serialize.
    pub(crate) fn into_bytes<M: Message>(message: M) -> Vec<u8> {
        ...
    }
}
```

## Messages

Since the messages are all different one from another, there is no construction method in the `Message` trait. All the
`Message` implementations are expected to have a convenient `new` method to build them from primitive types.

### Endianness

All multi-byte number fields of the messages of the protocol are represented as [big-endian](https://en.wikipedia.org/wiki/Endianness).

### Derived traits

The following traits are expected to be derived by every `Message` implementation:

- `Default` which is very convenient for the implementation of `Message::from_bytes`;
- `Clone` which is necessary to provide ownership in the context of a message broadcast;

### Version 0

#### `Handshake`

Type ID: `1`

A message that allows two nodes to pair.
Contains useful information to verify that the pairing node is operating on the same configuration.
Any difference in configuration will end up in the connection being closed and the nodes not pairing.

|Name|Description|Type|Length|
|----|-----------|----|------|
|`port`|Protocol port of the node<sup>1</sup>.|`u16`|2|
|`timestamp`|Timestamp - in ms - when the message was created by the node.|`u64`|8|
|`coordinator`|Public key of the coordinator being tracked by the node.|``[u8; 49]``|49|
|`minimum_weight_magnitude`|Minimum Weight Magnitude of the node.|`u8`|1|
|`supported_versions`|Protocol versions supported by the node<sup>2</sup>.|`Vec<u8>`|1-32|

<sup>1</sup> When an incoming connection is created, a random port is attributed. This field contains the actual port
being used by the node and is used to match the connection with a potential white-listed peer.

<sup>2</sup> Bit-masks are used to denote what protocol versions the node supports. The LSB acts as a starting point.
Up to 32 bytes are supported, limiting the number of protocol versions to 256. Examples:
* `[0b00000001]` denotes that the node supports protocol version 1.
* `[0b00000111]` denotes that the node supports protocol versions 1, 2 and 3.
* `[0b01101110]` denotes that the node supports protocol versions 2, 3, 4, 6 and 7.
*  `[0b01101110, 0b01010001]` denotes that the node supports protocol versions 2, 3, 4, 6, 7, 9, 13 and 15.
* `[0b01101110, 0b01010001, 0b00010001]` denotes that the node supports protocol versions 2, 3, 4, 6, 7, 9, 13, 15, 17
  and 21.

### Version 1

#### `LegacyGossip`

Type ID: `2`

A legacy message to broadcast a transaction and request another one at the same time.

|Name|Description|Type|Length|
|----|-----------|----|------|
|`transaction`|Transaction to broadcast. Can be compressed<sup>1</sup>.|`Vec<u8>`|292-1604|
|`hash`|Hash of the requested transaction.|`[u8; 49]`|49|

<sup>1</sup> Compression is detailed at the end.

**Note**: This message is the original IRI protocol message before the TLV protocol was introduced. It was kept by
HORNET for compatibility with IRI but is not used between HORNET nodes. Its "ping-pong" concept has complex consequences
on the node design and as such will not be implemented by Bee.

### Version 2

#### `MilestoneRequest`

Type ID: `3`

A message to request a milestone.

|Name|Description|Type|Length|
|----|-----------|----|------|
|`index`|Index of the requested milestone.|`u32`|4|

#### `TransactionBroadcast`

Type ID: `4`

A message to broadcast a transaction.

|Name|Description|Type|Length|
|----|-----------|----|------|
|`transaction`|Transaction to broadcast. Can be compressed<sup>1</sup>.|`Vec<u8>`|292-1604|

<sup>1</sup> Compression is detailed at the end.

#### `TransactionRequest`

Type ID: `5`

A message to request a transaction.

|Name|Description|Type|Length|
|----|-----------|----|------|
|`hash`|Hash of the requested transaction.|`[u8; 49]`|49|

#### `Heartbeat`

Type ID: `6`

A message that informs about the part of the tangle currently being fully stored by a node.
This message is sent when a node:
* just got paired to another node;
* did a local snapshot and pruned away a part of the tangle;
* solidified a new milestone;

It also helps other nodes to know if they can ask it a specific transaction.

|Name|Description|Type|Length|
|----|-----------|----|------|
|`solid_milestone_index`|Index of the last solid milestone.|`u32`|4|
|`snapshot_milestone_index`|Index of the snapshotted milestone.|`u32`|4|

### Compression

A transaction encoded in bytes has a length of `1604`. The `payload` field itself occupies `1312` bytes and is often
partially or completely filled with `0`s. For this reason, trailing `0`s of the `payload` field are removed, providing
a compression rate up to nearly 82%. Only the `payload` field is altered during this compression and the order of the
fields stays the same.

```rust
pub(crate) fn compress_transaction_bytes(bytes: &[u8]) -> Vec<u8> {
    ...
}

pub(crate) fn uncompress_transaction_bytes(bytes: &[u8]) -> [u8; 1604] {
    ...
}
```

# Drawbacks

There is no specific drawback occurring because of the introduction of this protocol.

# Rationale and alternatives

There are alternatives to a type-length-value protocol but it is very efficient and easily updatable without breaking
change. Also, since this is the protocol that has been chosen for the IOTA network, there is no other alternative for
Bee.

# Unresolved questions

There are no open questions at this point.
This protocol has been used for a long time and this RFC will be updated with new message types when/if needed.