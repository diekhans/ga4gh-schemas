# GA4GH Wire Protocol Specification

## Internet protocol

GA4GH clients and servers communicate via streaming HTTP/HTTPS using chunked
transfer encoding (rfc7230).  Data in the stream consists of variable length
message encode in one of the formats defined in this document. The encoding is
determined using HTTP content-type negotiation (rfc7231).

The HTTP 1.1 chunked transfer encoding error report model is used to indicate
failed transfers.  A connection that is closed without receiving a trailer
chunk must be treated as an error.  When a trailer chunk is received, it must
be interrogated to see if it contains a `Status` header that indicates an error.

## Messaging

Messages are used to implement the streaming protocol.  A message is not a
GA4GH data object, it is a transfer facility with a predefined set of message.
One type of the message is used to send GA4GH query and response data.  In
response to a query, as stream of messages is sent as a single HTTP chunk
transfer encoded response.  The protocol allows for a stream of mixed GA4GH
object types.


### Transfer resume
The GA4GH streaming protocol supports resume of a stream by sending periodic
checkpoint message that can be used to restart a transfer at that point. See
the Rational section for discussion of this design decision.  To resume from a
failure, the original query, along with a checkpoint token is sent to the
server.  The transfer will resume at that checkpoint, with the first message
being the checkpoint message.

The frequency of checkpoint messages can be requested by the client using the
`X-GA4GH-CHECKPOINT-FREQUENCY` request header.  This is an advisory request,
The exact frequency of checkpoints is up to the discretion of the server.  The
value is the number of approximate number of bytes of to stream before sending
a checkpoint message.  The value is an integer with an optional
case-insensitive suffix of K for kilobytes, M for megabytes, and G for
gigabytes.  Specifying 0 disables checkpoints.

The client tunes the checkpoint frequency based on network speed and
reliability.  Since it's expected that that checkpoint will on be used for
larger transfers over wide area networks, the default value is disabled (0),
There is no requirement that clients implement checkpoint resume.


### Message types

The following message types are defined:
- `dataType typeKey typeUri`: Defines a key used to identify the types of data
  objects that follow in a stream.  This supports mixed GA4GH data types in
  given stream. A message for a given types is only sent once, before any data
  objects that type are sent.  They maybe sent at any point in the stream, the
  only requirement is that a `dataType` message must precede the first message
  of the type.  If a stream is resumed from a checkpoint, `dataType` objects
  will be resent before objects of a given type are sent.  The`typeKey`
  parameter is an integer that is included in the `dataObject` messages.  The
  value is server-instance dependent and its scope is only the current data
  stream.  If a request is resumed, it should not be assume that the same
  `typeKey` values will be the same as the original stream.  The `typeUrl`
  parameter is a URI that identifies the type.  This allows for extensions to
  the GA4GH protocol.  The `typeUrl` should resolve to the protobuf `*.proto`
  file for the type.

- `dataObject typeKey data` - Used to send a data object.  The `typeKey`
  parameter is used to find the matching `dataType` message.  The `data`
  parameter is the data object.

- `checkpoint checkpointObject` - Specify a checkpoint in the data stream.
  The `checkpointObject` is an opaque, server-dependent object.  It should be
  returned as-is when resuming a stream.

# needs to work below here

### Message Encoding

- JSON text encoding

- Protocal Buffers binary encoding


## Rational
The original GA4GH paging protocol offers a simple client interface that
allows clients to read a complete JSON documents 'off-the-wire' and also
allows for the easy resumption of failed transfers.  However, during the
course of implementing the protocol, we are recognizing its drawbacks:

- Paging is performance limiting due to the need to buffer
  the returned JSON document. It requires a tradeoff between
  client/server memory and the number of requests.  Even if a
  given client can dedicate a lot of memory for a transfer,
  the server must impose limits to prevent DoS attacks and
  manage an unpredictable request load.
- Paging introduces problematic latency issues, as the client must
  get a complete response and parse the document before it can
  issue the request for the next page. Large pages make for
  poor interactive responsiveness, and small pages lead to a
  high protocol overhead.
- Paging makes the implementation of a server complex.  It
  must be able to efficiently resume every query.
- Fixed return structures limit the flexibility of queries.
  New result structures must be defined for every new query.
- Result structures limited to a single returned object type
  influence the data model.  We end up with more complex,
  larger objects to include more information in a single
  response.  Variant is the degenerate example, were it can
  contain thousands of calls.

We would like to propose some options for streaming.  Some ideas:
- Streaming protocol that can user either binary or JSON
  encoding. The encoding is chosen by the client using the HTTP
  Accept header. Servers must support JSON output, and may
  optionally support the binary encoding.
- Build on top of HTTP 1.1 chunked transefer encoding, allowing for
  streaming to the client with reliable error reporting.
  This also supports keep-alive streams, allowing the same
  stream to be reused for subsequent queries.
- To facilitate resumption of failed transfers, 'checkpoints'
  can be included in the stream. This differs from paging in that
  it's only used in error cases, so resuming a transfer from a
  checkpoint does not have to be efficient.  Unlike paging, clients that
  do not need to implement checkpoints can ignore them.
- Include object type information in the stream.  This will simplify client
  side code and will support future non-REST query languages that return
  complex results.

  This
approach is used over the more common paging mechanism due to the increased
complexity in efficiently implementing paging with complex queries.  Resume is
normally used in exceptional situations, such as network errors. The
checkpoint approach allows the server to trade off slower performance for
implementation efficiency.

Adding an object type and checkpoint token to every object would greatly
increase the overhead for small objects.  However these could only be sent as
needed.  Either as a special object in the stream or by sending batches of
objects.


#### todo:
- chuncked and paging can still work, but makes both clientro an serve
- data type as URL
- define mime type
- defining versioning
- add line-oriented JSON to protobuf
- GA4GH versioning (if not part of mime type, then allow extension).
- message design on efficent change of many small messsages
- negotation of checkpoint period
- security and encryption
-  This differs from common used paging protocol as it's goal is to
- MIME types
- define URLs
- request streams

### References
[rfc7231 -Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content](http://tools.ietf.org/html/rfc7231)
[rfc7230 - Hypertext Transfer Protocol (HTTP/1.1): Message Syntax and Routing](http://tools.ietf.org/html/rfc7230)
