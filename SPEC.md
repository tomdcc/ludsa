# LUDSA protocol specification

**L**ocal **U**nix **D**omain **S**ocket **A**uthentication

Status: **frozen at version 1.0**. §4.1 states what a later version may change,
and what a peer that predates the change does with it.

This document states what a conforming implementation must do, and nothing about
why. The reasoning is in `RATIONALE.md`. Guidance on how to satisfy these rules
is in `IMPLEMENTATION.md`. Neither is normative.

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted
as described in BCP 14 (RFC 2119, RFC 8174) when, and only when, they appear in
all capitals.

## 1. Purpose and actors

LUDSA is a request/response protocol for one process to defer an authentication
decision to another process on the same machine. The two communicate over a unix
domain socket.

LUDSA is backend-agnostic. The protocol does not state what a credential is or
how to check it. A server can validate JWTs, query a directory, or read a flat
file.

| term | meaning |
|---|---|
| **client** | The party that holds a credential it cannot validate itself. A PAM module, or a PostgreSQL OAuth validator module. |
| **server** | The party that makes the decision. It holds the upstream connections, caches and policy that the decision needs. |
| **host application** | The program that contains the client. PostgreSQL, or `sshd`. |
| **remote party** | Whoever presents the credential to the host application. Neither the client nor the server controls this party. |

Client and server name **roles, not processes**. Where a supervisor binds the
socket and hands a descriptor to another process, the two together are the
server, and a rule binding the server binds whichever of them performs the act.

This document uses only these four terms for the parties to an exchange. A rule
that names the client binds the client. A rule that names the server binds the
server. A rule that names both binds both, in either direction.

Known clients: `pam_ludsa` (PAM module), `pg_ludsa` (PostgreSQL OAuth validator,
planned), `ludsa-probe` (diagnostics), `ludsa-conformance` (test driver).

Known servers: `ludsa-k8s` (Kubernetes TokenReview), `ludsa-file` (reference
implementation, flat file).

## 2. Transport

**Unix domain stream sockets only.** TCP MUST NOT be used, including on
loopback. The socket MUST be a pathname socket; the Linux abstract namespace
MUST NOT be used.

**One request per connection.** The client connects, writes one request, reads
one response, and closes. The protocol has no pipelining and no connection
reuse.

**Half-close after the request.** After it writes the request, a client MUST
shut down its write direction (`shutdown(SHUT_WR)`, `CloseWrite()`). It MUST
leave the read direction open for the response.

The server MUST observe both of these:

- **EOF on the read direction MUST NOT be treated as a closed connection.** The
  connection stays writable, and the server still owes a response.
- **Reading on does not wait.** A request is delivered at its declared octets
  (below), so a server holding them does not need the half-close and does not
  defer its response. One that looks for octets beyond the declared length
  checks what has already arrived. Where a rejection owes a drain (§3), anything
  later is observed there, and a server that has reached the drain has answered.
  A server that answered the message it held owes no drain, and observes nothing
  after its response.

**A server answers once it can.** The obligation begins when it has read
`ludsa_version_major`. That value opens every message, and until the server
holds it there is no version the client chose to answer in. Before that point
nothing obliges it to reply, whether it received no octets at all or fewer than
that value needs: it closes, or answers as the next paragraph allows.

**Answering without one.** A server that has decided to answer but has not read
`ludsa_version_major` MAY answer in its own major version instead of closing. It
MAY instead wait for the version word and answer in the version the client
chose, which is the better answer where it can afford to wait. Where a server in
that position refused at accept, §3 has the refusal owe no drain, so it MAY
bound that wait more tightly than the request deadline, as §5's peer check may.
One that refused nothing waits under the request deadline alone. Where the bound
or the request deadline expires with no version word read, it answers in its own
major version or closes.

§4 already has a server reply in its own major version when it cannot use the
client's; this is the same move where it has not been given one.

Having read it, a server MUST send a response, whatever the request turns out to
be, and replies in its own major version where it does not implement the
client's. The exceptions share a property. A refusal reached at accept (§3)
asked nothing of the peer, so a server that refused there MAY close without
replying, whether or not it read the version word on the way: §5's peer refusal
is one, and a server at its concurrency bound (below) is another. And having
begun writing a response, a server finishes it unless writing fails, its
response deadline expiring included, or a fault leaves it unable to.

A client cannot separate a server that closed on a refusal from one that failed.
It learns only that no response arrived and the connection ended, which §7 names
`RESPONSE_ABSENT`, and nothing about why or about whether the server read its
request at all.

**The server closes after it responds.** Having written its response the server
MUST shut down its write direction (`shutdown(SHUT_WR)`, `CloseWrite()`), and
MUST do so whether or not §3 owes a drain. Where it owes one it completes the
drain after that, with its read direction still open, and closes when the drain
ends. A server owing no drain MAY close outright instead, which ends both
directions at once. There is no connection reuse, so a client that reads on for
end of file never blocks on a draining server. It MUST NOT require end of file
to complete an exchange whose declared octets it already holds, since a server
closing with octets still unread may end that read in an error instead (§7).

**A message is delivered at its declared octets.** A receiver holding every
octet the envelope declared holds the message. Nothing that happens on the
connection afterwards changes what it holds: not the sender's side ending, in
whatever form the receiver's platform reports a close, and not the receiver's
own deadline expiring. A client whose deadline expires after the declared octets
arrived therefore reports the response it holds, and a server answers the
request it holds whether or not the client's half-close has arrived. The
half-close each side owes is its own obligation, and the check below on octets
beyond the declared length is what reaches anything after them.

**Octets beyond the declared length.** A client or server MUST NOT process
anything past the declared length, and SHOULD reject the exchange as
`REQUEST_MALFORMED` if it observes any. Detection is not required to be
exhaustive: excess octets may not have arrived yet, and nothing obliges a
receiver to wait for them.

**Timeouts.** The client and the server MUST both apply read and write
deadlines. This document sets no values for them; they belong to an
implementation and its configuration.

- A client's deadline is an **overall deadline** covering connect, any retries
  and the exchange. It is not a per-attempt deadline.
- The server MUST apply a deadline spanning **accept to complete request**, and
  a separate deadline on writing the response.

When the server's request deadline expires before it holds the declared octets,
it responds `REQUEST_TIMEOUT` (§5) and closes. Where it expires afterwards, the
server holds the request (above) and answers it. Where the deadline expired
before it read `ludsa_version_major`, it answers in its own major version or
closes without replying, as above. Where it has already answered, having
rejected the message early, it closes without replying and §3's drain is what
the deadline was bounding. When its response deadline expires it closes without
replying, because the channel it would answer on is the one that is blocked.

A server MUST respond as soon as it determines it cannot reach a decision, with
`UNAVAILABLE` or `INTERNAL_ERROR` as §5 requires. Whether it bounds the time it
spends trying is its own choice.

A client's own deadline has no representation on the wire, since a status
appears only in a response and the client has already sent its request. The
client reports it locally and never as a `REQUEST_` code. §7 has it report
`RESPONSE_TIMEOUT` where the deadline expired before the declared octets
arrived, whatever had arrived by then. §5 gives `REQUEST_TIMEOUT` to the
server's request deadline, and a client whose own deadline expired has learned
only that the server has not answered yet.

The server SHOULD bound concurrent connections. At its limit it MAY respond
`RATE_LIMITED`, and MAY instead stop accepting and leave connections in the
kernel backlog. The limit is reached at accept, before any octet has arrived, so
a server that chooses to answer reads `ludsa_version_major` first, or answers in
its own major version without waiting for one. The refusal is settled at accept
on §5's terms. A version it does not implement, read while it waits, is answered
`RATE_LIMITED` in its own major version. An expiry while it waits, of the
request deadline or of a bound of its own (above), is `RATE_LIMITED` in its own
major version or a close. §3 has a refusal reached at accept owe no drain,
whatever the server read while waiting.

**Connect-time retry.** A client MAY retry a connection that failed for a reason
a later attempt could change: `ECONNREFUSED` and `ENOENT`, because the server
may not have published its socket yet; `EAGAIN`, because a full backlog may
drain; and `EINTR`, because the attempt was interrupted before it was answered.
It is subject to:

- A client MUST NOT retry a permission failure, a timeout, a protocol error or a
  denial. None of those changes on a second attempt.
- A client MUST NOT open a new connection after it has written any part of the
  request. Resuming a short or interrupted write on the same connection is not
  covered by this rule.
- Every retry happens within the client's overall deadline.

A client that never establishes a connection it is willing to use reports
`CONNECTION_FAILED` (§7). That covers a permission failure, a path that is not
there, a retryable reason that had not resolved by the client's deadline, and a
peer whose identity is not the one the client was configured to expect. No
`RESPONSE_` name fits it, because a response cannot be absent from a request
that was never sent. §3.1's `INVALID_REQUEST` is the other name a client reports
with no exchange behind it, for input it may not put on the wire at all.

**A client may be configured with the server identity it expects**, as a uid.
Where it is, the client MUST confirm the peer's identity before it writes. It
MUST close the connection where it cannot, without issuing a request, and report
the condition §7 names `CONNECTION_FAILED`.

Not confirming covers a peer that is someone else and a peer the client could
not identify. A deployment that asks for a check its platform cannot perform
therefore fails rather than proceeds.

This document names no mechanism, because none works everywhere. Where the two
sides do not share a user namespace, a peer uid is translated or comes back as
the overflow uid. A client with no such configuration is conforming. What the
rule closes is a client that was told what to expect and sent a credential to
something else.

**Socket visibility.** At the path clients use, the socket MUST never be
observable with any of the twelve mode bits `07777` set that `0660` does not, or
with a group other than the one configured for it. A server MUST NOT publish
that path until both are correct, and MUST fail to start if it cannot make them
correct.

That is a bound and not a value. A mode narrower than `0660` conforms, including
one that leaves no client able to connect, and this document does not require a
server to check that anyone can reach the socket it published. The two
directions fail differently: too narrow stops every client at once and a
deployment finds it on its first authentication, where too wide surfaces
nowhere and admits whoever the extra bits admit.

**The socket's group MUST be configured.** There is no default. A server with
none configured MUST fail to start, and MUST NOT fall back to its own primary
group.

**The socket path.** The configured path MUST be absolute. A relative one
resolves against a working directory neither side controls, and a client loaded
into a host process neither chooses that directory nor knows when it changes. A
server configured with a relative path MUST fail to start. A client configured
with one MUST NOT connect, and reports the condition §7 names
`CONNECTION_FAILED`. Neither resolves it against a working directory to find out
what it names.

Each side configures its own path, and the two need not be the same string: a
volume mounted at different points gives one socket two names, and neither side
reads the other's.

A client or server MUST establish that its path is representable in full in the
platform's socket address before it binds or connects, and MUST NOT truncate it
to fit. A path is representable in full where the address structure holds it
with room for a terminator, so the usable length is one less than the array, and
neither side may rely on a platform accepting a path that fills it. Neither call
reports the condition afterwards: a truncated path either names nothing or names
a different socket, and both are indistinguishable from a correct one. A server
whose path does not fit MUST fail to start. A client whose path does not fit
MUST NOT connect, and reports the condition §7 names `CONNECTION_FAILED`. Each
SHOULD name the maximum its platform allows in the diagnostic it emits, since
that is what makes the failure actionable and neither side can learn it from the
other.

A server MUST refuse to start if the configured path exists
and is not a socket, and MUST apply that test before it removes or replaces
whatever is there. Any socket already at that path MUST be gone before the
server publishes its own, whether the server unlinks it or replaces it
atomically. This document requires no test of whether anything is listening on
it: a unix socket outlives the process that bound it, so a path holding one says
nothing about whether a server is running.

### 2.1 Independence of the two sides

The client and the server share nothing but the socket. These consequences bind
every version of the protocol:

- **Nothing is passed by reference.** Every value the server needs MUST travel
  in the message. A field MUST NOT carry a path, a file descriptor, or a
  shared-memory identifier. This holds for every later version.
- **Lifecycles are independent.** The client and the server restart
  independently. A client MUST NOT keep a connection or descriptor between
  requests. The protocol does not guarantee startup order, which is a deployment
  concern.
- **A client survives what this document lets a server do.** A client MUST NOT
  cause its host application to terminate as a result of a write to the socket
  failing. §2, §3 and §5 each authorise a server to close a connection a client
  is still writing to, so that write failing is a conforming exchange rather
  than a fault, and a client is loaded into a host application it does not own
  (§1). This binds nothing about how a client is built beyond that outcome.

## 3. Framing

All integers are in **network byte order (big-endian)**, whatever the host
architecture.

```
envelope := u16 ludsa_version_major  fixed in every version, at offset 0
            u16 ludsa_version_minor
            u32 length               octets following this field

request  := envelope
            message

response := envelope
            message

message  := u32 msg_type             §6, and §5 for ERROR
            field*
            u16 0xFFFF               TAG_END

field    := u16 tag                  §3.3
            u32 len                  dynamic tags only: octets of value
            u8  value[...]           len octets, or the tag's fixed width
```

In that grammar, `uN` is an N-bit unsigned integer in the byte order above, and
`x*` is zero or more repetitions of `x`.

A request and a response have the same shape, and a request type and its
response type are distinct message types (§6). So `msg_type` alone selects a
message's schema, and nothing has to know which direction a message travelled in
order to parse it.

**Only `ludsa_version_major` is fixed across major versions.** It occupies the
first two octets of every request and every response, in every version of this
protocol. Everything after it, including whether there is a length prefix at
all, belongs to the major version it declares.

A client or server MUST read `ludsa_version_major` first, and MUST NOT interpret
any later octet until it has established that it implements that major version
(§4). The rest of this section describes version 1 only.

A client or server that parses a message MUST read `length`, then read exactly
that many octets. It MAY read further from the socket, as a buffered reader
will, but MUST NOT interpret any octet past the declared length and MUST discard
what it read beyond it. §2 governs what it does with octets it observes there.
The envelope lies outside the declared length and is read from the stream
directly.

A receiver that has rejected the message is not parsing it, and this rule does
not bind it. A server drains as this section requires below. A client MAY close
as soon as it has determined what it will report.

A receiver is not required to retain the whole declared length. It MAY discard a
field it does not keep, including an unrecognised optional field it skips
(§3.3), provided it still reads exactly the declared length and interprets
nothing beyond it.

`MAX_LENGTH` is 32768 octets, and bounds the declared `length` alone. The
envelope lies outside it. A client or server MUST accept any declared length up
to `MAX_LENGTH`, so a message this document permits is one every conforming peer
accepts. A declared length greater than `MAX_LENGTH` is `REQUEST_TOO_LARGE`. The
minimum declared length in version 1 is 6, being `msg_type` and `TAG_END`; below
that is `REQUEST_MALFORMED`.

**Where this document names a status, it names the server's.** A client that
detects the same condition in a response reports the name §7 gives it, never a
`REQUEST_` code (§5).

**Closing on a peer that is still writing.** A close cuts off whatever the
counterpart had not finished writing. Reading and discarding first is what
avoids that, and how far the obligation goes follows what the closing side owes
the writer. A server that rejected a message was willing to serve that peer, and
asked for the request it would be cutting off, so it drains. A server that
refused the connection asked for nothing, and no refusal reached at accept owes
a drain: §5's peer check is settled there, and §2's concurrency bound is reached
there. Such a server MAY still read and discard so that a client mid-write is
not cut off, on the bounds below, and one shedding load may decline, since the
load is what its refusal is about. Reading the version word to choose an
encoding does not change that, on either ground. A client owes nothing either,
since nothing is waiting on its read (above). Nor does a server that rejects on
octets beyond the declared length (§2): it has read the message it asked for, so
the drain below does not bind it, and it MAY read and discard on the same
bounds.

A server that rejects a message before it has read all of it MUST NOT allocate
storage to hold what it discards. It MUST send its response, then read and
discard until the earliest of these, and MUST then close:

- it reaches end of file on the read direction, or an error that ends it. An
  interrupted or would-block read is neither;
- it has read `2 x MAX_LENGTH` octets from the connection in total, counting the
  envelope and everything already read before it rejected;
- §2's accept-to-complete-request deadline expires;
- a shorter bound it set on the drain expires. A server MAY set one.

Where it read a declared length it may rely on, it MAY stop once it has consumed
that many octets of the body, counting what it read before it rejected. That is
an optimisation rather than a terminator. A server that reads on to one of the
bounds above conforms, and drains at least as long, which is the direction this
section's purpose points.

§2's deadline runs on through the drain, since the request it bounds never
completed, and it is what guarantees the drain ends. Its expiry here means
close, not answer: §2 has a server respond `REQUEST_TIMEOUT` where the deadline
expires before it has answered, and on this path it has answered already. A
server sends one response to a connection.

Where it expired before the response went out, the earliest terminator is
already met and the drain ends at once. So a server answering `REQUEST_TIMEOUT`
writes and closes, discarding nothing. So does any rejection whose terminator
was already met when the response went out, `REQUEST_INCOMPLETE` among them,
since a message that ended before its declared length ended at end of file. A
rejection reached while every terminator is still unmet drains. The client is
not left without its answer either way, since a response already written
survives the close (§2), and it may take `EPIPE` on a write it had not
finished.

The status it sends is the one it had determined when it rejected the message.
Neither §2's deadline nor a bound of its own, expiring during the drain, changes
it to `REQUEST_TIMEOUT`.

A length a server may rely on is one it read under a major version it
implements, and which passed §4's step 2: within `MAX_LENGTH` and not below the
minimum for the version. Where it has none, the permission above does not apply
and the bounds do.

A server answering `VERSION_UNSUPPORTED` has none, since §3 puts everything
after the version word inside the major version the message declares, so it
knows neither the length nor whether there is one. A server answering
`REQUEST_TOO_LARGE`, or `REQUEST_MALFORMED` for a length below the minimum, has
none either: the number failed the one check that would have made it worth
acting on, and a server that honoured it would close on a peer still writing,
which is what the drain exists to avoid.

**A client or server MUST NOT allocate storage from, or compute an offset from,
any length or count read from a message before checking it against the extent of
the structure that contains it.** That covers `length` against `MAX_LENGTH`, a
field's `len` against the remaining message. The offset is named because it is
reached first: a check written as an addition against the buffer's end overflows
before anything is allocated, and then passes.

A response MUST carry the same `ludsa_version_major` as the request it answers,
whenever the server implements that major version and has read it from the
request. Its `ludsa_version_minor` is the server's own for that major version,
never an echo of the request's, so a server reports what it speaks rather than
what it was addressed in.

Where a server cannot match the client's major version, because it does not
implement the one it was sent or because it read none, it answers in any major
version it implements.

A server MAY tailor a response to the request's `ludsa_version_minor`, by
carrying information in a field the client understands instead of one it does
not. A later version that adds an optional diagnostic field may do that for a
client predating the field. The permission is to substitute and not to withhold:
a server MUST NOT omit, on the strength of a declared minor version, information
it would otherwise have carried. The value MUST NOT inform an authentication
decision (§4), and an older client skips what it does not recognise in any
case.

### 3.1 Field maxima

| field | maximum octets |
|---|---|
| `user` | 256 |
| `service` | 64 |
| `credential` | 16384 |
| `rhost`, `ruser`, `tty` | 256 |
| `authn_id` | 512 |
| `error_detail` | 512 |
| `server_desc` | 512 |

A receiver MUST reject a field above its maximum, except that §5 governs a
client receiving `error_detail`, `server_desc` or `authn_id`. A server reports
`REQUEST_MALFORMED`; §7 says what a client reports. A sender MUST NOT emit a
field above its maximum, and no party may truncate a value to fit. A client
asked to send one MUST NOT open a connection to send it, and reports the
condition §7 names `INVALID_REQUEST` to its host application. This holds whether
the field is required or optional. A client MUST NOT omit the field and send the
rest, because a request without it asks something other than what its host
application asked. Dropping a field, or shortening a value, is the host
application's decision, and it makes it with the report in hand.

A server with a value to carry in a field, and no way to carry it within that
field's maximum, answers `INTERNAL_ERROR` (§5). It MUST NOT truncate the value,
and MUST NOT omit the field in its place. Where this document lets a server omit
an optional field, the permission is about having no value to send rather than
about having one that does not fit, and a client deployed to expect the field
cannot tell those apart. `error_detail` is the exception: a server composes that
value, so §5 has it shorten what it embeds and the message still goes.

No maximum here is set where an ordinary value approaches it, so a server that
meets one has a fault of its own to report rather than a request it can answer.
That is what `INTERNAL_ERROR` is for: waiting will not change it and someone
needs to look (§5).

`error_detail`, `server_desc` and `authn_id` are the only fields in this table
that a server sends and a client receives. A defect in any field a server sends
is a fault in the server that produced it, so a client MUST NOT report a
request-side code for it. Nothing in the client's own request caused it. §5 says
what a client does instead.

When a server rejects a field for length, it SHOULD name the bound in
`error_detail`, and MAY name the field. A field's name is not part of its value,
so naming it discloses nothing (§5).

A fixed-width field carries no maximum, because its tag states its width (§3.3).

### 3.2 Deterministic encoding

When it emits a message, a client or server MUST:

- Emit fields in ascending tag order.
- Emit a tag more than once only where the message type permits it. No version
  1.0 message type does, so under version 1.0 every tag appears at most once.
- Omit an optional field that has no value.
- Emit no tag its own version does not define for that message type.
- Emit a value it is passing on as the octets it was given.
- Emit a `msg_type` its own version assigns, and a request type in a request.
- Emit a value its field's type permits.
- Declare no `length` greater than `MAX_LENGTH`.

Ascending order binds every message, so a receiver checks it against the tag
before it without knowing which schema applies. Whether a repeat is legal does
not: only the message type says. §4 therefore checks the two at different
points, and a receiver that rejects a repeat before it knows the type is
refusing a message it cannot adjudicate.

The rule against emitting an undefined tag binds a sender to its own version and
not to 1.0, because a later version emits tags 1.0 never assigned (§4.1). What
it forbids is emitting a tag no version defines. A receiver skips one, so
nothing breaks today; what breaks is later, when a minor version assigns that
tag and a peer implementing it reads the padding as the field.

It binds a sender. What a receiver does about a breach is §4's, and it turns on
the tag's field number rather than on the version the message declares.

**A sender carries a value as it received it.** The octets a client emits in a
field are the octets its host application supplied. No party normalises a value
it is passing on: no trimming, no stripping a trailing newline, no case folding,
no Unicode normalisation, no re-encoding. §3.1 forbids truncating a value to
fit, and this is the same rule for every other way a value can change in
transit.

A value of zero octets is a value, not an absent field. Whether one is
acceptable is a decision for the receiving application, not for this protocol.

### 3.3 Tags

A tag is a 16-bit value that identifies a field and states how to handle it:

| bits | meaning |
|---|---|
| 15–8 | field number |
| 7 | required when set, optional when clear |
| 6 | dynamic when set: a `u32 len` follows the tag |
| 5–4 | width when bit 6 is clear: `00` = 1, `01` = 2, `10` = 4, `11` = 8 octets |
| 3–0 | type, from the table below |

The field number is the high octet, so a tag's numeric order is its field
number's order. §3.2's ascending rule therefore sorts by field number, and a
field a later version adds sorts after every field already defined.

`0xFFFF` is `TAG_END` and MUST NOT be used as a field tag.

**Field number 0 is reserved and MUST NOT be assigned.** Field numbers run
from 1.

| type | value | length | rules |
|---|---|---|---|
| `integer` | `0x0` | fixed | unsigned |
| `enum` | `0x1` | fixed | §3.4 |
| `string` | `0x2` | dynamic | MUST NOT contain a NUL octet |
| `utf8` | `0x3` | dynamic | MUST NOT contain a NUL octet; MUST be valid UTF-8 |
| `bytes` | `0x4` | dynamic | none |

A sender MUST NOT emit a value its field's type does not permit (§3.2). A client
asked to send one MUST NOT open a connection to send it, and reports the
condition §7 names `INVALID_REQUEST` to its host application. §3.1 says the same
for a value above its maximum. Neither lets a client alter the value to make it
sendable (§3.2).

**A tag's shape is checked before its identity.** A receiver MUST reject a tag
as `REQUEST_MALFORMED` on any of these grounds, whether or not it recognises
the tag, and whether the tag is required or optional:

- bit 6 is set and either of bits 5-4 is set;
- bit 6 disagrees with whether the tag's assigned type is fixed or dynamic;
- the field number is zero.

The checks on bits 5-4 and on the field number read only bits every tag
carries, so they reach a tag whose type is unassigned like any other. The check
against the assigned type cannot: nothing states whether an unassigned type is
fixed or dynamic, so a tag carrying one that passes the others is unrecognised
rather than malformed, which is what keeps §4.1's promise about a new value in
the type table.

A tag that cannot describe a field is not a field to skip, and skipping one
would mean trusting a width the tag itself contradicts. Field number 0 is
reserved, so no tag carrying it names a field, and rejecting it is what makes
zero safe as the initial value of the previous-tag comparison §3.2 describes.
Holding bits 5-4 at zero on a dynamic tag is what lets a receiver compare a tag
as one whole 16-bit value. Only a tag that passes every one of those checks is
compared against a schema.

**A tag is validated whole.** A client or server compares a received tag against
the tags its schema defines for that message type. It MUST NOT read a value out
of a field according to the type the tag claims when the tag as a whole is one
it does not recognise. How far it skips is what the skip rule below states: bit
6, and bits 5-4 where bit 6 is clear.

**A message type's ceiling** is the highest field number the receiver's own
version assigns in that message type. Where its version assigns no field there,
every field number is above the ceiling. The ceiling is a property of the
receiver's schema and never of the message, so nothing the sender declares moves
it.

For a tag it does not recognise:

- **Required**: it MUST abort the exchange and return `REQUEST_MALFORMED`.
- **Optional**: where the tag passed the shape checks above, its field number
  decides. Above the ceiling, it MUST skip the field without error, using bit 6
  and, when clear, bits 5–4 to know how many octets to skip. §4 qualifies that
  skip alone, where the message declares a minor version the receiver
  implements. At or below the ceiling, no version can give the tag meaning.
  §4.1 freezes a number's assignment, and requires a later version to take a
  number higher than every number already assigned. The sender is broken, and
  the receiver MUST abort the exchange and return `REQUEST_MALFORMED`.

A receiver that skips such a tag skips every occurrence of it, including one
equal to a tag it has already skipped. Whether a repeat is legal is a property
of the message type's schema (§3.2), and a receiver holds no schema for a tag it
does not recognise. A later minor version may assign a repeatable optional field
(§4.1), and skipping each occurrence is what lets a predating peer read a
message carrying one.

A value of type `integer` is unsigned, at the width its tag states. Version 1.0
defines no signed type.

"Valid UTF-8" means well-formed UTF-8 as RFC 3629 and Unicode Table 3-7 define
it: no overlong encoding, no code point from U+D800 to U+DFFF, and none above
U+10FFFF. A receiver applies exactly this definition and no other.

Type values `0x5` to `0xF` are unassigned. A tag carrying one is unrecognised
like any other, and the rule above for an unrecognised tag applies to it: an
unassigned type is unknown rather than impossible, and skipping it is what keeps
§4.1's promise about an optional field a later version adds.

The table gives the constraints a type implies. A receiver MUST reject a value
that violates its type's rules, except that §5 governs a client receiving
`error_detail`, `server_desc` or `authn_id`. A field's own definition may add
others, and §3.1 and §6 hold all of them. A client or server MUST NOT impose
any beyond those. In particular it MUST NOT reject a control character except
where a field's own definition forbids one, and no type but `utf8` requires a
particular encoding.

### 3.4 Enumerated values

Every enumeration in this protocol has the same shape:

| value | meaning |
|---|---|
| 0 | unassigned, and never affirmative |
| 1 | the value a receiver uses for one it does not recognise |
| 2 | the affirmative outcome, where the enumeration has one; unassigned where it has none |
| 3 and up | the remaining specific outcomes |

Each enumeration names its own 1, so the name says which enumeration it belongs
to: `STATUS_UNKNOWN` (§5), `AUTHN_UNKNOWN` (§6.2.1), `SERVING_UNKNOWN` (§6.3).
What this section fixes is the number, because that is what travels.

A sender MUST NOT emit 0 or 1, and MUST NOT emit a value its own version does
not assign in that enumeration. §4.1 makes a new value a minor bump, so what a
sender may emit is what its own version defines. §3.2 binds `msg_type` and a
field's type on the same terms. A receiver that meets 0 or 1, or any other value
it does not recognise, MUST treat it as 1, and MUST NOT treat it as an
affirmative outcome.

**No value that names a non-affirmative outcome in one enumeration is
affirmative in another.** Holding the affirmative outcome at 2 everywhere, and
leaving 2 unassigned in an enumeration that has none, is what secures that. A
receiver that reads an enumerated value out of the wrong message type therefore
cannot turn a refusal into an approval, whatever else it gets wrong.

Two enumerations do both put an affirmative outcome at 2, `result`'s `OK` and
`serving`'s `SERVING`, so this section does not keep a health check from reading
as a grant, and §6.4's tags do not either: `result` and `serving` are both
`0x0181`, each in its own schema. §6 does, by binding a client to establish
`msg_type` before it reads any field, and §4's order rejects a response type the
request does not permit before any field is read.

**Zero is never affirmative**, and the rule reaches past this section. A field
number of 0 names no field (§3.3). `msg_type` `0x00` is `ERROR` (§5), which
reports that the server decided nothing. A `ludsa_version_major` of 0 is a
version nobody implements, so a message of nothing but zeroes stops at the first
step of §4's order. A zero-filled buffer cannot express an approval anywhere in
this protocol, in either direction.

## 4. Versioning

`ludsa_version_major` and `ludsa_version_minor` open every message. This
document specifies version **1.0**.

**The major version is the compatibility contract.** A client or server MUST
reject a message whose `ludsa_version_major` it does not implement. It is the
only value either side may test to decide whether the two can communicate.

Recognising a tag in such a message does not license parsing it. §4.1 constrains
major version 1 only, so a later major version may reassign a tag or change what
a field means, and a familiar tag therefore carries no guarantee about its
meaning under a major version the receiver does not implement.

A server that receives a `ludsa_version_major` it does not implement MUST reply
`VERSION_UNSUPPORTED` (§5) in its own major version, except where a refusal
reached at accept displaces it or relieves it of answering at all (§2, §5).

A client that does not implement the version that reply carries cannot read the
status, because §3 forbids interpreting any octet past a version word it does
not implement. It reaches the same conclusion from the version word itself,
which §3 fixes at offset 0 of every message in every version. The status is for
a peer that implements both versions, and the envelope carries the outcome to
one that does not.

**A recommendation to later major versions**, which this document cannot bind.
A server implementing major version N should be able to frame
`VERSION_UNSUPPORTED` as any major version from 1 to N, and a client
implementing N should be able to read one framed as any of them. A server would
then answer in the highest version it implements that does not exceed the
version the client declared. That version is one both sides read, so the
mismatch states itself instead of being inferred from the envelope.

A version 1 server has only its own version to answer in, which is what the rule
above requires of it. Version 1 is the oldest, so the recommendation costs a
version 1 implementation nothing and the whole obligation falls on later
versions. That is why it is written here rather than in the document that first
needs it.

**The minor version is informational.** A client or server MUST accept a message
whose `ludsa_version_minor` differs from its own, in either direction, and MUST
parse it under the rules of §3.3. A higher minor version can only add fields
that are safe to skip.

**A receiver parses by tag and not by the declared version.** It matches a tag
against its own schema for that message type, and the version a message declares
never selects the schema it parses under.

That value settles less than it appears to. What a receiver does with a tag it
does not recognise is decided by the tag's field number, not by the version the
message declares. §4.1 fixes what a number may mean before any message declares
anything.

Where the field number is at or below that message type's ceiling (§3.3), no
version can give the tag meaning. Either the number is assigned there already,
and §4.1 freezes that assignment. Or it is one every version skipped, and §4.1
requires a later version to take a number higher than every number already
assigned. The sender is broken and the receiver can prove it, so it MUST reject
the message as `REQUEST_MALFORMED`, whatever minor version the message declares.
§3.3 carries the rule.

Where the field number is above the ceiling, an optional tag may be one a later
version assigns, and the receiver holds no schema that would settle it. It MUST
skip, as §3.3 has it skip any other. Rejecting there would fail a conforming
peer.

A required tag is settled at any field number. A field a later version adds is
optional, since a required one forces a major bump (§4.1), so a required tag a
receiver does not recognise is one no version within that major version defines.
§3.3 has it abort and return `REQUEST_MALFORMED`.

The declared minor version qualifies that second case alone. A message declaring
a minor version the receiver implements has broken §3.2 by carrying the tag at
all, so a receiver MAY reject it on that ground instead of skipping. Neither is
preferred and both conform. A deployment that wants strict input in an
authentication path may have it, and one that wants tolerance may have that. A
receiver implements every minor version at or below its own within a major
version it implements, since each minor version's schema contains the ones
before it (§4.1). A declared minor version above the receiver's own is one it
does not implement, and there it skips.

The minor version is for logs, and a server MAY tailor what it sends to the one
a request declares (§3). It MUST NOT inform an authentication decision, and it
states nothing about which optional fields a peer populates: a field a later
version adds is optional (§4.1), so its absence never separates a peer that
predates it from one that had no value for it.

The protocol has no negotiation. Each side declares its own version, and §3.3
makes a minor difference safe without either side agreeing to anything.

**Validation proceeds outermost first**, and a receiver holding the outcomes of
more than one of these checks reports the earliest:

1. `ludsa_version_major` implemented
2. declared length within `MAX_LENGTH` and not below the minimum
3. the declared octets all arrive
4. framing, independent of any schema: tag shape (§3.3), no tag lower than the
   tag before it, each field's extent, `TAG_END`
5. `msg_type` implemented, which binds a server; a client takes step 6 for any
   `msg_type` it did not expect
6. for a response, a `msg_type` the request permits (§6)
7. tags recognised for that message type, and repeated only where it permits
8. field maxima (§3.1)
9. content of recognised fields: type rules (§3.3)
10. required fields present for that message type
11. selectors, such as `service`
12. limits, such as `max_age`
13. the decision

The peer check §5 describes is not one of these steps. It concerns the
connection rather than the message, and §5 has it made on accept, before any of
this order has run. A server that refuses there and then waits for
`ludsa_version_major` may reach step 1's status as well, and §5 has
`PEERCRED_DENIED` displace it. §2's concurrency bound is reached at accept in
the same way, and `RATE_LIMITED` displaces it there too: a refusal reached at
accept was settled before the version could be read, and nothing the version
draws changes it. A server whose rules also govern which peers may reach which
`service` values cannot settle that second question before step 11.

§2's check for octets beyond the declared length is not one of these steps
either. Those octets lie outside the message (§3), so the check concerns the
connection rather than the message, and a receiver may observe them at any
point. Where it observes them before it has written its response, §2's SHOULD
reaches whatever outcome this order had produced, the decision included. Where
it observes them afterwards it has already answered, and §3 has a server send
one response to a connection.

Step 4 checks only what a receiver can check without knowing which schema
applies. A tag equal to the tag before it is a repeat, which step 7 judges
against the schema. Steps 7 to 10 need the schema that `msg_type` selects, and
cannot precede step 5. Step 6 binds a client only, since only a client has a
request to compare against.

Step 3 is the only step that requires waiting. Every other check runs on octets
already read, so a receiver MAY report a fault it has already detected without
waiting for the declared length to arrive, and the order binds what it holds
rather than what a later octet would have given it. A message that is both
truncated and faulty in the octets that arrived may therefore take either
status, and both conform. A message truncated with no fault in the octets that
arrived is `REQUEST_INCOMPLETE`. The same follows wherever an earlier step's
fault lies beyond the octets a receiver acted on: a receiver that read a
`msg_type` it does not implement and answered may not have reached a framing
fault after it, and one that ran the framing pass before it dispatched holds
that fault and answers it as the order requires. Both conform.

**None of that is forced, deliberately.** Forcing the fault would oblige a
receiver that reads a message whole to parse a prefix it has already given up
on. Forcing the truncation would oblige a receiver that validates as it reads to
wait for the rest of a message it has already judged. Each rule conscripts one
way of reading a socket, and nothing this protocol governs turns on the answer:
a peer that sends a message both short and faulty is broken twice over, and one
that stopped because it died is not reading the response whichever status it
gets.

Steps 10 to 13 are reached only once the declared octets have arrived, whatever
a receiver observed before then. A required field is missing only once the
message ends. A selector or a limit is legible as soon as its own field is, and
the permission above covers a fault alone. A field number does not order these
steps and can run against them: an `AUTHENTICATE_REQUEST` carries `service` at
field 1 and `credential` at field 2, and a `STATUS_REQUEST` carries `max_age` at
field 1 and `service` at field 2. A receiver that answered from one of those
before the declared octets arrived has taken the order backwards.

Steps 8 and 9 yield the same status, so the order between them is not
observable and a receiver taking them the other way round conforms. The order
shown spares a receiver from scanning a value it is about to reject.

### 4.1 What a later version may change

Within major version 1, a later version MUST NOT change any of the following:

- A tag. Its number, type, width and required status are all part of its value.
- A field number, once assigned in a message type, for anything else in that
  message type, or whether it is assigned at all. A later version MAY deprecate
  a field, and MUST keep accepting it on the terms of the version that assigned
  it, so no number is ever removed or reassigned. Numbers are independent across
  message types (§6).
- The message type a `msg_type` code identifies.
- The response types a request type permits (§6).
- A code's number or meaning, in any enumeration.
- A field maximum (§3.1).
- `MAX_LENGTH`, the minimum declared length, and the envelope's layout (§3).
- The tag bit assignments and the value of `TAG_END` (§3.3).
- Whether an assigned tag may repeat. A tag assigned as non-repeatable stays
  non-repeatable, because a peer that predates a loosening holds the schema for
  that tag and would meet duplicates it has no rule for.

A new major version MAY change any of them. That is what a major version is, and
why §4 forbids a client or server from parsing a message whose major version it
does not implement.

Additions are possible within major version 1. The kind of addition determines
which number changes, and what a peer that predates the addition does with it:

| addition | bump | a peer that predates it |
|---|---|---|
| an **optional** field | minor | skips the field (§3.3) |
| a new **message type** | minor | answers `REQUEST_UNSUPPORTED` (§6) |
| a new value in any **enumeration** | minor | treats it as that enumeration's 1 (§3.4) |
| a new value in the **type** table | minor | skips an optional field using it (§3.3) |
| a **required** field | major | answers `VERSION_UNSUPPORTED` (§4) |

A later version MUST NOT assign a field that takes a message type's derived
maximum (Appendix A) above `MAX_LENGTH`. `MAX_LENGTH` is frozen and every field
maximum is frozen, so each message type's headroom is fixed at 1.0 and an
addition spends it. Without this a version could define a message legal under
its own schema that every conforming receiver, one implementing that version
included, MUST reject as `REQUEST_TOO_LARGE` (§3).

A new optional field MAY be repeatable from the version that assigns it. A peer
that predates it does not recognise the tag, so it skips each occurrence (§3.3)
rather than adjudicating a repeat it has no schema for. What it may not do is
become repeatable later, which is the frozen rule above.

A field a later version adds to a message type MUST take a number higher than
every number already assigned in that type. That is what makes §3.3's ordering
property hold: the field sorts after every field already defined there, so an
encoder appends rather than inserting. It is also what lets a receiver reject a
tag it can prove no version defines (§3.3), since a number at or below that
type's ceiling can never acquire a new meaning. Version 1.0 assigns every number
in sequence, and a number a later version skips stays empty.

A required field is the only addition here a predating peer never sees. It stops
at step 1 of §4's order, on the major version. It reaches no tag. §3.3's rule
for an unrecognised required tag covers the other path, where the same field
ships in a minor version by mistake. That one draws `REQUEST_MALFORMED`.

## 5. The ERROR message

A server answers with an `ERROR` message whenever it does not answer with the
response type §6 pairs with the request it was sent. `ERROR` is `msg_type`
`0x00`. It appears only in a response, and it is the only message that carries a
status.

Response:

| field | type | | |
|---|---|---|---|
| `status` | `enum`, 1 octet | required | `0x0181` |
| `error_detail` | `utf8` | optional | `0x0243` |

§3.2 binds a server to the fields its own version defines, which for version 1.0
are these two, whatever the request it answers. §3.3 governs any other tag a
receiver meets. A server SHOULD populate `error_detail`.

`status` states how the server handled the message. It states nothing about what
the server decided, because an `ERROR` message means it decided nothing.

| code | name | meaning |
|---|---|---|
| 1 | `STATUS_UNKNOWN` | a status this implementation does not recognise |
| 3 | `REQUEST_MALFORMED` | the request was structurally wrong |
| 4 | `REQUEST_INCOMPLETE` | the request ended before it was complete |
| 5 | `REQUEST_TOO_LARGE` | the declared length exceeded `MAX_LENGTH` |
| 6 | `REQUEST_UNSUPPORTED` | the message type is not implemented |
| 7 | `VERSION_UNSUPPORTED` | `ludsa_version_major` is not implemented |
| 8 | `REQUEST_TIMEOUT` | the request did not arrive within the server's deadline |
| 9 | `UNAVAILABLE` | no verdict, and waiting may change that: a dependency that did not answer and may answer later |
| 10 | `INTERNAL_ERROR` | no verdict, and waiting will not change that: a fault that needs someone to look |
| 11 | `RATE_LIMITED` | the server declined under a limit of its own: the work a request would have required, or the connection carrying it |
| 12 | `PEERCRED_DENIED` | the server declined the connection on the peer's credentials |

This enumeration has no affirmative case at all, so §3.4 leaves both 0 and 2
unassigned in it. A response to a request the server answered carries a message
of that request's type, so there is no successful outcome for a status to state,
and holding 2 empty keeps every affirmative value in the protocol at one number
a status never carries.

Every code a server emits is its statement about the request it was sent. The
`REQUEST_` prefix says so on the codes that name a fault in it. The unprefixed
codes describe the server's own state or the pair's compatibility, so they carry
no direction, and §7 has a client relay every status it read. `STATUS_UNKNOWN`
belongs to neither family and a server never emits it: §3.4 forbids a sender
from emitting any enumeration's 1, and it is what a client records for a status
it does not recognise. §7 gives a client the names for what it finds in a
response.

A client MUST NOT report a `REQUEST_` code for a fault it found itself. The
names exist so a caller can tell the two apart. An operator reading
`INVALID_REQUEST` (§7) shortens a credential. One reading `REQUEST_TOO_LARGE`
has a peer that declared a length §3 forbids it to send. One reading §7's other
response codes goes to the server.

`REQUEST_MALFORMED` means the sender produced something §3, §3.1, §3.2, §3.3 or
§4 forbids it to send, other than a condition §5 gives a status of its own.
Invalid structure, a declared length below the minimum for the version, a value
its type rules reject, a field above its §3.1 maximum, a required field absent,
a tag repeated where its message type permits it once, and an unrecognised tag
that §3.3 and §4 have the receiver reject rather than skip are instances. It
also covers octets a receiver observed beyond the declared length (§2), which
lie outside the message and are still something §2 forbids the sender to put on
the connection. A message that stopped early is `REQUEST_INCOMPLETE` instead,
because a peer that crashed or was killed produced a short message with no
defect in its encoder. `REQUEST_INCOMPLETE` covers a message that ended before
its declared length, and one that ended before it declared a length at all.

`REQUEST_TOO_LARGE` is what the envelope decides. A receiver reaches it from the
declared length alone and answers without parsing any further, which is the
whole of its value. A field above its §3.1 maximum is not `REQUEST_TOO_LARGE`,
because a receiver finds one only inside a body it has already accepted.

`PEERCRED_DENIED` is for a server that checks the connecting peer, which this
document does not require. **A server configured to check MUST do so
immediately after it accepts the connection**, before it reads anything from it.

A server whose check fails owes the peer nothing it asked for, and §2's terms
for a refusal reached at accept govern what it may do. These are the shapes it
takes:

- closes the connection without replying;
- answers `PEERCRED_DENIED` at once in its own major version, having read
  nothing;
- waits a bounded interval of its own choosing to read `ludsa_version_major`,
  and answers `PEERCRED_DENIED` in the version the client declared, or closes
  without replying, as §2 lets any refusal reached at accept do.

The third reads a version to choose an encoding and for nothing else. A version
this server does not implement, and an interval that expires with none read, are
both answered `PEERCRED_DENIED` in the server's own major version. Nothing
displaces it, because the refusal was settled before any other answer could
arise and nothing the peer sends can change it. A server refusing on this check
and at its concurrency bound together answers `PEERCRED_DENIED`.
`VERSION_UNSUPPORTED` and `REQUEST_TIMEOUT` describe what the peer did
afterwards. The `RATE_LIMITED` §2 gives a server at its concurrency bound is
reached at accept alongside this check, and it would tell a peer that a later
attempt may fare better, which is never true of a refusal on identity.

Such a server owes no drain and MAY read and discard, on §3's terms. It parses
nothing beyond `ludsa_version_major`, and holds no length it may rely on, so
§3's permission to stop early does not apply and its bounds do.

A server whose rules also govern which peers may reach which `service` values
has a second gate. It cannot close that one before it has read a `service`. Its
credentials check still happens at accept. What waits is which services this
peer may reach, a question §4's step 11 settles and `PEERCRED_DENIED` also
answers. The gate concerns services the server holds: a `service` it does not
hold draws `SERVICE_UNSUPPORTED` (§6.2.1, §6.3) whoever the peer is, since there
is nothing behind the name for the gate to guard. A request too malformed to
yield a `service` never lets it decide, so the framing status is the only answer
it has.

It discloses that the peer was refused and why. That is a smaller disclosure
than it looks, because a uid is not freely chosen: an attacker must already be
running as one to try it, where a user name can be guessed without limit. A
server unwilling to disclose even that closes without replying. That is closing
without replying, whether or not it read the version word first, and it is the
answer that says nothing.

`RATE_LIMITED` is for a limit the server set on itself, on work or on
connections, and the producers below are instances. One is §6.3's, where a
server declines a check that `max_age` would have forced, which is work a
request asked for. Another is §2's, where a server declines at its concurrency
bound, which is reached at accept before any octet has arrived, so nothing has
asked for anything yet. A server declining, under a limit of its own, the work
an AUTHENTICATE request would have required is in the same position. What they
share is the only thing a client acts on: the server had capacity for this and
has none now, so a later attempt may fare better.

`REQUEST_TIMEOUT` describes the arrival of the request, not the time spent
reaching a verdict. A server that bounds a dependency call reports `UNAVAILABLE`
when that bound expires, never `REQUEST_TIMEOUT`.

`UNAVAILABLE` and `INTERNAL_ERROR` both mean no verdict was reached, and they
separate the two causes by whether waiting could change the answer.
`UNAVAILABLE` covers a dependency that may answer later. `INTERNAL_ERROR` covers
a fault in the server itself, including one in its own configuration or in data
it loaded: an unparseable line in a backing file, or an expired trust bundle,
will not fix itself, and needs someone to look. A server MUST NOT report one as
the other.

Where a dependency is unusable and will stay unusable until someone acts, such
as one refusing the server's own credentials or a trust bundle that expired,
`INTERNAL_ERROR` governs. The test is whether waiting could change the answer,
not whose component failed. Appendix B keys its PAM mapping on the same
distinction.

`error_detail` is for the server's log and the host application's log. A client
MUST NOT return it to the remote party, and MUST NOT pass it to its host
application by any route that reaches the remote party. It MUST NOT contain any
part of a `credential`. A server describing a defect in one describes the
condition and not the octets: "exceeds 16384 octets".

It MUST NOT contain a control character, meaning U+0000 to U+001F, U+007F, and
U+0080 to U+009F, so that a receiver may write it as a datum to a line-oriented
log without escaping it. That is a statement about the octets and about nothing
else. It says the value needs no escaping; it does not say what a receiver may
hand it to. A server that embeds text from elsewhere in it MUST render that text
safely first, and MUST NOT let that text push the field above its §3.1 maximum.
It shortens what it embeds, not the message. §3.1's rule against truncating a
value reaches a server carrying one and a receiver rejecting one, and names
`error_detail` as the exception because a server describing a condition composes
the value.

A client MUST NOT log `error_detail`, return it, or otherwise act on it until it
has established that the value is valid UTF-8 (§3.3), holds no control character
and is within its §3.1 maximum. A client that does not use the value need not
check it. One that checks and finds any of those faults SHOULD report the
response as malformed (§7). The same holds for `server_desc` and `authn_id`. The
value is the octets the field's length declared, and a client MUST make these
checks over that extent, before it converts the value to any form that carries
no length. Terminating it first substitutes a shorter value and checks that,
which a NUL makes silent: two identities differing only past the NUL arrive as
one.

A server is bound to emit none of those faults. §5 forbids the control
character, §3.1 bounds the length, and §3.3 forbids the NUL and the encoding,
which §3.2 binds a sender to. So a client meeting one has found a bug in the
server rather than a value to work around, and reporting it is what gets the bug
fixed. More tolerant handling is a client's own choice, and this document does
not specify it.

Tolerance has one bound. A client MAY render a defective value in a transformed
form to log it or report it, since a fault nobody can describe is a fault nobody
fixes. It MUST NOT act on the transformed form as though it were the value.
Substituting a replacement character for an invalid sequence in `authn_id` and
then using the result as an identity is the case that matters, because two
identities differing only in the defective octets become one.

The control-character range is over code points, not octets, and the test falls
out of the same pass that validates the encoding. A receiver MUST NOT test the
octets `0x80` to `0x9F` directly, because those occur as continuation octets
inside characters that are not control characters.

## 6. Messages

`msg_type` is a u32 opening every message. It is not a tag, and §3.3 does not
apply to it. Version 1.0 assigns the values below.

| `msg_type` | name | |
|---|---|---|
| `0x00` | `ERROR` | §5 |
| `0x01` | `PING_REQUEST` | §6.1 |
| `0x02` | `PING_RESPONSE` | §6.1 |
| `0x03` | `AUTHENTICATE_REQUEST` | §6.2 |
| `0x04` | `AUTHENTICATE_RESPONSE` | §6.2 |
| `0x05` | `STATUS_REQUEST` | §6.3 |
| `0x06` | `STATUS_RESPONSE` | §6.3 |

**A request type and its response type are different message types.** Each
carries its own schema, and §6.4 gives each its own group. That is what lets
`msg_type` select a schema on its own, without a receiver having to know which
direction a message travelled.

**These codes are opaque.** A receiver MUST NOT derive a message's direction,
or the request a response answers, from arithmetic on a `msg_type`. This table
and the one below are the only authorities, and a later version MAY assign a
code that fits no pattern visible here.

A response to a request carries one of these:

| request | a response carries |
|---|---|
| `PING_REQUEST` | `PING_RESPONSE` or `ERROR` |
| `AUTHENTICATE_REQUEST` | `AUTHENTICATE_RESPONSE` or `ERROR` |
| `STATUS_REQUEST` | `STATUS_RESPONSE` or `ERROR` |

A server that responds MUST send one of them, and §2 says when it responds. A
client MUST establish a response's `msg_type` before it reads the value of any
field in it. `ERROR` is the only message that carries a status, so a response
carrying any other type is itself the statement that the server answered the
request it was sent.

§4.1 forbids a later version within major version 1 from reassigning any code in
either table, or from changing the response types an existing request type
permits. It may add a request type and its response type, in a minor version.

Every other value is unassigned. A server MUST respond `REQUEST_UNSUPPORTED`
(§5) to a `msg_type` it does not accept, whether because it is unassigned,
because the server does not implement it, or because it is a response type or
`ERROR`, which a server never accepts.

A client has nothing to respond with. It reports a response whose `msg_type`
the request does not permit, and §7 gives it two names for that. A `msg_type` it
does not implement, whether unassigned or from a later minor version, is
`RESPONSE_UNSUPPORTED`. One it implements but the request did not permit,
including a request type or a response type belonging to a different request, is
`RESPONSE_MALFORMED`.

Both are defects in the server. §4.1 freezes the response types a request type
permits, so no conforming server sends either, at any minor version. The two
names separate a server that sent something the client has never heard of from
one that sent a type the client knows and should not have received.

**Tags are scoped to one message type.** The same field number in an
`AUTHENTICATE_REQUEST` and an `AUTHENTICATE_RESPONSE` are unrelated, because
those are different types. §6.4 lists every assignment in one place.

### 6.1 PING (`0x01` request, `0x02` response)

Tests liveness, and reports the server's protocol version at startup. The
response envelope carries that version, so PING answers the compatibility
question without any field. The fields below are diagnostic.

Request: no fields.

Response:

| field | type | | |
|---|---|---|---|
| `server_desc` | `utf8` | optional | `0x0143` |
| `uptime_seconds` | `integer`, 8 octets | optional | `0x0230` |

`server_desc` names the server implementation and its release, for example
`ludsa-k8s v1.2.3`. The format is conventional and this document does not
specify it. It is for logs and diagnostics only. A client MUST NOT parse it, and
MUST NOT treat it as a protocol version. The envelope carries the only version
either side may test (§4). It MUST NOT contain a control character, on the terms
§5 sets for `error_detail`.

`uptime_seconds` is whole seconds since the server began serving on this socket.
It is for logs and diagnostics only. A client MUST NOT let it inform an
authentication decision, and MUST NOT infer availability from it. A server that
cannot determine it omits the field.

### 6.2 AUTHENTICATE (`0x03` request, `0x04` response)

The primary operation.

Request:

| field | type | | | source | trust |
|---|---|---|---|---|---|
| `service` | `string` | required | `0x01C2` | host configuration | trusted |
| `credential` | `bytes` | required | `0x02C4` | end user | the value to validate |
| `user` | `string` | optional | `0x0342` | client application | claim, MUST be verified |
| `rhost` | `string` | optional | `0x0442` | host observation | trusted |
| `ruser` | `string` | optional | `0x0542` | host application | claim |
| `tty` | `string` | optional | `0x0642` | host application | claim |
| `pid` | `integer`, 4 octets | optional | `0x0720` | client's own `getpid()` | self-reported, logging only |

`user` is the identity that the remote party requests, and the remote party
controls it. **It is optional, and sending it decides who enforces policy.** A
client sends it when it wants the server to decide whether the credential's
identity may act as that user, and omits it when it will decide for itself. Some
credentials identify their subject, so a client holding one may need nothing
from the server but validation.

When `user` is present, a server MUST NOT answer `OK` unless the credential
identifies the requested `user`, or the server's policy maps the credential's
identity to it. A valid credential alone is never sufficient.

A client that normalises `user` asks about a name its host application did not
ask about, and an `OK` then licenses a role nobody requested. §3.2 forbids it.
What a server's policy does with the name it received is that deployment's
decision, and §3.2 does not reach it.

An absent `user` and a present `user` of zero octets are different requests. The
first asks the server not to decide. The second asks it to decide about the user
named by the empty string, which §3.2 makes a value like any other. A server
MUST NOT treat them alike.

**A server that denies chooses how much to say.** `AUTHN_DENIED` says only that
it declined. The rest name a reason, and §6.2.1 defines them.

Naming a reason helps a legitimate client and whoever reads its logs. It also
discloses. `AUTHN_DENIED_USER_UNKNOWN` tells a client which user names exist,
and that reaches the remote party through whatever the host application does
differently for an account it does not know. The disclosure does not stop at the
codes a server names. A code left covering one case alone names it as surely as
a code that says so, so a server that names some reasons discloses whatever its
remaining code is left covering.

Which reasons to name is a deployment decision, and this document does not make
it. A server SHOULD offer both a mode that names a reason and one that answers
`AUTHN_DENIED` for every denial, and leave the choice to whoever runs it.
Diagnosis and traceability weigh against disclosure, and a reason often reaches
a log and goes no further, since what reaches the remote party is the host
application's behaviour rather than this protocol's.

When `user` is absent, `OK` states that the credential is valid and that the
server raised no objection of its own. A server MAY still apply policy to the
identity it extracted, and MAY answer `AUTHN_DENIED_POLICY` on that basis.
Nothing obliges it to, and nothing on the wire tells a client whether it did.

So a client MUST NOT treat `OK` as an authorisation decision when it sent no
`user`. It MUST NOT grant anything unless it has established which identity the
credential names, from `authn_id` or by its own examination of the credential,
and has applied its own policy to that identity. Omitting `user` moves that
obligation rather than removing it, and a server that happens to apply policy of
its own does not discharge it.

`service` states which policy applies. The remote party does not influence it,
so a server MAY route on it. A server that does not recognise a `service` it was
sent MUST answer `SERVICE_UNSUPPORTED` (§6.2.1), and MUST NOT fall back to a
default policy. It recognises a `service` by comparing octets, and MUST NOT
case-fold, trim or normalise the value to find a match. §3.2 delivered the
octets the client's configuration carried, and a name differing in any of them
is a different name. This differs from `user`, where the received name goes to a
policy and §6.2 leaves that deployment its own decision, because `service`
selects which policy runs.

`credential` is carried **verbatim, as raw bytes with an explicit length**, and
is never re-encoded. §3.2 binds every sender to this, and here it binds the
server too: **neither party may normalise it.** No trimming, no stripping of a
trailing newline, no case folding. A signature covers the octets that were
signed, so a value altered on either side is a different value. A server SHOULD
still detect leading or trailing whitespace and record the condition, not the
value, in its own log. A credential that fails validation is a decision
(§6.2.1), and a decision carries no field for a diagnosis.

`pid` is for log correlation only. The client self-reports it, and it MUST NOT
inform any decision.

Response:

| field | type | | |
|---|---|---|---|
| `result` | `enum`, 1 octet | required | `0x0181` |
| `authn_id` | `utf8` | optional | `0x0243` |

`authn_id` is the canonical identity the server resolved, for example
`system:serviceaccount:app:web`. A client that cannot use it ignores it. A
server MAY omit it whether or not the request carried a `user`, and version 1.0
gives a client no way to require one. That permission is about a server with no
identity to send. One that resolved an identity it cannot carry within the
field's maximum does not omit it, and §3.1 says what it answers instead.

A client that omits `user` and cannot derive the identity from the credential
therefore depends on being deployed against a server that supplies `authn_id`.
That is an arrangement between a client and a deployment, not something the
protocol settles.

Within one `service`, a server SHOULD carry the same optional fields on every
AUTHENTICATE response it sends with the same `result`, whatever path it took to
that answer. Answering from a cache or from a result derived mechanically rather
than by consulting a live service is the mechanism to watch, and it is not the
only one. A deployment sees one exchange and generalises from it, so a field
that appears on the path a check took and not on the path it did not is what
leaves the arrangement above unverifiable.

The comparison is between answers alike, which is what makes it satisfiable. A
server that resolved no identity has none to send, so a denial carrying no
`authn_id` where an `OK` carries one is the field meaning what it says rather
than a server varying its answers.

This says nothing across `service` values. Two of them may route to backends
that genuinely differ, and a server that resolves an identity for one and not
the other is answering honestly. Nor does it reach §3's permission to carry
information in a field an older client understands: that varies which field
holds a value and not what the value says, and a client sits at one minor
version, so it meets the same substitution every time.

`authn_id` MUST NOT contain a control character, on the terms §5 sets for
`error_detail`.

#### 6.2.1 Auth result

`result` states what the server decided, or that the request's `service` named
no policy it holds. It appears only in an AUTHENTICATE response. A denial is not
a protocol failure: the server understood the request and answered it.

| code | name | meaning |
|---|---|---|
| 1 | `AUTHN_UNKNOWN` | a result this implementation does not recognise |
| 2 | `OK` | permitted |
| 3 | `AUTHN_DENIED_CREDENTIAL` | credential invalid, expired, or wrong audience |
| 4 | `AUTHN_DENIED_POLICY` | credential valid, but policy denies the identity, or denies it acting as `user` |
| 5 | `AUTHN_DENIED_USER_UNKNOWN` | the requested `user` does not resolve, or no `user` was sent and the credential resolved no identity |
| 6 | `AUTHN_DENIED` | the server declined, and does not say why |
| 7 | `SERVICE_UNSUPPORTED` | the server holds no policy by the name `service` gave, and reached no decision |

`AUTHN_DENIED` is always available and always sufficient. The codes that name a
reason are ones a server MAY use where it is willing to disclose that much. A
server that uses none of them is conforming, and so is one that uses some
(§6.2).

`SERVICE_UNSUPPORTED` is neither a denial nor a verdict, and the disclosure
choice does not reach it: a server that recognises no such `service` MUST answer
it. Nothing about the credential was examined and nothing about the remote party
is disclosed, since `service` comes from the client's own configuration and the
remote party does not influence it (§6.2). It sits here rather than among §5's
statuses because those describe how a server handled a message, and this one
handled it completely. What disagrees is the two deployments.

A client MUST NOT grant anything on `SERVICE_UNSUPPORTED` and MUST NOT report it
as an authentication outcome. It reports that the `service` it was configured to
send is not one the server holds.

Every code is available whether or not the request carried a `user`. With one,
the reasons are about the requested `user`: whether it exists as far as this
server is concerned, or whether the identity may act as it. Without one there is
no `user` to be unknown, so `AUTHN_DENIED_POLICY` is about the identity itself,
which a server MAY examine and apply policy to on its own account, and
`AUTHN_DENIED_USER_UNKNOWN` means the server resolved no identity at all.

A server that could not reach a decision because it failed, or a dependency did,
answers with an `ERROR` message carrying `UNAVAILABLE` or `INTERNAL_ERROR` (§5)
instead, and carries no `result` at all. `SERVICE_UNSUPPORTED` is the one answer
that reached no decision and is still an AUTHENTICATE response, because nothing
failed.

One code carries both decisions a client may need, because every code that is a
decision belongs to exactly one of them. `AUTHN_DENIED_POLICY` is an
authorisation outcome: its meaning is that the credential was valid and policy
denied it anyway. Every other non-`OK` code is a credential outcome,
`AUTHN_DENIED` included, because a client told nothing MUST NOT conclude that
the credential was good. `AUTHN_UNKNOWN` is not a server's decision at all but
what §3.4 has a receiver substitute for a value it does not recognise, and it
lands on the credential side for the same reason: a client that cannot read the
answer has not been told the credential was good. `SERVICE_UNSUPPORTED` is not a
decision either, and a client fails both steps on it.

Nothing here obliges a server to check a credential before it answers.
`AUTHN_DENIED_USER_UNKNOWN` in particular does not imply that the credential was
valid, and a server MAY answer it without having examined one.
`AUTHN_DENIED_POLICY` is the exception: a server MUST NOT answer it unless it
established that the credential was valid, since that is what the code states.
`AUTHN_DENIED_CREDENTIAL` carries the converse. A server MUST NOT answer it
where it established that the credential was valid, since a deployment reading
it reissues a credential that was never the fault.

It carries a second precondition for the same reason. A server MUST NOT answer
`AUTHN_DENIED_POLICY` where the requested `user` does not resolve, or where it
resolved no identity from the credential. The first is what
`AUTHN_DENIED_USER_UNKNOWN` names with a `user`, and the second is what it names
without one. With a `user` that resolves and no identity from the credential,
`AUTHN_DENIED_USER_UNKNOWN` is untrue, and `AUTHN_DENIED` is available as it
always is. A `user` resolves where the server holds the name, whatever its
policy says about who may act as it. So the line is whether the name resolves.
Whether any mapping reaches it, and the order a server did its work in, leave it
where it is. A name that resolves and is then refused is `AUTHN_DENIED_POLICY`,
and a name that does not resolve is `AUTHN_DENIED_USER_UNKNOWN` whether or not
the credential was examined first. `AUTHN_DENIED_USER_UNKNOWN` carries the
matching prohibition. A server MUST NOT answer it where the requested `user`
resolves, or, where no `user` was sent, where it resolved an identity from the
credential. §7 tests that, and Appendix B keys `PAM_USER_UNKNOWN` on it, which
is the code an operator configures a module to stand aside on.

None of these preconditions asks a server to disclose anything. Every code that
names a reason is optional and `AUTHN_DENIED` is always available, so this binds
only a server that has chosen to name one, and says which name is true.

A client that sent a `user` and must answer at two separate points derives both
from this one value. A credential outcome fails the credential step. An
authorisation outcome passes the credential step and fails the authorisation
step. A later version that adds a code MUST state which of the two decisions it
belongs to, or that it is not a decision, in which case both steps fail.

A client that sent no `user` gets the credential step from this value and owes
the authorisation step itself (§6.2).

A client SHOULD cache `result` rather than make a second request.

### 6.3 STATUS (`0x05` request, `0x06` response)

Reports health, for readiness probes and diagnostics.

**STATUS carries no credential.** The socket's own access controls are the only
gate on it.

Request:

| field | type | | |
|---|---|---|---|
| `max_age` | `integer`, 4 octets | optional | `0x0120` |
| `service` | `string` | optional | `0x0242` |

`max_age` bounds how stale the answer may be, in seconds. A server MUST NOT
answer from data older than `max_age`, so zero requires a check before
answering. If `max_age` is absent, the server MAY answer from data of any age.

Attempting a check satisfies `max_age` whatever its outcome. A server whose
dependency did not answer has just learned that it is not answering, and reports
that.

A server MAY rate-limit the checks `max_age` provokes. One that would have to
check, and whose limit forbids checking now, MUST respond `RATE_LIMITED` (§5).
It MUST NOT answer from data older than the client asked for.

`service` selects which policy to describe. If it is absent, the server answers
for itself as a whole rather than for any one policy. If it is present and the
server does not recognise it, the server MUST answer `SERVICE_UNSUPPORTED`.

Response:

| field | type | | |
|---|---|---|---|
| `serving` | `enum`, 1 octet | required | `0x0181` |
| `server_desc` | `utf8` | optional | `0x0243` |

`serving` is the readiness signal:

| code | name | meaning |
|---|---|---|
| 1 | `SERVING_UNKNOWN` | a value this implementation does not recognise |
| 2 | `SERVING` | the server can authenticate a *new* principal right now |
| 3 | `NOT_SERVING` | it cannot |
| 4 | `SERVICE_UNSUPPORTED` | the server holds no policy by the name `service` gave |

`SERVICE_UNSUPPORTED` is not a readiness signal, and it answers the selector
rather than the question. A probe MUST NOT read it as an outage: no wait
resolves it, because the name it sent is not one this server holds. It is the
same condition §6.2.1 gives the same name, answered in the enumeration each
response type carries.

A server that has lost a dependency but still holds the material it needs to
validate a new credential is serving. One validating against a previously
downloaded key set is in that position: the signature checks without reaching
the dependency, and revocation does not. A server MAY decline to authenticate
at all while it cannot check revocation, and reports `NOT_SERVING` where it
does. Declining to answer a STATUS request is not open to it: §2 obliges an
answer once it has read `ludsa_version_major`, and `NOT_SERVING` is that
answer.

A server that can answer only for principals it has already decided is not
serving, whatever it holds in cache. Answering a new principal is what `SERVING`
states.

The protocol defines no field that only some servers can populate. A server
with more to report than `serving` and `server_desc` carry exposes it by its own
means, alongside a client that understands it.

### 6.4 Tag assignments

Every tag in version 1.0, in the order §3.2 requires for emission. Tags sort by
their whole value and the field number is the high octet, so they sort by field
number and a field a later version adds sorts after every field already defined.
Within each group the required fields are numbered first. Each `msg_type` is a
group with its own schema.

| message type | tag | № | required | length | type | field |
|---|---|---|---|---|---|---|
| `ERROR` `0x00` | `0x0181` | 1 | **yes** | 1 octet | `enum` | `status` |
| `ERROR` `0x00` | `0x0243` | 2 | no | dynamic | `utf8` | `error_detail` |
| `PING_REQUEST` `0x01` | | | | | | *no fields* |
| `PING_RESPONSE` `0x02` | `0x0143` | 1 | no | dynamic | `utf8` | `server_desc` |
| `PING_RESPONSE` `0x02` | `0x0230` | 2 | no | 8 octets | `integer` | `uptime_seconds` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x01C2` | 1 | **yes** | dynamic | `string` | `service` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x02C4` | 2 | **yes** | dynamic | `bytes` | `credential` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x0342` | 3 | no | dynamic | `string` | `user` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x0442` | 4 | no | dynamic | `string` | `rhost` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x0542` | 5 | no | dynamic | `string` | `ruser` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x0642` | 6 | no | dynamic | `string` | `tty` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x0720` | 7 | no | 4 octets | `integer` | `pid` |
| `AUTHENTICATE_RESPONSE` `0x04` | `0x0181` | 1 | **yes** | 1 octet | `enum` | `result` |
| `AUTHENTICATE_RESPONSE` `0x04` | `0x0243` | 2 | no | dynamic | `utf8` | `authn_id` |
| `STATUS_REQUEST` `0x05` | `0x0120` | 1 | no | 4 octets | `integer` | `max_age` |
| `STATUS_REQUEST` `0x05` | `0x0242` | 2 | no | dynamic | `string` | `service` |
| `STATUS_RESPONSE` `0x06` | `0x0181` | 1 | **yes** | 1 octet | `enum` | `serving` |
| `STATUS_RESPONSE` `0x06` | `0x0243` | 2 | no | dynamic | `utf8` | `server_desc` |

Tags are unique within a group and not across the table. `0x0181` is `status`,
`result` and `serving`; `0x0243` is `error_detail`, `authn_id` and
`STATUS_RESPONSE`'s `server_desc`. `PING_RESPONSE` carries `server_desc` under
`0x0143` because it has no required field to number ahead of it, so the same
field has a different tag in the two response types that carry it.

A tag's width and type are fixed by the field it carries, and its required bit
by the field's role in that message type, so only the field number is free.
§3.3 matches a tag against the schema its `msg_type` selects, and a receiver
holds that schema before it reads a field: §6 binds a client to establish
`msg_type` first, and §4's order settles the message type at step 5, before any
tag is compared. A tag shared across schemas is therefore never offered to the
wrong one, and §4.1 leaves a later version free to assign a tag another message
type already uses.

What keeps a value read out of the wrong schema from reading as an approval is
§3.4 rather than the numbering, and only where the two enumerations differ over
2: it is unassigned in `status`, so a `status` read as a `result` is never `OK`.
Where both put an affirmative outcome at 2, as `result` and `serving` do,
nothing below §6's `msg_type` rule separates them.

Field number 0 is reserved (§3.3). Every other number here is either assigned
above or unassigned, and an assigned number stays assigned in every later minor
version (§4.1).

## 7. Conformance

A receiver MUST reject each of the following. A server rejects by sending the
status shown. A client has nothing to send, so it rejects by reporting to its
caller, and the condition binds it rather than the name: the names below are the
ones it SHOULD use, and a driver is told the ones it does use (F7). *Against*
names the side a case can be constructed for: every version 1 `string` field is
a request field, and a request has no `msg_type` a response permits, so some
cases exist in one direction only. Case 21 is marked `server` on other grounds.
It can be constructed against a client, but §5 makes a client's outcome there a
SHOULD, so the table of SHOULD-report cases below carries that direction.
*Needs* names a fixture this document does not define. F7 is a precondition of
every case that reads an outcome from the client under test, rather than of any
one case, so the column does not repeat it. A case whose assertions are all on
the wire does not need it. F11 sits there for the same reason: §6 obliges a
server to implement no particular request type, so every case that drives one is
a case the peer may not answer, and the column does not repeat that either.

Every case driven against a server is answerable once the connection is one the
server will serve, because each carries at least a version word and a server can
answer in its own major version (§2). A server making the check §5 permits
refuses before that, and refuses every case here alike, so F12 settles whether
it accepts the uid the driver connects from. A driver that has not established
this cannot tell a refusal from a fault (§2). A case driven against a client has
nothing to answer, so it reports to its caller instead. Where a case is marked
`either`, the status shown is the server's and the client's counterpart is the
name paired with the same condition below.

| # | case | against | status | needs |
|---|---|---|---|---|
| 1 | A `ludsa_version_major` it does not implement | either | `VERSION_UNSUPPORTED` | |
| 2 | A declared length above `MAX_LENGTH` | either | `REQUEST_TOO_LARGE` | |
| 3 | A declared length below the minimum for the version | either | `REQUEST_MALFORMED` | |
| 4 | A field length extending beyond the declared message length | either | `REQUEST_MALFORMED` | |
| 5 | A **recognised** tag appearing twice in a message type that permits it only once | either | `REQUEST_MALFORMED` | |
| 6 | A tag lower than the tag before it | either | `REQUEST_MALFORMED` | |
| 7 | An unrecognised **required** tag | either | `REQUEST_MALFORMED` | |
| 8 | A message not terminated by `TAG_END` | either | `REQUEST_MALFORMED` | |
| 9 | Trailing octets after `TAG_END` within the declared length | either | `REQUEST_MALFORMED` | |
| 10 | A tag with bit 6 set and either of bits 5-4 set | either | `REQUEST_MALFORMED` | |
| 11 | A tag whose assigned type disagrees with bit 6 | either | `REQUEST_MALFORMED` | |
| 12 | A tag whose field number is 0 | either | `REQUEST_MALFORMED` | |
| 13 | A NUL octet in a `string` field | server | `REQUEST_MALFORMED` | |
| 14 | A missing required field for the message type | either | `REQUEST_MALFORMED` | |
| 15 | The message ending before it is complete, after `ludsa_version_major` has been read | either | `REQUEST_INCOMPLETE` | |
| 16 | A `msg_type` it does not accept, including a response type or `ERROR` | server | `REQUEST_UNSUPPORTED` | |
| 18 | A STATUS `max_age` it cannot satisfy under its own limit | server | `RATE_LIMITED` | F2 |
| 19 | The request deadline expiring after `ludsa_version_major` has been read and before the declared octets have all arrived, in octets carrying no fault the receiver could have reported | server | `REQUEST_TIMEOUT` | F6 |
| 20 | A response carrying a `msg_type` the client implements, which the request does not permit (§6) | client | `RESPONSE_MALFORMED` | |
| 21 | A field whose value exceeds its §3.1 maximum, in an otherwise well-framed message | server | `REQUEST_MALFORMED` | |
| 22 | An unimplemented `msg_type` in a message that also has a repeated tag | either | `REQUEST_UNSUPPORTED` | |
| 23 | An unrecognised `service` in a message that also omits `credential` | server | `REQUEST_MALFORMED` | F1 |
| 24 | A response of exactly one octet, then end of file | client | `RESPONSE_INCOMPLETE` | |
| 25 | A response carrying a `msg_type` the client does not implement | client | `RESPONSE_UNSUPPORTED` | |
| 27 | A declared length above `MAX_LENGTH`, in a message whose octets also carry a NUL | either | `REQUEST_TOO_LARGE` | |
| 28 | The connection accepted and the request read, then closed with no response octet | client | `RESPONSE_ABSENT` | |
| 29 | The connection accepted and the request read, then held open past the client's deadline | client | `RESPONSE_TIMEOUT` | F6 |

Cases 17, 26 and 30 are retired. 17 drove an unrecognised `service`, which
§6.2.1 and §6.3 now answer with `SERVICE_UNSUPPORTED` rather than a rejection.
26 drove a shape-bad tag behind an unimplemented `msg_type`, and 30 a truncated
message whose arrived octets carried a fault. §4 leaves both to the receiver. No
retired number is reused, on the ground §4.1 gives for never reassigning a field
number: a citation that outlives what it named should fail to resolve rather
than resolve to something else.

Every version 1.0 `utf8` field is one a client may reject for its content. §5
makes a defect in `error_detail`, `server_desc` or `authn_id` a fault in the
server that produced it, and the cases below drive each defect against a
client.

A request that ends, or stalls, before `ludsa_version_major` is complete may
draw no response at all, or a response in the server's own major version (§2).
Both conform, so it is not a case here and a driver MUST NOT record either as a
failure. Case 24 is its counterpart on the other side, where one octet is a
report rather than a silence.

A message both truncated and faulty in the octets that arrived may draw either
status (§4). So may one whose framing fault lies behind a `msg_type` the
receiver does not implement, since a receiver that answered on the type it read
never reached the fault. Both conform in each, so neither is a case here and a
driver MUST NOT record any of those outcomes as a failure.

A message declaring a minor version the receiver implements, carrying a tag that
version does not define at a field number above that message type's ceiling
(§3.3), may draw `REQUEST_MALFORMED` or be accepted with the tag skipped (§4).
Both conform, so it is not a case here either. At or below the ceiling only
rejection conforms, and the rule below drives it.

No case drives `UNAVAILABLE`. It reports a dependency the driver does not
control, and a driver records one wherever a run produces it. No case drives
`STATUS_UNKNOWN` either, because it is what a receiver substitutes for a status
it does not recognise, and the acceptance rule below drives that substitution
for every enumeration at once.

Some of these depend on how the peer under test is configured, which this
document does not specify. A driver MUST be given all of them:

- **F1**: a `service` value the peer recognises, and one it does not.
- **F2**: whether the peer implements the rate limiting §6.3 permits. Case 18
  applies only where it does, and a peer without a limit is not non-conforming
  for never exhibiting it.
- **F3**: a credential the peer accepts, and a `user` the server holds that its
  policy does not map that credential's identity to. The policy must also refuse
  that identity a `user` of zero octets, which a deployment whose policy grants
  any `user` cannot supply. Such a peer cannot run the second decision case and
  is not non-conforming for that. It also says whether appending a trailing
  newline to that credential makes it invalid to the peer's backend, which is
  what the credential case below appends. A backend that accepts the modified
  value cannot run that case and is not non-conforming for that either. It also
  says whether the server is configured to supply `authn_id`. Where it is and
  the backend can supply one, it also gives a credential whose resolved identity
  is longer than `authn_id`'s §3.1 maximum, and which the server would answer
  `OK` to with no `user` sent. A server not configured to supply `authn_id`, or
  whose backend cannot supply such an identity, cannot run the over-long
  identity case and is not non-conforming for that.
- **F4**: the bound the peer sets on the drain (§3), where it sets one. §2's
  request deadline bounds the drain otherwise, and F6 supplies that. A server
  whose bound is shorter than the driver's round trip has stopped draining
  legitimately and cannot be told from one that never drained.
- **F5**: the server identity the client under test is configured to expect, and
  a driver socket bound and listened on by a process running as a different uid,
  at a path that uid owns. §2 names no mechanism, so the driver supplies what a
  peer-credential check and an ownership check each read. A client configured
  with none owes nothing under §2 and cannot run the case below.
- **F6**: the deadlines the peer applies (§2). A driver needs the server's
  accept-to-complete-request deadline to know how long to stall, and the
  client's overall deadline to know how long to hold a connection open and how
  long to wait. Neither has a value this document sets.
- **F7**: the names the client under test reports, mapped onto the client-local
  names below, and the channel a driver reads them on. Every case that reads an
  outcome from the client under test needs it, including one marked `either`. A
  case whose assertions are all on the wire does not. None of those names
  crosses the socket and a client using its own is conforming, so a driver can
  infer neither the names nor the channel from anything the wire carries.
- **F8**: a route by which a driver hands the client under test a value of its
  choosing, through the client's own host interface, and the fields that route
  reaches. For a PAM client that is a conversation supplying a token. A client
  whose host cannot present the value a case names cannot run that case and is
  not non-conforming for that.
- **F9**: whether the driver can set the working directory of the client under
  test's host process. The relative-path case needs the driver's socket to sit
  where a client that resolved its configured path would land, which the driver
  arranges by choosing that directory. A client whose host cannot have one set
  cannot run that case and is not non-conforming for that.
- **F10**: whether the client under test can be configured to write its request
  in bounded pieces, and to pause between them. This is a test affordance and
  not anything the protocol asks of a client at runtime, where one write is the
  better behaviour. It is what lets a driver close a connection while a write is
  genuinely outstanding, since no request this document permits is large enough
  to arrange that on its own. A client that cannot be configured that way cannot
  run the case below in its reliable form and is not non-conforming for that.
- **F11**: which message types the peer under test implements, and for a client
  which response fields it consumes. §6 obliges a server to implement no
  particular request type, and §5 relieves a client of checking a value it does
  not use, so neither is derivable from the wire. For a client this also settles
  which response types a driver can put in front of it: one that sends no
  request whose response carries `server_desc` cannot be driven that field at
  all, and §6.4 assigns it in `PING_RESPONSE` and in `STATUS_RESPONSE`.
- **F12**: whether the peer under test makes the peer-credentials check §5
  permits, which of §5's answers it gives where it does, whether it accepts the
  uid the driver connects from for every other case in this section, and a
  second uid it refuses, which the case below connects from. §5 leaves the check
  to a deployment and the answer to the server, so none of them is derivable
  from the wire. F5's case asks a driver to run as a second uid, so the
  affordance is one a driver has already. F5 settles that uid against what the
  client expects, and this fixture settles its own against what the server
  refuses.

One further case is a decision rather than a rejection, and a driver exercises
it too. It is the only test of §6.2's binding rule, which no wire-level input
can reach:

| case | against | expected | needs |
|---|---|---|---|
| A credential valid for one identity, sent with a `user` the server holds that identity may not act as, under a `service` the server recognises | server | an `AUTHENTICATE_RESPONSE` whose `result` is one of §6.2.1's `AUTHN_DENIED` codes, and is not `AUTHN_DENIED_USER_UNKNOWN` | F1, F3 |
| A credential the peer accepts, sent with a `user` of zero octets, under a `service` the server recognises | server | an `AUTHENTICATE_RESPONSE` whose `result` is one of §6.2.1's `AUTHN_DENIED` codes | F1, F3 |

Requiring an `AUTHENTICATE_RESPONSE`, and requiring a denial within it, each
carry weight. An `AUTHENTICATE_REQUEST` requires a `service`, so a driver must
choose one, and a `service` the server does not hold obliges it to answer
`SERVICE_UNSUPPORTED` (§6.2.1), which is not `OK` and is not a decision either.
A peer holding no services would otherwise pass the only test of §6.2's binding
rule without examining a credential.

The first case excludes one code. F3 supplies a `user` the server holds and a
credential it accepts, so the name resolves and an identity resolved, which is
what §6.2.1 makes `AUTHN_DENIED_USER_UNKNOWN` untrue of. The case cannot require
`AUTHN_DENIED_POLICY` in its place, since §6.2 leaves a server free to name no
reason at all. The second case excludes nothing, because whether a `user` of
zero octets resolves is a deployment's own fact and F3 settles only that the
policy refuses it.

Further cases exercise the selector, which is neither a rejection nor a
decision. Each drives the same condition into the enumeration its response type
carries:

| case | against | expected | needs |
|---|---|---|---|
| An AUTHENTICATE request naming a `service` the server does not recognise | server | an `AUTHENTICATE_RESPONSE` whose `result` is `SERVICE_UNSUPPORTED` | F1 |
| A STATUS request naming a `service` the server does not recognise | server | a `STATUS_RESPONSE` whose `serving` is `SERVICE_UNSUPPORTED` | F1 |
| A STATUS request naming a `service` the server does not recognise, whose `max_age` its own limit also forbids | server | a `STATUS_RESPONSE` whose `serving` is `SERVICE_UNSUPPORTED` | F1, F2 |

The third pins §4's step 11 before step 12. §4 says the order between steps 8
and 9 is not observable because both yield the same status; this pair is not
like that, since the selector answers in a `STATUS_RESPONSE` and the limit
answers `RATE_LIMITED` in an `ERROR`, so a receiver taking them the other way
round is visible on the wire.

One further case observes what a server does after it has answered, which no
status can show. The driver writes an 8-octet envelope declaring a
`ludsa_version_major` the server does not implement, reads the response, and
then writes further octets without half-closing:

| case | against | expected | needs |
|---|---|---|---|
| Further octets written straight after a `VERSION_UNSUPPORTED` response, on a connection the driver has not half-closed | server | the write is accepted, end of file has already arrived on the driver's read direction, and the driver terminates normally | F4, F6 |

§3 has the server draining at that point, so it consumes the write. The two
assertions together are what test §2's ordering: end of file proves the server
shut down its write direction, and the accepted write proves it was still
draining when it did. A server that deferred the half-close until the drain
ended fails on the first, and one that closed fails on the second. A server
that closed instead gives the driver `EPIPE`, or `SIGPIPE` where it has not
suppressed one. A driver may suppress the signal or may treat termination by it
as the failing outcome, and this document does not choose.

A server that shuts down its write direction and never reads also accepts the
write, so this case detects a server that closed rather than one that did not
drain. Whether a drain ran is not observable from outside, which is what F4
concedes for the bound.

The remaining bounds are ruled out by construction. §3 puts the declared length
out of play for `VERSION_UNSUPPORTED`, so the permission to stop early is not
available either; the driver does not half-close, so there is no end of file;
and an envelope plus a kilobyte is far below `2 x MAX_LENGTH`.

A driver checks the encoding of every response it receives on §4's terms: no tag
lower than the tag before it, and each recognised tag appearing at most once.
Equal tags are not a breach, since §4.1 lets a later minor version assign a
repeatable field. Where a response declares a `ludsa_version_minor` the driver
implements, it also records a tag that version does not define for that message
type, which is the sender breaking §3.2. It records rather than rejects: whether
to reject is a receiver's own choice (§4), and against a higher minor version
the check cannot be made at all, since the driver does not hold the schema that
would settle it.

It checks their content on the same terms, over every field it recognises: no
enumerated value of 0 or 1 (§3.4), no `utf8` value carrying a control character
or an invalid encoding (§5), and no value above its §3.1 maximum. Each of those
binds a sender and no input provokes a breach, so a driver records one wherever
a run happens to produce it rather than in a case of its own. That is separate
from what it does as a receiver, where §5 has it report the response as
malformed.

One further case pins what no other input reaches, since a server's own version
word is not something a request can provoke:

| case | against | expected | needs |
|---|---|---|---|
| Any request carrying a `ludsa_version_minor` other than the server's own, which every response envelope reports (§3) | server | a response carrying the server's own `ludsa_version_minor`, not the one sent | |

§3.2's rule against emitting an optional field that has no value is not
driveable. A zero-length value is a value (§3.2), so nothing distinguishes the
two from outside.

One further case observes what a client does before it writes anything, which
§2's conditional obligation reaches and no message can. The driver accepts the
connection and writes nothing:

| case | against | expected | needs |
|---|---|---|---|
| A connection accepted by a peer whose identity is not the one the client was configured to expect | client | the client writes no octet and reports `CONNECTION_FAILED` | F5 |

This case turns on which uid is at the far end of the socket, which is why a
driver needs a second uid for it. The server-side counterpart turns on the same
uid, from the other direction. The driver connects from one F12 says the server
refuses, and writes a complete request:

| case | against | expected | needs |
|---|---|---|---|
| A complete `AUTHENTICATE_REQUEST` on a connection from a uid the server refuses | server | the server serves no request: it closes without replying, or answers `PEERCRED_DENIED` on §5's terms, and no `AUTHENTICATE_RESPONSE` arrives | F1, F3, F12 |

§5 leaves a refusing server its choice of answer, so what the case asserts is
what every shape of that choice shares: nothing the request asked for. A driver
that cannot connect from a refused uid cannot run it, and is not non-conforming
for that.

One further case observes a client that never connects at all, which §3.1's and
§3.3's prohibitions reach and no message can. The driver listens, and hands the
client a value one of them forbids:

| case | against | expected | needs |
|---|---|---|---|
| A value above the §3.1 maximum of a field F8 reaches, handed to the client through its host interface | client | the client reports `INVALID_REQUEST`, and the driver's socket saw no connection | F8 |
| A NUL octet in a value bound for a `string` field F8 reaches, handed to the client through its host interface | client | the client reports `INVALID_REQUEST`, and the driver's socket saw no connection | F8 |

The report is what ends the case, so no deadline is needed: a client that
connected before reporting has already failed it. §3.1 and §3.3 each state the
other half of that sentence in the indicative rather than as a MUST, so the name
is a client-local one under this section's usual terms and the prohibition on
connecting is what binds. A client that omitted the field and sent the rest has
failed it too, since the driver's socket then saw a connection.

One further case observes a client configured with a path it cannot use, which
§2's prohibition reaches and no message can. The driver binds a socket at each
length a truncating client could land on, which its platform gives as the size
of the address structure's path array and one less, and configures the client
with a longer path sharing that prefix. A length the platform refuses to bind is
one a truncating client cannot connect at either, so the assertion holds there
without a socket. It reads those lengths from the platform rather than from this
document:

| case | against | expected | needs |
|---|---|---|---|
| A configured socket path too long for the platform's socket address | client | the client reports `CONNECTION_FAILED`, and no socket the driver bound saw a connection | |

No fixture beyond F7 is needed, since every case driven against a client already
configures it with a socket path. The driver's own paths are not free, though. A
client that truncates lands on one of the sockets it bound, so the assertion
that no connection arrived is what fails such a client. A driver that bound
anywhere else could not tell one from a conforming client, since a truncated
path that names nothing draws the same `CONNECTION_FAILED` as a path never
used.

One further case observes a client configured with a path §2 forbids for a
different reason. The driver binds its socket in a directory of its choosing,
sets the client's working directory to that directory (F9), and configures the
client with the socket's basename alone:

| case | against | expected | needs |
|---|---|---|---|
| A configured socket path that is not absolute | client | the client reports `CONNECTION_FAILED`, and the driver's socket saw no connection | F9 |

The second assertion is the one that separates the answers, and the driver's
choice of directory is what gives it force. A client that resolves the path
finds the driver's socket at the far end of it and connects. A driver that let
the resolution land anywhere else would see no connection either way, and would
pass a client that resolves alongside one that refuses.

One further case observes the octets a client puts on the wire, which §3.2
binds. The driver hands the client a `user` carrying leading and trailing
whitespace, and reads the request:

| case | against | expected | needs |
|---|---|---|---|
| A `user` carrying leading and trailing whitespace, handed to the client through its host interface | client | the `user` field on the wire carries those octets unchanged | F8 |

A client that trims, folds case or re-encodes fails it. §3.2's rule names no
field; `user` is what the case drives, because §6.2 has a server's decision turn
on it.

One further case observes the octets a server acts on, which §6.2 binds it to as
well: the rule there reaches the server and not only the sender. The driver
takes a credential the peer accepts, appends a newline, and sends it under a
`service` the server recognises:

| case | against | expected | needs |
|---|---|---|---|
| A credential the peer accepts, with a trailing newline appended, under a `service` the server recognises | server | an `AUTHENTICATE_RESPONSE` whose `result` is one of §6.2.1's `AUTHN_DENIED` codes | F1, F3 |

A server that trims the credential before validating it answers `OK` and fails.
Which denial it names is its own choice (§6.2): one that discloses no reason
answers `AUTHN_DENIED`, and one that does answers `AUTHN_DENIED_CREDENTIAL`,
since the value it was handed did not validate. Naming the codes by their prefix
rather than requiring one of them is what leaves that choice open.

No wire-level input reaches this rule. The octets are well formed either way,
and what differs is what the server did with them before it decided, which is
why F3 has to say whether the appended octet makes the credential invalid to the
peer's backend.

One further case observes a server holding a value it cannot carry, which §3.1
binds and no wire-level input reaches. The driver sends a credential whose
resolved identity is longer than `authn_id`'s maximum, with no `user`, under a
`service` the server recognises, to a server configured to supply `authn_id`:

| case | against | expected | needs |
|---|---|---|---|
| A credential whose resolved identity is longer than `authn_id`'s §3.1 maximum, sent with no `user`, under a `service` the server recognises, to a server configured to supply `authn_id` | server | an `ERROR` whose `status` is `INTERNAL_ERROR` | F1, F3 |

A server that truncates the identity answers an `AUTHENTICATE_RESPONSE` carrying
an `authn_id` at exactly its maximum, and fails. One that omits the field
answers a response carrying none, and fails too, since §3.1 keeps omission for a
server with no value to send. The identity is the value, so unlike
`error_detail` there is nothing to embed less of, and the request was well
formed throughout: what the case observes is what the server did with a value of
its own. The request carries no `user`, and F3 supplies a credential the server
would answer `OK` to without one, so the only thing between the server and an
`OK` carrying the field is the identity's length.

One further case observes a client whose peer closes while it is still writing,
which §2.1 requires it to survive and §2 permits the server to do. Where F10
lets it, the driver accepts, reads part of the request, and closes:

| case | against | expected | needs |
|---|---|---|---|
| The connection closed while the client under test is still writing its request | client | the client reports `RESPONSE_ABSENT` | F8, F10 |

What the case detects is a client whose host application did not survive the
write failing, since that client reports nothing at all.

Request size cannot arrange the condition. Appendix A's largest request fits
inside a default socket send buffer on common platforms, so a client that issues
it in one call has returned from the write before the driver can read anything,
and `MAX_LENGTH` forbids a larger one. F10 is what makes the case reliable: a
client that writes in bounded pieces, pausing between them, is still writing
when the driver closes. F8's near-maximum `credential` widens that window rather
than creating it.

A client that offers no such configuration is driven by a driver that accepts
and closes at once, reading nothing. The case is then best-effort, because
whether the write fails is not the driver's to arrange: a client that completed
its write in one call is not exercised by the case and cannot be told from one
that survived.

One further case observes a client that holds a complete response and no end of
file. The driver accepts a connection, reads the request, writes a well-formed
`AUTHENTICATE_RESPONSE`, and then holds the connection open past the client's
own deadline without half-closing:

| case | against | expected | needs |
|---|---|---|---|
| A complete response, then the connection held open past the client's deadline | client | the client reports that response's `result`, before its deadline | F6 |

§2 delivers a message at its declared octets, so the client holds the response
and the end is nothing it waits for. A client reporting `RESPONSE_TIMEOUT` there
has let its deadline take away a response it held. One that reports the `result`
only once an end arrives has waited for something §2 does not make part of the
message, and holding the connection open past the deadline is what tells either
of them from a client that reports at once.

One further case observes what a client does after it writes, which §2 requires
of every client and no status reports. The driver accepts a connection, reads
the request, and answers it:

| case | against | expected | needs |
|---|---|---|---|
| A request written in full by the client under test | client | the driver reads end of file, and its response is then accepted for writing | F6 |

The two assertions are the two halves of §2's rule, and one of them fails
without the other. A client that shuts down its write direction gives the driver
end of file and stays readable. One that closes the connection outright gives
the driver end of file too, and the write that follows draws `EPIPE`, or
`SIGPIPE` where the driver has not suppressed one. The terms are §3's drain
case's: a driver may suppress the signal or may treat termination by it as the
failing outcome, and this document does not choose. F6 bounds the driver's wait
for end of file, since a client that never half-closes holds it until its own
deadline ends the exchange.

Nothing here obliges a server to wait for end of file. §2 delivers the request
at its declared octets, and its check on octets beyond the declared length is a
SHOULD a server need not make exhaustive.

It SHOULD reject the following. A conforming receiver may accept or reject it,
so it carries no expected outcome. The *against* and status columns mean what
they mean in the numbered table, so a client detects the same condition and
reports the name paired with it below:

| case | against | status |
|---|---|---|
| Octets beyond the declared length, where observed | either | `REQUEST_MALFORMED` |

A defect in `error_detail`, `server_desc` or `authn_id` is a server bug (§5),
and a client SHOULD report it rather than work around it. Tolerance is
unspecified, so these carry no MUST on the outcome. What they do carry is the
prohibition: a client MUST NOT report a `REQUEST_` code for any of them.

| case | against | SHOULD report |
|---|---|---|
| A response `utf8` field that is not valid UTF-8 | client | `RESPONSE_MALFORMED` |
| A response `utf8` field carrying a control character | client | `RESPONSE_MALFORMED` |
| A response `utf8` field carrying a NUL octet | client | `RESPONSE_MALFORMED` |
| A response `utf8` field above its §3.1 maximum | client | `RESPONSE_MALFORMED` |

These name no field, and a driver chooses one F11 says the client consumes, in a
response type F11 says it can receive. §5 relieves a client of
checking a value it does not use, so a driver that picks otherwise records a
SHOULD the client never owed.

A receiver MUST accept a message whose `ludsa_version_minor` differs from its
own, in either direction, and MUST skip an unrecognised **optional** tag without
error, including one that appears more than once. The skipping obligation
reaches a field number above that message type's ceiling (§3.3). At or below the
ceiling §4 has it reject, whatever minor version the message declares. A driver
drives the skip with a message declaring a minor version the receiver does not
implement, since §4 leaves rejection open where it declares one the receiver
does, and a receiver that rejects there conforms.

A receiver MUST accept a message whose declared length is exactly the minimum
for the version, which a `PING_REQUEST` carrying no fields is, and a field whose
value is exactly its §3.1 maximum. What the driver asserts is that the length
was not what the receiver rejected: a server answering `REQUEST_UNSUPPORTED` has
accepted the length and failed at §4's step 5 instead, and §6 obliges it to
implement no particular request type. So the driver drives a type F11 says the
peer implements. A peer implementing none that can carry a minimum-length
message cannot be driven for this rule and is not non-conforming for that, since
`AUTHENTICATE_REQUEST` has required fields and cannot be one. It
MUST report an enumerated value it does not recognise as that enumeration's 1
(§3.4) and MUST NOT report it as an affirmative outcome, so a `result` of 200 is
`AUTHN_UNKNOWN` and never `OK`, and so is a `result` of 0.

A receiver MUST reject a tag whose whole value is not the assignment its field
number carries. That reaches a number its version assigns, and a number left
unassigned below the ceiling (§3.3). §3.3 matches a tag whole and never by field
number, and §4.1 is what makes the rejection safe: no version can give such a
tag meaning. `0x0242` placed before `0x02C4` in an `AUTHENTICATE_REQUEST` drives
this against a server, and `0x0142` before `0x0181` in an
`AUTHENTICATE_RESPONSE` against a client. A receiver fails the case if it reads
the tag's value into the defined field, and fails it if it skips the tag and
parses the rest.

**A client reports a fault to its caller.** It has no status to send: a fault it
found in a response is not a statement about the request it sent, and a request
it never sent is not a statement about anything. These are the names it SHOULD
use. None of them appears on the wire, and a client that uses its own names is
still conforming:

| name | the client found |
|---|---|
| `RESPONSE_MALFORMED` | the response was structurally wrong, or a field in it was defective or above its §3.1 maximum |
| `RESPONSE_TOO_LARGE` | the response declared a length above `MAX_LENGTH` |
| `RESPONSE_UNSUPPORTED` | the response carried a `msg_type` the client does not implement |
| `RESPONSE_INCOMPLETE` | the server's side ended before the declared octets had arrived, or before it declared a length at all, by end of file or an error that ends the read. An interrupted or would-block read is neither |
| `RESPONSE_ABSENT` | the server's side ended with no response octet sent |
| `RESPONSE_TIMEOUT` | the client's deadline expired before the declared octets had arrived |
| `CONNECTION_FAILED` | the client never established a connection it was willing to use |
| `INVALID_REQUEST` | the caller's own input could not be carried in a conforming request |

A client whose host wants a distinct code per condition MAY subdivide these,
`INVALID_REQUEST_CREDENTIAL_TOO_LARGE` and so on. What it MUST NOT do is report
a code a server sends for a fault it found itself. §5's statuses describe a
request a server received, and a request that was never sent gives them nothing
to describe.

The table above binds a client by its conditions, not by these names. Where it
gives a status for a case marked `either`, a client detects the same condition
and reports the response name beside it here, except where the status carries no
direction and serves both sides. Where a case is marked `client`, the status
column already holds one of these names, and the same latitude applies to it.

A client reports `VERSION_UNSUPPORTED` for a response major version it does not
implement, and relays every status it read: under the name the server sent it,
or as `STATUS_UNKNOWN` (§3.4, §5) where it does not recognise the value. §5's
prohibition reaches a code a client would report for a fault it found itself,
and a status a server emitted is that server's statement about a request the
client did send.

How much had arrived when the wait ended decides which of the three a client
reports. Where the server's side ended it, a client that read no octet reports
`RESPONSE_ABSENT` and one that read any reports `RESPONSE_INCOMPLETE`. Where its
own deadline ended it first, the client reports `RESPONSE_TIMEOUT` whatever had
arrived. None of the three is reached once the declared octets are in hand: a
client holding them holds the response (§2) and reports what it says, however
the connection ends afterwards. It MUST NOT report any of the three as a
decision.

That is not the line §2 draws, and case 15's qualification binds a server only.
A server needs `ludsa_version_major` entire before it can answer at all, so
below two octets it says nothing and cannot tell the two cases apart. A client
answers nothing, so one octet is enough to separate a server that spoke from one
that never did. Case 24 is the case that pins it.

`ludsa-conformance` sends each of these. Test vectors in `testdata/` MUST be
hand-authored from this document. They MUST NOT be generated by dumping an
implementation's output.

## 8. Non-goals

- Confidentiality or integrity on the wire.
- A mechanism for authenticating the server to the client. §2 requires a client
  configured with an expected identity to confirm it before writing, and leaves
  the confirmation itself to the deployment.
- Streaming, pipelining, or multiplexing.
- Carrying a PAM conversation. LUDSA cannot drive prompts, so it does not
  support interactive flows.
- A machine-readable companion to `error_detail`. Version 1.0 defines none, and
  a client acts on `status` alone.

## Appendix A: derived message maxima

Derived, not prescriptive. Every figure here follows from §3.1's field maxima
and §6.4's tag assignments, and those two are the authority. Where this table
disagrees with them, they are right and this table is stale.

A declared length is `msg_type` at 4 octets, plus each field, plus `TAG_END` at
2. A dynamic field costs its tag at 2, a `u32 len` at 4, and its value. A fixed
field costs its tag at 2 and the width the tag states. The envelope's 8 octets
lie outside the declared length (§3).

| message | max declared length |
|---|---|
| `PING_REQUEST` | 6 |
| `STATUS_REQUEST` | 82 |
| `ERROR` | 527 |
| `AUTHENTICATE_RESPONSE` | 527 |
| `STATUS_RESPONSE` | 527 |
| `PING_RESPONSE` | 534 |
| `AUTHENTICATE_REQUEST` | 17520 |

The largest version 1.0 request is 17520 and the largest response is 534,
against a `MAX_LENGTH` of 32768.

**These are what a version 1.0 peer emits, not what a receiver may insist on.**
A later minor version may add optional fields, so every figure here can grow
within major version 1. §3 still requires accepting any declared length up to
`MAX_LENGTH`, and §3.3 still requires skipping an unrecognised optional field. A
receiver that rejects a message for exceeding a figure in this table is not
conforming.

## Appendix B: mapping to PAM

Non-normative, except where it says MUST or SHOULD. Those bind a client that
adopts this mapping. It applies only where a client's host application is
PAM. A client that maps differently is still conforming, and `pg_ludsa` has no
PAM at all.

It is here rather than left to each client because two PAM clients that map the
same outcome differently give an operator a stack that behaves differently
against each, and no case in §7 can detect that. The wire is identical either
way.

The split is between something needing a person and something needing a retry.
An operator reading `PAM_SYSTEM_ERR` goes to a bug report or a configuration
file; one reading `PAM_AUTHINFO_UNAVAIL` waits, or falls through to another
module.

A status reaches a client only in an `ERROR` message (§5). A response carrying a
message of the request's type is the case where the server answered, and a
client reads the result out of it.

| status | PAM code |
|---|---|
| `STATUS_UNKNOWN` | `PAM_AUTHINFO_UNAVAIL` |
| `REQUEST_MALFORMED` | `PAM_SYSTEM_ERR` |
| `REQUEST_INCOMPLETE` | `PAM_AUTHINFO_UNAVAIL` |
| `REQUEST_TOO_LARGE` | `PAM_SYSTEM_ERR` |
| `REQUEST_UNSUPPORTED` | `PAM_SYSTEM_ERR` |
| `VERSION_UNSUPPORTED` | `PAM_SYSTEM_ERR` |
| `REQUEST_TIMEOUT` | `PAM_AUTHINFO_UNAVAIL` |
| `UNAVAILABLE` | `PAM_AUTHINFO_UNAVAIL` |
| `INTERNAL_ERROR` | `PAM_SYSTEM_ERR` |
| `RATE_LIMITED` | `PAM_AUTHINFO_UNAVAIL` |
| `PEERCRED_DENIED` | `PAM_AUTHINFO_UNAVAIL` |

These are defaults, not requirements. A client MAY let a deployment override the
code it returns for an outcome, and one that does SHOULD keep these as its
defaults, so two clients behave alike where nobody has configured otherwise. A
client MUST NOT let an override admit a caller on a `result` other than `OK`, or
turn any failure into an admission. It MUST NOT let one return `PAM_SUCCESS` or
`PAM_IGNORE` from an entry point where the result table below does not, so the
deferral this appendix makes for `AUTHN_DENIED_POLICY` stays the appendix's own.
That cell admits nobody by itself, because the same outcome denies at
`pam_sm_acct_mgmt` and the rule below binds a client that relies on it, and it
is the one place where the test of what a stack does rather than what an entry
point returns has to be applied.

`PEERCRED_DENIED`, and a `CONNECTION_FAILED` a client reached by refusing the
server's identity or by being refused the socket, describe one condition from
opposite ends: a client and a server that a configuration does not permit to
talk. Each is permanent until somebody changes a configuration, which is what
separates them from the retryable conditions the same name also covers. A client
that offers an override SHOULD apply one setting across them, so an
administrator who changes how a peer refusal behaves changes every view of it a
client can name. A server that refuses in silence (§5) leaves the client
`RESPONSE_ABSENT`, which §2 makes indistinguishable from a server that failed,
so no override reaches that shape and the default is what it gets.
`CONNECTION_FAILED` also covers conditions that clear on a retry, such as a
socket that is not there yet, and an override keyed on a peer refusal does not
reach those.

The response codes §7 gives a client:

| response code | PAM code |
|---|---|
| `RESPONSE_MALFORMED` | `PAM_SYSTEM_ERR` |
| `RESPONSE_TOO_LARGE` | `PAM_SYSTEM_ERR` |
| `RESPONSE_UNSUPPORTED` | `PAM_SYSTEM_ERR` |
| `RESPONSE_INCOMPLETE` | `PAM_AUTHINFO_UNAVAIL` |
| `RESPONSE_ABSENT` | `PAM_AUTHINFO_UNAVAIL` |
| `RESPONSE_TIMEOUT` | `PAM_AUTHINFO_UNAVAIL` |
| `CONNECTION_FAILED` | `PAM_AUTHINFO_UNAVAIL` |
| `INVALID_REQUEST` | `PAM_SYSTEM_ERR` |

`RESPONSE_MALFORMED` is evidence of a bug in the server (§5), so it needs a
person. `RESPONSE_ABSENT` means the server accepted the connection and then said
nothing, which is the shape of an unreachable directory, and
`PAM_AUTHINFO_UNAVAIL` is the code an operator already keys a fall-through on
for that. A stack that tolerates an unreachable directory then tolerates an
unreachable LUDSA server without new configuration.

Auth result (§6.2.1):

| result | from `pam_sm_authenticate` | from `pam_sm_acct_mgmt` |
|---|---|---|
| `OK` | `PAM_SUCCESS` | `PAM_SUCCESS` |
| `AUTHN_UNKNOWN` | `PAM_AUTH_ERR` | `PAM_AUTH_ERR` |
| `AUTHN_DENIED_CREDENTIAL` | `PAM_AUTH_ERR` | `PAM_AUTH_ERR` |
| `AUTHN_DENIED_POLICY` | **`PAM_SUCCESS`** | `PAM_PERM_DENIED` |
| `AUTHN_DENIED_USER_UNKNOWN` | `PAM_USER_UNKNOWN` | `PAM_USER_UNKNOWN` |
| `AUTHN_DENIED` | `PAM_AUTH_ERR` | `PAM_AUTH_ERR` |
| `SERVICE_UNSUPPORTED` | `PAM_SYSTEM_ERR` | `PAM_SYSTEM_ERR` |

Both columns come from one rule. `pam_sm_acct_mgmt` returns `PAM_SUCCESS` for
`OK` and the outcome's own code otherwise. `pam_sm_authenticate` does the same,
except that an authorisation outcome passed the credential step, so it returns
`PAM_SUCCESS` and leaves the denial to the `account` phase. A credential outcome
usually stops the stack before `pam_sm_acct_mgmt` runs at all; its column is
what a control flag that does not stop the stack reaches, and it fails there
too.

The `OK` row presumes the request carried a `user`. A client that sent none owes
the authorisation step itself (§6.2), and returns `PAM_SUCCESS` only once it has
discharged it.

`AUTHN_UNKNOWN` is a decision the client could not read, and §6.2.1 places it on
the credential side for that reason, so it takes the code every other credential
outcome takes. Waiting does not change it: §3.4 has a receiver reach it for a
value a later minor version assigns, and §4.1 permits that addition, so a server
upgrade would otherwise turn a denial every deployed client used to stop on into
one they fall through. `PAM_SYSTEM_ERR` would stop the stack too, and the
appendix gives it where something needs a person. Here the client read the
envelope, the message type and the response it expected, and only the outcome
value was unassigned. §4.1 lets a later minor version add to `result`, and what
it adds is the outcome of an authentication attempt, so an authentication
failure is the likelier report. `STATUS_UNKNOWN` keeps `PAM_AUTHINFO_UNAVAIL`
because it is not a decision at all: an `ERROR` message says the server decided
nothing.

A client adopting this mapping MUST fail its configuration closed. A module
argument it does not recognise MUST fail rather than be ignored, which is not
PAM's usual treatment of one, and a socket path MUST be configured, with no
default supplied by the client. §2 adds that the path be absolute and
representable in full.

A client answering `pam_sm_acct_mgmt` from a cached `result` MUST hold one
obtained in the same PAM transaction, for the `user` it is now asked about.
Where the request carried no `user`, it MUST instead hold one obtained in the
same transaction for the identity it established under §6.2. Where it holds none
it MUST NOT return `PAM_SUCCESS` or `PAM_IGNORE`, and SHOULD return
`PAM_SYSTEM_ERR`: a stack that reaches this module's `account` phase without its
`auth` phase having run, or with `PAM_USER` changed between the two, is a
configuration that needs a person. A deployment MAY override that code on the
terms above, and the bound above still holds, so no override turns the absence
of a verdict into an admission.

The emphasised cell is the appendix's one fail-open hazard, and the paragraphs
below are about it.

`AUTHN_DENIED` maps to `PAM_AUTH_ERR` because a client told nothing may not
conclude the credential was good, and `PAM_AUTH_ERR` is the outcome that assumes
least. A server unwilling to disclose a reason therefore costs its clients the
`auth`-versus-`account` distinction, which is the price of saying nothing.

`SERVICE_UNSUPPORTED` maps to `PAM_SYSTEM_ERR` and not to an authentication
failure. The `service` came from the stack's own configuration and no user chose
it, so denying the user reports the wrong party at fault, and every login
afterwards fails the same way with nothing pointing at the line that caused it.
It is the one `result` that needs a person rather than a different credential.

The split the table shows follows §6.2.1's two decisions. A credential outcome
fails the credential step, so it fails `pam_sm_authenticate`. An authorisation
outcome, which in version 1.0 means `AUTHN_DENIED_POLICY` alone, passes that
step and fails the other.

**That split is where a stack fails open.** A client that returns `PAM_SUCCESS`
from `pam_sm_authenticate` for an authorisation outcome MUST document that its
stack requires an `account` line. Without one a `AUTHN_DENIED_POLICY` admits the
caller, and the module cannot detect the omission.

Which is why §6.2.1 forbids a server answering `AUTHN_DENIED_POLICY` for a
`user` that does not resolve. That condition used to reach either code, and the
two sit on opposite sides of the split above, so the same request failed the
`auth` phase against one conforming server and passed it against another. A
deployment missing its `account` line was then exploitable or not depending on
which server it met, which is not a property a deployment can test for.
