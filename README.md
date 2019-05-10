<pre>
  PIP: 32
  Title: Technical Specification of BlockChain Voting System
  Type: DATA-Protocol
  Impact: None
  Author: Benjamin Ansbach <i>&lt;benjaminansbach@gmail.com&gt;</i>
  Comments-URI: *reserved for PIP editor*
  Status: WIP
  Created: 2019-05-08
</pre>


## Summary

This PIP describes a voting protocol purely based on the PascalCoin Blockchain.

## Motivation  

XXX

## Specification

While this document describes a possible specification for a voting protocol in the PascalCoin Blockchain, it is not the specification itself. There are no specifications for concrete protocols using a payload in PascalCoin yet. But once accepted, it will be the first and constructed out of this PIP.

To create a poll or cast a vote, only PascalCoin itself will be used, no outside service should be needed. The voting process can be done through one of the wallets.

The evaluation of the votes is only provided to some extend, as different methods to count votes can apply. 

To create this kind of voting specification, a number of different message types, embedded in the payload field of an operation, need to be defined.

### DATA Operations

This specification makes use of DATA operations implemented with V4 (https://www.pascalcoin.org/development/pips/pip-0016) to create a poll and let users participate. A DATA operation allows to send operations with a payload without necessarily transferring value like in a typical transaction from a sender to a receiver. This makes casting a vote essentially for free. 

DATA operations define a `dataType` field that is used to specify the nature of it's payload content. While it is not prohibited to send wrong payload contents using an already established `dataType` specification, it helps PascalCoin adopters to organize and filter operations.

This PIP defines a wide range of messages and will block the `dataType` from the value 1000 to 1020, giving it the possibility to extend to up to 20 message specifications.

It should be noted that a PASA can send DATA operations to itself, so in essence only one account is needed to setup and run a poll.

We will not make use of operation-streams as V4 did not implement the GUID field that would allow us to merge multiple operations into one. It's still possible, but not worth the additional specifications. It will be usable in PascalCoin V5 and the specification can then be adjusted.

All contents will be saved in binary format, which is the most space saving option. We could use other established or human readable formats, but they all come with some overhead (JSON, ..).

### Document storage on IPFS

Right now the grounds (in case of PascalCoin foundation: a PIP) of a poll are hosted on github. While this is a viable solution, not everyone has access to it and it needs action of a small group of persons to merge documents into the codebase and make them public. A reference to this document (e.g. a link) in a poll definition would also take up too much space, since a payload only provides 256 bytes of data. 

URL shorteners can theoretically be used, but since a file related to a poll should be immutable and not changed once referenced, another solution is needed.

Uploading the document using IPFS removes this hurdle and makes this protocol usable for other parties that want to use it, not solely related to the PascalCoin foundation. The SHA256 IPFS hash of a document is used as the link to any document.

It is also possible to upload a document using PascalCoin, but this proposal will not cover this feature. Main reason is the missing GUID of DATA operations in V4. Also, there is also no specification on how a general file upload using DATA operations can be standardized at the time of writing, defining such a format here is beyond the scope of this specification.

### Versioning of message types

Each message is prefixed with a 1 byte value determining the version number of the message type. It could be possible to implement a new `dataType` for each breaking change in the protocol, but this would just clutter the `dataType` registry.

### Operation links in payloads

A link to another operation on the same account is constructed from the operation-hash. A PascalCoin operation consists of 4 values: Block number (4 bytes), account number (4 bytes), n_operation of the executing account (4 bytes) and a ripemd160 hash (20 bytes).

We are able to shorten this ophash by 2 values: The block number and the account number. 

The blocknumber can be fixed to 0, which makes it an ophash of a pending operation. The JSON-RPC API will find it even with a "wrong" block number (wrong = 0 = pending).

Since an operation link in this specification will only link to the same account, the account itself is known and can be added dynamically by the implementation.

This reduces the ophash size from 32 bytes to 24 bytes (4 bytes block number + 4 bytes account).

Using links to other operations makes the display of polls in client software simpler and fast. In case of a spam attack to the poll account, it would take too long to find the matching operations and resolve links. With links we can explicitly read the complete structure of a poll.

 It is called a `ROpHash` (reduced ophash) in this specification and is always 24 bytes in size.

*Remark: It might be possible to use operation.block + operation.opblock to uniquely identify an operation, reducing the size to 8 bytes - but this would presume that all operations are definitely mined in the current block + 1 at the time of creation.*

### Dynamic length values

Values with a dynamic length (strings) will be prefixed with the length itself (UInt8, LE). This is the most space saving option. It is called a `DBytes` in this specification and is at minimum 1 byte in size (length = 0). PascalCoin internally uses 2 bytes, but since a single message type fits into 1 operation payload, a single byte to define the length is sufficient.

### Protocol

The protocol consists of 7 different message types embedded in a DATA operations using the payload field, which are explained below. A message type is the the definition of the contents of a message + it's purpose. Each message has an (sometimes adjusted) acronym to make referencing it within the specification easier.

#### Poll initialization message (PIM)

This message initializes a new poll. It defines the data of the poll that needs to be immutable.

| Spec           | Value |
| -------------- | ----- |
| DATA Data Type | 1000  |
| Acronym        | PIM   |

| Field         | Description                                                  | Specification    | Position |
| ------------- | ------------------------------------------------------------ | ---------------- | -------- |
| version       | The version of the message according to the used specification. | UInt8, LE        | 0        |
| id            | A unique ID identifying the poll itself. Since DATA operations in V4 are missing the GUID field based on the specification (PIP-0016), we will have to create our own UNIQUE id. <br />To get a unique value using PascalCoin related to a PASA before creating an operation, the sum of the n_operation value of the executing account and the account number will be used. All further operations related to this poll will need to refer this ID. | Uint64, LE       | 1-9      |
| start         | The start block when the voting starts (inclusive). <br /><u>Example:</u> Vote starts at block 3000, votes can be casted as soon as block 2999 is mined. | UInt32, LE       | 10-13    |
| duration      | The duration of the vote in blocks (inclusive).<br /><u>Example:</u> Vote starts at block 3000, duration is 1000 blocks. Last votes can be casted until Block 4000 is mined. | UInt16, LE       | 14-15    |
| choices       | The number of choices a user can vote for. In case of rendering the question, you could imagine that one means radio buttons, > 2 = checkboxes. This value cannot be zero and must be leq 10. | UInt8, LE        | 16       |
| options       | The number of available options of a vote. This value cannot be zero and must be < `PIM.choices`. | UInt8, LE        | 17       |
| public        | A flag identifying if the votes should be sent and could be read while voting or if the votes are secret. 0 = no, 1 = yes. If the votes are not public, they need to be encrypted using one of the public keys of `PIM.PPUBKLM`.<br />After the vote ends, the related private keys of `PIM.PPUBKLM` will be published using `PPRIVKM`. | UInt8, LE        | 18       |
| pasa          | The PASA where votes should be send to.                      | UInt32, LE       | 52-55    |
| PTM           | The link to the `PTM` operation as described [here..].       | ROphash          | 56-79    |
| PPUBKLM       | The link to the PPUBKLM message.                             | RopHash          | 80-103   |
| spec          | SHA2-256 IPFS hash of the documentation of the poll (read PIP) on IPFS. | String, 32 bytes | 104-135  |
|               | **Repeating until the end of payload.**                      |                  |          |
| setting       | Setting name (e.g. author).                                  | DBytes           | 136-..   |
| setting-value | The settings value (e.g. benjaminansbach@gmail.com)          | DBytes           |          |

### Reduced Ophash List (ROHL)

This is a re-usable list of max 10 `ROpHash` values.

| Spec           | Value |
| -------------- | ----- |
| DATA Data Type | 1001  |
| Acronym        | ROHL  |

| Field   | Description                                             | Specification | Position   |
| ------- | ------------------------------------------------------- | ------------- | ---------- |
| version | The version of the message according to a specification | UInt8, LE     | 0          |
| poll-id | The ID of the poll initialization message.              | Uint64, LE    | 1-8        |
|         | **Repeating until end of payload**                      |               |            |
| option  | The link to the option.                                 | ROpHash       | 9-31, 32-. |

#### Poll title message (PTM)

This message defines the title of the poll. It could have been embedded in the `PIM` but is in a seperate message type to allow more characters to be used. It allows you to add up to 221 Bytes to describe the poll in short, which is sufficient as the `PIM.spec` field, which links to an IPFS document, can be used to find out more.

| Spec           | Value |
| -------------- | ----- |
| DATA Data Type | 1002  |
| Acronym        | PTM   |

| Field   | Description                                                  | Specification | Position |
| ------- | ------------------------------------------------------------ | ------------- | -------- |
| version | The version of the message according to `PIM.version`.       | UInt8, LE     | 0        |
| poll-id | The ID of the poll initialization message.                   | UInt64, LE    | 1-8      |
| polm    | The link to the `POLM` message.                              | ROpHash       | 9-32     |
| text    | The introductory text of the poll that can be used without any parsing or whatever of the spec. | String        | 33-255   |

#### Poll option links message (POLM, virtual)

This message equals the `ROHL` message type and contains a list of operation links where each operation contains a `POM` message.

#### Poll option message (POM)

This message defines a single option of the poll. It allows you to add up to 213 bytes, which is sufficient as the `POM.spec` field can be used to find out more about the option itself.

| Spec           | Value |
| -------------- | ----- |
| DATA Data Type | 1003  |
| Acronym        | POM   |

| Field   | Description                                                  | Specification | Position |
| ------- | ------------------------------------------------------------ | ------------- | -------- |
| version | The version of the message according to `PIM.version`.       | UInt8, LE     | 0        |
| poll-id | The ID of the poll initialization message.                   | UInt64, LE    | 1-8      |
| spec    | SHA2-256 IPFS hash of the documentation of the option (read PIP) on IPFS. Default value are 32 x 0 bytes for a non-existing spec. | 32 bytes      | 9-40     |
| text    | The text of the poll option.                                 | String        | 41-255   |

#### Poll Vote message (PVM)

This message defines a vote of a user for a poll. It needs to be encrypted using the public key of `PIM.pasa` in case `PIM.public = false`.

| Spec           | Value |
| -------------- | ----- |
| DATA Data Type | 1004  |
| Acronym        | PVM   |

| Field   | Description                                                  | Specification | Position |
| ------- | ------------------------------------------------------------ | ------------- | -------- |
| version | The version of the message according to `PIM.version`.       | UInt8, LE     | 0        |
| poll-id | The ID of the poll initialization message.                   | UInt64, LE    | 1-8      |
|         | **Repeating depending on the number of `POM.choices`**       |               |          |
| option  | A 16 bit murmur3 hash of the options full ophash to validate that the vote was indeed for this option and the client chose the correct options. | UInt16, LE    | 9-10     |
| value   | 0 = No, 1 = Yes                                              | UInt8, LE     | 11       |

#### Poll Public Key Link message (PPUBKLM, virtual)

This message equals the `ROHL` message type and holds a list of ophashes that itself contain messages of type `PPUBKM`.

#### Poll Public Key message (PPUBKM)

This message is used to publish a public key.

| Spec           | Value |
| -------------- | ----- |
| DATA Data Type | 1005  |
| Acronym        | PPKLM |

| Field   | Description                                                  | Specification | Position |
| ------- | ------------------------------------------------------------ | ------------- | -------- |
| version | The version of the message according to `PIM.version`.       | UInt8, LE     | 0        |
| poll-id | The ID of the poll initialization message.                   | UInt64, LE    | 1-8      |
| key     | The public key that will be published, Encoded in PascalCoin format. | Bytes         | 9-..     |

#### Poll Private Key message (PPRIVKM)

This message is used to publish a private key.

| Spec           | Value |
| -------------- | ----- |
| DATA Data Type | 1006  |
| Acronym        | PPKM  |

| Field   | Description                                                  | Specification | Position |
| ------- | ------------------------------------------------------------ | ------------- | -------- |
| version | The version of the message according to `PIM.version`.       | UInt8, LE     | 0        |
| poll-id | The ID of the poll initialization message.                   | UInt64, LE    | 1-8      |
| key     | The private key that will be published, Encoded in PascalCoin format. | Bytes         | 9-..     |

### Process to create a poll

Let's describe the process to create a poll. Since we are referencing messages from Down to Top, we need to go the following way to create a poll: `POM [n] -> POLM -> PTM -> PIM`.

1. Create operations of type `POM` and collect the resulting operation hashes.
2. Create the `POLM` operation that will contain the links to the `POM` messages. Save the ophash.
3. Create operations of type `PPUBKM` and collect the resulting operation hashes.
4. Create the `PPUBKLM` operation that will contain the links to the `PPUBKM` messages. Save the ophash.
5. Create the `PTM` operation that links to the `POLM` message. Save the ophash.
6. Create the `PIM` operation that links to the `PTM` and `PPUBKLM` message.

### Process to read / render the poll

A poll can be read fast by a voting client by publishing making the ophash of the `PIM` message public and read the account that executed the poll.

- Read the `PIM` message of the published ophash.
- Read `PIM.ptm`
- Load operation `PIM.ptm` by applying account of `PIM`
- Load operation `PTM.polm` by applying account of `PIM` and collect.
- Load operations from previously collected `POLM` message.
- Read `PIM.PPUBKLM`
- Load operation `PIM.PPUBLM` by applying account of `PIM` and collect.
- Load operations from previously collected `PPUBLM` message.
- Render

### Process to calculate the poll results

- Read the poll

- get all operations of `PIM.pasa` and filter them by the following rules (using AND)
  - `operation.optype = 10` (DATA operations)
  - `operation.dataType = 1004`
  - `operation.block >= PIM.start && operation.block <= PIM.start + PIM.duration`
    - Decode payload if `PIM.public` = false with `PIM.PPUBKLM` public keys.
    - Check if `operation.payload.poll-id` equals `PIM.id`
    - Load all option responses
      - check if `PVM[n].option` equals `murmur3(POM[n].text)`
  - Apply own counting logic like filtering duplicates etc.



## Rationale

XXX

## Backwards Compatibility

XXX

## Reference Implementation

XXX

## Links

XXX
