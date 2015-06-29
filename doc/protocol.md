# GA4GH Wire Protocol Specification

### Rational
The GA4GH paging protocol offers a simple client interface that allows clients
to read complete JSON documents 'off-the-wire' and also allows for the easy
resumption of failed transfers.  However, during the course of implementing the
protocol, we are recognizing its drawbacks:

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

Adding an object type and checkpoint token to every object would greatly
increase the overhead for small objects.  However these could only be sent as
needed.  Either as a special object in the stream or by sending batches of
objects.

One possibility for such a streaming protocol would be to have a
stream of messages of the form

    <message type> <length> <message>

Message type indicates the type of the message, not the type data. So we could
have messages of types:

    - D - data object
    - C - checkpoint object
    - T - data type object

One possible example of this in use would be the following stream of messsages
which are the results of a searchVariantSets query:

    T 20 {"type":"VariantSet"}
    D 61 {"id":"variantSet1", "metadata":{}, "referenceSetId":"grch38"}
    D 61 {"id":"variantSet2", "metadata":{}, "referenceSetId":"grch38"}
    D 63 {"id":"variantSet100", "metadata":{}, "referenceSetId":"grch38"}
    C 37 {"checkpoint": "xsdfewsd234wsdfdsf23"}
    D 63 {"id":"variantSet101", "metadata":{}, "referenceSetId":"grch38"}

Clients read in the stream by taking the first two tokens (space delimited,
say, for the sake of argument), and using these to understand the rest of the
message. The first token is a character, telling the client how to interpret
the message body. The second token is the number of bytes in the message
encoded as (for the sake of argument) a base-10 ASCII string. The client then
reads <length> bytes from the stream, interprets them, and moves on to the next
message.

The first message in this stream is a Type message, which lets the client know
how to parse the incoming messages. This simplifies clients, because the stream
parser no longer needs to know the context of the request to be able to parse
the response.

The following three messages are Data messages, which contain the payload we
are interested in. These are shown in JSON here, but they could also be binary
encoded. Because each message is preceded by its length, clients can
efficiently buffer and process messages.

The fourth message is a Checkpoint message. A checkpoint message is something
that the server inserts into the stream at points where it can resume the
stream in the case of an error condition.  If an error occurs, the client
should be able to resume the transfer by providing the server with the
checkpoint object.  (This example is very much a straw man; the details of this
mechanism would need a lot of thought to do properly). Clients who are not
interested in checkpointing simply ignore these messages.
