# GA4GH Wire Protocol Specification
Version 0.1 [draft]

This document defines the GA4GH Internet protocol.

This document assumes that the GA4GH has switch from Avro to [Protocol Buffers
V3](https://developers.google.com/protocol-buffers/docs/proto3).


## Internet protocol GA4GH clients and servers communicate via streaming
HTTP/HTTPS using chunked transfer encoding ([rfc7230 - Hypertext Transfer Protocol (HTTP/1.1): Message Syntax and Routing](http://tools.ietf.org/html/rfc7230)
).
Data in the stream
consists of variable length message encode in one of the formats defined in
this document. The encoding is determined using HTTP content-type negotiation
([rfc7231 -Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content](http://tools.ietf.org/html/rfc7231)).
A response stream of messages is sent as an HTTP 1.1 chunked
transfer encoding response.  A request stream maybe send using a HTTP 1.0 request,
with content length or chunked transfer encoding.

The HTTP 1.1 chunked transfer encoding error report model is used to indicate
failed transfers.  A connection that is closed without receiving a trailer
chunk must be treated as an error.  When a trailer chunk is received, it must
be interrogated to see if it contains a `Status` header that indicates an
error.

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
failure, a `resume` message, along with a checkpoint object is sent to the
server. 

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
- `dataType typeKey typeName`: Defines a key used to identify the types of
  data objects that follow in a stream.  This supports mixed GA4GH data types
  in given stream. A message for a given types is only sent once, before any
  data objects that type are sent.  They maybe sent at any point in the
  stream, the only requirement is that a `dataType` message must precede the
  first message of the type.  If a stream is resumed from a checkpoint,
  `dataType` objects will be resent before objects of a given type are sent.
  The`typeKey` parameter is an integer that is included in the `dataObject`
  messages.  The value is server-instance dependent and its scope is only the
  current data stream.  If a request is resumed, it should not be assume that
  the `typeKey` values will be the same as the original stream.  The
  `typeName` parameter is a full qualified record name `package.message` that
  identifies the type.  All GA4GH objects will match `org.ga4gh.*`.  Non-GA4GH
  data types may also use this protocol, however they must not be under the
  `org.ga4gh` package.

- `dataObject typeKey data` - Used to send a data object.  The `typeKey`
  parameter is used to find the matching `dataType` message.  The `data`
  parameter is the data object.

- `checkpoint checkpointObject` - Specify a checkpoint in the data stream.
  The `checkpointObject` is an opaque, server-dependent object.  It should be
  returned as-is when resuming a stream.

- `resume checkpointObject` - Resume a transfer at the specified checkpoint in
  the data stream.  The next data object that was or would have been received
  after in the corresponding `checkpoint` message will be the next data object
  received on the resume. It will be preceded by the required `dataType`
  messages.  The same checkpoint may resumed multiple times, as would be
  required if there was a failure before another `checkpoint` messages is
  received in the stream.

# NEEDS WORK BELOW HERE


## Message Encoding
Messages are encoded using
[Protocol Buffers V3](https://developers.google.com/protocol-buffers/docs/proto3).


- JSON text encoding

- Protocal Buffers binary encoding


## Rational


The original GA4GH paging protocol offers a simple client interface that
allows clients to read a complete JSON documents 'off-the-wire'.  The returns
response objects that contained a homogeneous vector of results along with a
next page token.  This allowed for the easy resumption of failed transfers.
However, during the course of implementing the protocol, we are recognizing
its drawbacks:

- Paging in was performance limiting due to the need to buffer the returned
  JSON document. It requires a tradeoff between client/server memory and the
  number of requests.  Even if a given client can dedicate a lot of memory for
  a transfer, the server must impose limits to prevent DoS attacks and manage
  an unpredictable request load.  This could have been addressed by using a
  chuncked HTTP response, allowing for a much larger page side.
- Paging introduces problematic latency issues, as the client must get a
  complete response and parse the document before it can issue the request for
  the next page. Large pages make for poor interactive responsiveness, and
  small pages lead to a high protocol overhead.
- Paging makes the implementation of a server complex.  Since it
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

### Mixed object type results streams rational

While ReST APIs tend to return one of more objects of the same type, it may be
desirable for a query language API to produce more complex result streams.  It
was felt that the increased expressiveness and efficiency of query languages
will make them important for GA4GH and an, single, flexibly response protocol
should be use for all messages.

One type specification design that was considered was to send a `dataType`
message when the data types changes in the stream.  It was felt that this
could result in degenerate cases of a larger number of `dataType` messages.
It was also thought that it would be a similar level of client complexity to
build a `typeKey` map as to track the type of the stream.  There is also a
robustness advantage in `dataObject` messages having an inexpensive type
identifier include in the message.


### Checkpoint rational

The original query is resent when resuming, along with the checkpoint token,
to provide maximum flexibility for the server developer on how to implement
resuming.  TODO: It's debatable if this is worth the complexity to client
implementation.  The server could encode the query in the checkpoint object.

The checkpoint message is resent at the beginning of the resumed stream to
simplify the implementation of the client.  This avoids the client needing a
part to pass the checkpoint token into the resumed stream.  TODO: However,
if the resume stream fails before the resent checkpoint message is received,
the checkpoint token is needed.  Drop this and document the necessary to
preserved the object and that it's must be good for the resumed stream.


#### todo:
- chuncked and paging can still work, but makes both clientro an serve
- define mime type
- defining versioning
- add line-oriented JSON to protobuf
- GA4GH versioning (if not part of mime type, then allow extension).
- message design on efficent change of many small messsages
- negotation of checkpoint period
- security and encryption
-  This differs from common used paging protocol as it's goal is to
- MIME types
- request streams
- How to send back resume object?  Depend of in original query is required.
- response version header?  no, it's in in content type (specify this)
- raw json

### References
- [rfc7230 - Hypertext Transfer Protocol (HTTP/1.1): Message Syntax and Routing](http://tools.ietf.org/html/rfc7230)
- [rfc7231 -Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content](http://tools.ietf.org/html/rfc7231)
- [Protocol Buffers V3](https://developers.google.com/protocol-buffers/docs/proto3)
