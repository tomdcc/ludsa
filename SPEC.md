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
- **The read deadline still applies.** When a client omits the half-close, the
  server MUST fall back to the deadline rather than wait indefinitely.

**A server answers once it can.** The obligation begins when it has read
`ludsa_version_major`. That value opens every message, and until the server
holds it there is no version the client chose to answer in. Before that point
the server closes without replying, whether it received no octets at all or
fewer than that value needs.

**Answering without one.** A server that has decided to answer but has not read
`ludsa_version_major` MAY answer in its own major version instead of closing.
Before it does, it SHOULD allow the client a bounded interval to supply one,
because a client that is about to send is better answered in its own version.
§4 already has a server reply in its own major version when it cannot use the
client's; this is the same move where it has not been given one.

Having read it, the server sends a response whatever the request turns out to
be. It replies in its own major version when it does not implement the
client's. It closes without replying only when writing fails.

**The server closes after it responds.** Having written its response the server
MUST shut down its write direction or close the connection, except where §3 owes
a drain, which it completes before it closes. There is no connection reuse, so a
client may read to end of file without risk of blocking.

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

When the server's request deadline expires it responds `REQUEST_TIMEOUT` (§5)
and closes. Where the deadline expired before it read `ludsa_version_major`, it
answers in its own major version or closes without replying, as above. Where it
has already answered, having rejected the message early, it closes without
replying and §3's drain is what the deadline was bounding. When its response
deadline expires it closes without replying, because the channel it would answer
on is the one that is blocked.

A server MUST respond as soon as it determines it cannot reach a decision, with
`UNAVAILABLE` or `INTERNAL_ERROR` as §5 requires. Whether it bounds the time it
spends trying is its own choice.

A client's own deadline has no representation on the wire, since a status
appears only in a response and the client has already sent its request. The
client reports it locally and never as a `REQUEST_` code: `RESPONSE_TIMEOUT`
where no response octet had arrived, and `RESPONSE_INCOMPLETE` where part of one
had (§7). §5 gives `REQUEST_TIMEOUT` to the server's request deadline, and a
client whose own deadline expired has learned only that the server has not
answered yet.

The server SHOULD bound concurrent connections. At its limit it MAY respond
`RATE_LIMITED`, and MAY instead stop accepting and leave connections in the
kernel backlog. The limit is reached at accept, before any octet has arrived, so
a server that chooses to answer reads `ludsa_version_major` first. Where the
client supplies none within a bounded interval, it MAY answer in its own major
version. §3's drain covers what it did not read.

**Connect-time retry.** A client MAY retry a connection that failed for a reason
a later attempt could change: `ECONNREFUSED` and `ENOENT`, because the server
may not have published its socket yet; `EAGAIN`, because a full backlog may
drain; and `EINTR`, because nothing about the connection was established. It is
subject to:

- A client MUST NOT retry a permission failure, a timeout, a protocol error or a
  denial. None of those changes on a second attempt.
- A client MUST NOT open a new connection after it has written any part of the
  request. Resuming a short or interrupted write on the same connection is not
  covered by this rule.
- Every retry happens within the client's overall deadline.

A client that never establishes a connection it is willing to use reports
`CONNECTION_FAILED` (§7). That covers a permission failure, a path that is not
there, a retryable reason that had not resolved by the client's deadline, and a
peer whose identity is not the one the client was configured to expect. It is
the one outcome a client reports without an exchange to report on, so no
`RESPONSE_` name fits: a response cannot be absent from a request that was never
sent.

**A client may be configured with the server identity it expects**, as a user
name or a uid. Where it is, the client MUST confirm the peer's identity before
it writes. It MUST close the connection where it cannot, without issuing a
request, and report the condition §7 names `CONNECTION_FAILED`.

Not confirming covers two cases: a peer that is someone else, and a peer the
client could not identify. A deployment that asks for a check its platform
cannot perform therefore fails rather than proceeds.

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

**The socket path.** A server MUST refuse to start if the configured path exists
and is not a socket, and MUST apply that test before it removes or replaces
whatever is there. A stale socket MUST be gone from that path before the server
publishes its own, whether the server unlinks it or replaces it atomically.

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

A client or server MUST read `length`, then read exactly that many octets. It
MAY read further from the socket, as a buffered reader will, but MUST NOT
interpret any octet past the declared length and MUST discard what it read
beyond it. §2 governs what it does with octets it observes there. The envelope
lies outside the declared length and is read from the stream directly.

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

A server that rejects a message before it has read all of it MUST NOT allocate
storage for the body. It MUST send its response, then read and discard until the
earliest of these, and MUST then close:

- it has consumed the declared length, where it read one it may rely on;
- it reaches end of file on the read direction;
- it has read `2 x MAX_LENGTH` octets from the connection in total, counting the
  envelope and everything already read before it rejected;
- §2's accept-to-complete-request deadline expires;
- a shorter bound it set on the drain expires. A server MAY set one.

§2's deadline runs on through the drain, since the request it bounds never
completed, and it is what guarantees the drain ends. Its expiry here means
close, not answer: §2 has a server respond `REQUEST_TIMEOUT` where the deadline
expires before it has answered, and on this path it has answered already. A
server sends one response to a connection.

Where it expired before the response went out, the earliest terminator is
already met and the drain ends at once. So a server answering `REQUEST_TIMEOUT`
writes and closes, discarding nothing, and that is the only rejection with no
drain behind it: every other one is settled while the deadline still runs. The
client is not left without its answer, since a response already written survives
the close (§2), and it may take `EPIPE` on a write it had not finished.

The status it sends is the one it had determined when it rejected the message.
Neither §2's deadline nor a bound of its own, expiring during the drain, changes
it to `REQUEST_TIMEOUT`.

A server answering `VERSION_UNSUPPORTED` has read no length it may rely on, so
the declared-length terminator does not apply to it and the rest do.

**A client or server MUST NOT allocate storage from any length or count read
from a message before checking it against the extent of the structure that
contains it.** That covers `length` against `MAX_LENGTH`, a field's `len`
against the remaining message.

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
asked to send one reports the condition §7 names `INVALID_REQUEST` to its host
application, without opening a connection.

`error_detail`, `server_desc` and `authn_id` are the only fields in this table
that a server sends and a client receives. A defect in any field a server sends
is a fault in the server that produced it, so a client MUST NOT report a
request-side code for it. Nothing in
the client's own request caused it. §5 says what a client does instead.

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

Ascending order binds every message, so a receiver checks it against the tag
before it without knowing which schema applies. Whether a repeat is legal does
not: only the message type says. §4 therefore checks the two at different
points, and a receiver that rejects a repeat before it knows the type is
refusing a message it cannot adjudicate.

The last binds a sender to its own version and not to 1.0, because a later
version emits tags 1.0 never assigned (§4.1). What it forbids is emitting a tag
no version defines. A receiver skips one, so nothing breaks today; what breaks
is later, when a minor version assigns that tag and a peer implementing it reads
the padding as the field.

It binds a sender. What a receiver does about a breach is §4's: it MAY reject
where the message declares a minor version it implements, and MUST skip where it
does not, since only the first leaves it holding the schema that settles the
question.

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

**A tag's shape is checked before its identity.** A receiver MUST reject a tag
as `REQUEST_MALFORMED` on any of these grounds, whether or not it recognises
the tag, and whether the tag is required or optional:

- bit 6 is set and either of bits 5-4 is set;
- bit 6 disagrees with whether the tag's assigned type is fixed or dynamic;
- the field number is zero.

The first and third read only bits every tag carries, so they reach a tag whose
type is unassigned like any other. The second cannot: nothing states whether an
unassigned type is fixed or dynamic, so a tag carrying one that passes the other
two is unrecognised rather than malformed, which is what keeps §4.1's promise
about a new value in the type table.

A tag that cannot describe a field is not a field to skip, and skipping one
would mean trusting a width the tag itself contradicts. Field number 0 is
reserved, so no tag carrying it names a field, and rejecting it is what makes
zero safe as the initial value of the previous-tag comparison §3.2 describes.
Holding bits 5-4 at zero on a dynamic tag is what lets a receiver compare a
tag as one whole 16-bit value. Only a tag that passes all three is compared
against a schema.

**A tag is validated whole.** A client or server compares a received tag against
the tags its schema defines for that message type. It MUST NOT parse a field
according to the type or width the tag claims when the tag as a whole is one it
does not recognise.

For a tag it does not recognise:

- **Required**: it MUST abort the exchange and return `REQUEST_MALFORMED`.
- **Optional**: where the tag passed the shape checks above, it MUST skip the
  field without error, using bit 6 and, when clear, bits 5–4 to know how many
  octets to skip. §4 gives it one alternative, where the message declares a
  minor version it implements.

A receiver skips every occurrence of an unrecognised optional tag, including one
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

A sender MUST NOT emit 0 or 1. A receiver that meets either, or any other value
it does not recognise, MUST treat it as 1, and MUST NOT treat it as an
affirmative outcome.

**No value assigned in one enumeration is affirmative in another.** Holding the
affirmative outcome at 2 everywhere, and leaving 2 unassigned in an enumeration
that has none, is what secures that. A receiver that reads an enumerated value
out of the wrong message type therefore cannot turn a refusal into an approval,
whatever else it gets wrong.

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
`VERSION_UNSUPPORTED` (§5) in its own major version.

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

**A receiver parses by tag and not by the declared version.** It skips a tag it
does not recognise and reads one it does, and the version a message declares
never selects the schema it parses under.

That value settles one thing. Where a message declares a minor version the
receiver implements, and carries a tag that version does not define for that
message type, the sender has broken §3.2 and the receiver holds the schema that
proves it. It MAY reject the message as `REQUEST_MALFORMED`, and it MAY skip the
tag as §3.3 has it skip any other. Neither is preferred and both conform, so a
deployment that wants strict input in an authentication path may have it and one
that wants tolerance may have that.

Where the declared minor version is one the receiver does not implement, it MUST
skip. It holds no schema that would settle whether the tag is legitimate, and
§4.1 leaves a later version free to define it, so a receiver rejecting there
would fail a conforming peer.

The minor version is for logs, and a server MAY tailor what it sends to the one
a request declares (§3). It MUST NOT inform an authentication decision, and it
states nothing about which optional fields a peer populates: a field a later
version adds is optional (§4.1), so its absence never separates a peer that
predates it from one that had no value for it.

The protocol has no negotiation. Each side declares its own version, and §3.3
makes a minor difference safe without either side agreeing to anything.

**Validation proceeds outermost first**, and a message failing more than one
check takes the status of the earliest:

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
connection rather than the message, and §5 has a server answer a refusal at the
earliest point it can, which for a server whose refusal does not turn on the
request is before any of this order has run.

Step 4 checks only what a receiver can check without knowing which schema
applies. A tag equal to the tag before it is a repeat, which step 7 judges
against the schema. Steps 7 to 10 need the schema that `msg_type` selects, and
cannot precede step 5. Step 6 binds a client only, since only a client has a
request to compare against.

Step 3 is the only step that requires waiting. Every other check runs on octets
already read, so a receiver MAY report a fault it has already detected without
waiting for the declared length to arrive. A message that is both truncated and
faulty in the octets that arrived may therefore take either status, and both
conform. A message truncated with no fault in the octets that arrived is
`REQUEST_INCOMPLETE`.

**Neither of those is forced, deliberately.** Forcing the fault would oblige a
receiver that reads a message whole to parse a prefix it has already given up
on. Forcing the truncation would oblige a receiver that validates as it reads to
wait for the rest of a message it has already judged. Each rule conscripts one
of the two ways of reading a socket, and nothing this protocol governs turns on
the answer: a peer that sends a message both short and faulty is broken twice
over, and one that stopped because it died is not reading the response whichever
status it gets.

Steps 10 to 13 are not detectable early. A required field is missing only once
the message ends, and the selectors, limits and decision need the whole message,
so a receiver reaches them only after the declared octets arrive.

Steps 8 and 9 yield the same status, so the order between them is not
observable and a receiver taking them the other way round conforms. The order
shown spares a receiver from scanning a value it is about to reject.

### 4.1 What a later version may change

Within major version 1, a later version MUST NOT change any of the following:

- A tag. Its number, type, width and required status are all part of its value.
- A field number, once used in a message type, for anything else in that
  message type. A number whose field is removed is retired there and MUST NOT be
  reassigned. Numbers are independent across message types (§6).
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

A new optional field MAY be repeatable from the version that assigns it. A peer
that predates it does not recognise the tag, so it skips each occurrence (§3.3)
rather than adjudicating a repeat it has no schema for. What it may not do is
become repeatable later, which is the frozen rule above.

A field a later version adds to a message type MUST take a number higher than
every number already assigned in that type. That is what makes §3.3's ordering
property hold: the field sorts after every field already defined there, so an
encoder appends rather than inserting. Numbers are not dense within a type, and
a gap the numbering left stays empty.

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

§3.2 binds a server to the fields its own version defines, which for version
1.0 are these two, whatever the request it answers. A receiver applies §3.3 to
any other tag it meets: it skips an optional one and rejects the message for a
required one. A server SHOULD populate `error_detail`.

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
no direction and a client reports them unchanged. `STATUS_UNKNOWN` belongs to
neither family and a server never emits it: §3.4 forbids a sender from emitting
any enumeration's 1, and it is what a client records for a status it does not
recognise. §7 gives a client the names for what it finds in a response.

A client MUST NOT report a `REQUEST_` code for a fault it found itself. The
names exist so a caller can tell the two apart. An operator reading
`INVALID_REQUEST` (§7) shortens a credential. One reading `REQUEST_TOO_LARGE`
has a peer that declared a length §3 forbids it to send. One reading §7's other
response codes goes to the server.

`REQUEST_MALFORMED` means the sender produced something §3 forbids it to send:
invalid structure, a value its type rules reject, a field above its §3.1
maximum, or a tag its declared version does not define, which §4 leaves a
receiver free to reject or to skip. A message that stopped early is
`REQUEST_INCOMPLETE` instead, because a
peer that crashed or was killed produced a short message with no defect in its
encoder. `REQUEST_INCOMPLETE` covers a message that ended before its declared
length, and one that ended before it declared a length at all.

`REQUEST_TOO_LARGE` is what the envelope decides. A receiver reaches it from the
declared length alone and answers without parsing any further, which is the
whole of its value. A field above its §3.1 maximum is not `REQUEST_TOO_LARGE`,
because a receiver finds one only inside a body it has already accepted.

`PEERCRED_DENIED` is for a server that checks the connecting peer, which this
document does not require. **A server that refuses a peer SHOULD answer at the
earliest point it is permitted to answer**, and where it makes the check is what
decides that point:

- **At accept.** It has no version from the client, so §2 lets it answer in its
  own major version at once, or close without replying.
- **After reading `ludsa_version_major`.** It answers in the version the client
  chose. §2 binds it from that point: it MUST answer, and `PEERCRED_DENIED` is
  the status for this refusal.
- **After reading the request**, where the refusal depends on what the request
  says. A server whose rules govern which peers may reach which `service`
  values cannot decide before it has read one.

A server with no such rules SHOULD answer after the version word, or after a
bounded interval waiting for one, that being the earliest point at which it can
answer in the client's own version.

A server may hold both a peer refusal and a status from §4's order. Where one
was available earlier, answering at the earliest permitted point means answering
that one. Where both arrive together, either conforms: a peer that is not
permitted and whose `ludsa_version_major` this server does not implement is
misconfigured twice over, and this document does not choose which half it hears
about. A request too malformed to yield the `service` a server's rules turn on
settles the question by itself, since the peer check cannot complete and the
framing status is the only answer it has.

It discloses that the peer was refused and why. That is a smaller disclosure
than it looks, because a uid is not freely chosen: an attacker must already be
running as one to try it, where a user name can be guessed without limit. A
server unwilling to disclose even that makes the check before it reads a
version, which is the point at which saying nothing is still open to it.

`RATE_LIMITED` has two producers and they sit at opposite ends of an exchange.
§6.3 has a server decline a check that `max_age` would have forced, which is
work a request asked for. §2 has a server decline at its concurrency bound,
which is reached at accept before any octet has arrived, so nothing has asked
for anything yet. What they share is the only thing a client acts on: the server
had capacity for this and has none now, so a later attempt may fare better.

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
log without escaping it. The guarantee is about the value's content, not about
what a receiver may pass it to: it is data, and a receiver MUST NOT use it as a
format string, a template, or anything else that interprets what it contains. A
server that embeds text from elsewhere in it MUST render that text safely
first, and MUST NOT let that text push the field above its §3.1 maximum. It
shortens what it embeds, not the message. §3.1's rule against truncating a
value binds a receiver rejecting one, not a server describing one.

A client MUST NOT log `error_detail`, return it, or otherwise act on it until it
has established that the value is valid UTF-8 (§3.3), holds no control character
and is within its §3.1 maximum. A client that does not use the value need not
check it. One that checks and finds any of those faults SHOULD report the
response as malformed (§7). The same holds for `server_desc` and `authn_id`.

A server is bound to emit none of those faults. §5 forbids the control character
and the encoding, §3.1 bounds the length, and §3.3 forbids the NUL. So a client
meeting one has found a bug in the server rather than a value to work around,
and reporting it is what gets the bug fixed. More tolerant handling is a
client's own choice, and this document does not specify it.

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

A server MUST send one of them, and a client MUST establish a response's
`msg_type` before it reads the value of any field in it. `ERROR` is the only
message that carries a status, so a response carrying any other type is itself
the statement that the server answered the request it was sent.

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
| `server_desc` | `utf8` | optional | `0x0443` |
| `uptime_seconds` | `integer`, 8 octets | optional | `0x0530` |

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
default policy.

`credential` is carried **verbatim, as raw bytes with an explicit length**, and
is never re-encoded. **The client and the server MUST NOT normalise it.** No
trimming, no stripping of a trailing newline, no case folding. A server SHOULD
still detect leading or trailing whitespace and record the condition, not the
value, in its own log. A credential that fails validation is a decision
(§6.2.1), and a decision carries no field for a diagnosis.

`pid` is for log correlation only. The client self-reports it, and it MUST NOT
inform any decision.

Response:

| field | type | | |
|---|---|---|---|
| `result` | `enum`, 1 octet | required | `0x0281` |
| `authn_id` | `utf8` | optional | `0x0343` |

`authn_id` is the canonical identity the server resolved, for example
`system:serviceaccount:app:web`. A client that cannot use it ignores it. A
server MAY omit it whether or not the request carried a `user`, and version 1.0
gives a client no way to require one. A client that omits `user` and cannot
derive the identity from the credential therefore depends on being deployed
against a server that supplies `authn_id`. That is an arrangement between a
client and a deployment, not something the protocol settles.

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
| 5 | `AUTHN_DENIED_USER_UNKNOWN` | no mapping exists for the identity |
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
the credential was good. `SERVICE_UNSUPPORTED` is not a decision, and a client
fails both steps on it.

Nothing here obliges a server to check a credential before it answers.
`AUTHN_DENIED_USER_UNKNOWN` in particular does not imply that the credential was
valid, and a server MAY answer it without having examined one.
`AUTHN_DENIED_POLICY` is the exception: a server MUST NOT answer it unless it
established that the credential was valid, since that is what the code states.

It carries a second precondition for the same reason. A server MUST NOT answer
`AUTHN_DENIED_POLICY` where no mapping exists for the requested `user`, or where
it resolved no identity from the credential. Those are what
`AUTHN_DENIED_USER_UNKNOWN` names. The line is whether the name resolves and not
which order a server did its work in. A name that resolves and is then refused
is `AUTHN_DENIED_POLICY`, and a name that does not resolve is
`AUTHN_DENIED_USER_UNKNOWN` whether or not the credential was examined first.

Neither precondition asks a server to disclose anything. Every code that names a
reason is optional and `AUTHN_DENIED` is always available, so this binds only a
server that has chosen to name one, and says which name is true.

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
| `serving` | `enum`, 1 octet | required | `0x0381` |
| `server_desc` | `utf8` | optional | `0x0443` |

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
| `PING_RESPONSE` `0x02` | `0x0443` | 4 | no | dynamic | `utf8` | `server_desc` |
| `PING_RESPONSE` `0x02` | `0x0530` | 5 | no | 8 octets | `integer` | `uptime_seconds` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x01C2` | 1 | **yes** | dynamic | `string` | `service` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x02C4` | 2 | **yes** | dynamic | `bytes` | `credential` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x0342` | 3 | no | dynamic | `string` | `user` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x0442` | 4 | no | dynamic | `string` | `rhost` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x0542` | 5 | no | dynamic | `string` | `ruser` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x0642` | 6 | no | dynamic | `string` | `tty` |
| `AUTHENTICATE_REQUEST` `0x03` | `0x0720` | 7 | no | 4 octets | `integer` | `pid` |
| `AUTHENTICATE_RESPONSE` `0x04` | `0x0281` | 2 | **yes** | 1 octet | `enum` | `result` |
| `AUTHENTICATE_RESPONSE` `0x04` | `0x0343` | 3 | no | dynamic | `utf8` | `authn_id` |
| `STATUS_REQUEST` `0x05` | `0x0120` | 1 | no | 4 octets | `integer` | `max_age` |
| `STATUS_REQUEST` `0x05` | `0x0242` | 2 | no | dynamic | `string` | `service` |
| `STATUS_RESPONSE` `0x06` | `0x0381` | 3 | **yes** | 1 octet | `enum` | `serving` |
| `STATUS_RESPONSE` `0x06` | `0x0443` | 4 | no | dynamic | `utf8` | `server_desc` |

One tag appears in two groups: `0x0443` is `server_desc` in `PING_RESPONSE` and
in `STATUS_RESPONSE`, which is the same field with the same meaning. No tag
names two different fields.

A tag's width and type are fixed by the field it carries, and its required bit
by the field's role in that message type. Its field number is free, so a
collision between two different fields is a choice rather than a consequence.
§3.3 matches a tag against the group its `msg_type` selects, so one would be
harmless to a conforming receiver; the numbering avoids
it anyway, because a decoder that reads fields before it consults the type then
finds an unrecognised optional tag and skips it, rather than reading one field's
value as another's.

Field number 0 is reserved (§3.3). Every other number here is either assigned
above or unassigned, and none is retired. A number whose field is later removed
becomes retired and is never reassigned (§4.1).

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
every case driven against a client rather than of any one case, so the column
does not repeat it.

Every case driven against a server is answerable, because each carries at least
a version word and a server can answer in its own major version (§2). A case
driven against a client has nothing to answer, so it reports to its caller
instead. Where a case is marked `either`, the status shown is the server's and
the client's counterpart is the name paired with the same condition below.

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
| 19 | The request deadline expiring, after `ludsa_version_major` has been read | server | `REQUEST_TIMEOUT` | F6 |
| 20 | A response carrying a `msg_type` the client implements, which the request does not permit (§6) | client | `RESPONSE_MALFORMED` | |
| 21 | A field whose value exceeds its §3.1 maximum, in an otherwise well-framed message | server | `REQUEST_MALFORMED` | |
| 22 | An unimplemented `msg_type` in a message that also has a repeated tag | either | `REQUEST_UNSUPPORTED` | |
| 23 | An unrecognised `service` in a message that also omits `credential` | server | `REQUEST_MALFORMED` | F1 |
| 24 | A response of exactly one octet, then end of file | client | `RESPONSE_INCOMPLETE` | |
| 25 | A response carrying a `msg_type` the client does not implement | client | `RESPONSE_UNSUPPORTED` | |
| 26 | A tag with bit 6 set and either of bits 5-4 set, in a message whose `msg_type` the receiver does not implement | either | `REQUEST_MALFORMED` | |
| 27 | A declared length above `MAX_LENGTH`, in a message whose octets also carry a NUL | either | `REQUEST_TOO_LARGE` | |
| 28 | The connection accepted and the request read, then closed with no response octet | client | `RESPONSE_ABSENT` | |
| 29 | The connection accepted and the request read, then held open past the client's deadline | client | `RESPONSE_TIMEOUT` | F6 |

Cases 17 and 30 are retired. 17 drove an unrecognised `service`, which §6.2.1
and §6.3 now answer with `SERVICE_UNSUPPORTED` rather than a rejection. 30 drove
a truncated message whose arrived octets carried a fault, which §4 leaves to the
receiver. Neither number is reused, on the ground §4.1 gives for a field number:
a citation that outlives what it named should fail to resolve rather than
resolve to something else.

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
status (§4). Both conform, so it is not a case here and a driver MUST NOT record
either as a failure.

A message declaring a minor version the receiver implements, carrying a tag that
version does not define, may draw `REQUEST_MALFORMED` or be accepted with the
tag skipped (§4). Both conform, so it is not a case here either.

Some of these depend on how the peer under test is configured, which this
document does not specify. A driver MUST be given all of them:

- **F1**: a `service` value the peer recognises, and one it does not.
- **F2**: whether the peer implements the rate limiting §6.3 permits. Case 18
  applies only where it does, and a peer without a limit is not non-conforming
  for never exhibiting it.
- **F3**: a credential the peer accepts, and a `user` its policy does not map
  that credential's identity to. The policy must also refuse that identity a
  `user` of zero octets, which a deployment whose policy grants any `user`
  cannot supply. Such a peer cannot run the second decision case and is not
  non-conforming for that.
- **F4**: the bound the peer sets on the drain (§3), where it sets one. §2's
  request deadline bounds the drain otherwise, and F6 supplies that. A server
  whose bound is shorter than the driver's round trip has stopped draining
  legitimately and cannot be told from one that never drained.
- **F5**: the server identity the client under test is configured to expect, and
  a driver socket owned by a different uid. A client configured with none owes
  nothing under §2 and cannot run the case below.
- **F6**: the deadlines the peer applies (§2). A driver needs the server's
  accept-to-complete-request deadline to know how long to stall, and the
  client's overall deadline to know how long to hold a connection open and how
  long to wait. Neither has a value this document sets.
- **F7**: the names the client under test reports, mapped onto the client-local
  names below, and the channel a driver reads them on. Every case driven against
  a client needs it, including one marked `either`. None of those names crosses
  the socket and a client using its own is conforming, so a driver can infer
  neither the names nor the channel from anything the wire carries.

One further case is a decision rather than a rejection, and a driver exercises
it too. It is the only test of §6.2's binding rule, which no wire-level input
can reach:

| case | against | expected | needs |
|---|---|---|---|
| A credential valid for one identity, sent with a `user` that identity may not act as | server | an `AUTHENTICATE_RESPONSE` whose `result` is not `OK` | F3 |
| A credential the peer accepts, sent with a `user` of zero octets | server | an `AUTHENTICATE_RESPONSE` whose `result` is not `OK` | F3 |

Further cases exercise the selector, which is neither a rejection nor a
decision. Each drives the same condition into the enumeration its response type
carries:

| case | against | expected | needs |
|---|---|---|---|
| An AUTHENTICATE request naming a `service` the server does not recognise | server | an `AUTHENTICATE_RESPONSE` whose `result` is `SERVICE_UNSUPPORTED` | F1 |
| A STATUS request naming a `service` the server does not recognise | server | a `STATUS_RESPONSE` whose `serving` is `SERVICE_UNSUPPORTED` | F1 |

One further case observes what a server does after it has answered, which no
status can show. The driver writes an 8-octet envelope declaring a
`ludsa_version_major` the server does not implement, reads the response, and
then writes further octets without half-closing:

| case | against | expected | needs |
|---|---|---|---|
| Further octets written straight after a `VERSION_UNSUPPORTED` response, on a connection the driver has not half-closed | server | the write is accepted and the driver terminates normally | F4 |

§3 has the server draining at that point, so it consumes the write. A server
that closed instead gives the driver `EPIPE`, or `SIGPIPE` where it has not
suppressed one. A driver may suppress the signal or may treat termination by it
as the failing outcome, and this document does not choose.

A server that shuts down its write direction and never reads also accepts the
write, so this case detects a server that closed rather than one that did not
drain. Whether a drain ran is not observable from outside, which is what F4
concedes for the bound.

The remaining terminators are ruled out by construction. §3 puts the declared
length out of play for `VERSION_UNSUPPORTED`, the driver does not half-close, so
there is no end of file, and an envelope plus a kilobyte is far below
`2 x MAX_LENGTH`.

A driver checks the encoding of every response it receives: tags in ascending
order, and each recognised tag appearing at most once. Where a response declares
a `ludsa_version_minor` the driver implements, it also records a tag that
version does not define for that message type, which is the sender breaking
§3.2. It records rather than rejects: whether to reject is a receiver's own
choice (§4), and against a higher minor version the check cannot be made at all,
since the driver does not hold the schema that would settle it. One further case
pins what no other input reaches, since a server's own version word is not
something a request can provoke:

| case | against | expected | needs |
|---|---|---|---|
| Any request carrying a `ludsa_version_minor` other than the server's own, which a `PING_RESPONSE` envelope reports (§6.1) | server | a response carrying the server's own `ludsa_version_minor`, not the one sent | |

§3.2's rule against emitting an optional field that has no value is not
driveable. A zero-length value is a value (§3.2), so nothing distinguishes the
two from outside.

One further case observes what a client does before it writes anything, which
§2's conditional obligation reaches and no message can. The driver accepts the
connection and writes nothing:

| case | against | expected | needs |
|---|---|---|---|
| A connection accepted by a peer whose identity is not the one the client was configured to expect | client | the client writes no octet and reports `CONNECTION_FAILED` | F5 |

This is the only case that turns on who owns the socket rather than on the
octets crossing it, so a driver needs two uids where every other case needs a
byte string.

One further case observes what a client does after it writes, which §2 requires
of every client and no status reports. The driver accepts a connection, reads
the request, and answers it:

| case | against | expected | needs |
|---|---|---|---|
| A request written in full by the client under test | client | the driver reads end of file, and its response is then accepted for writing | |

The two assertions are the two halves of §2's rule, and one of them fails
without the other. A client that shuts down its write direction gives the driver
end of file and stays readable. One that closes the connection outright gives
the driver end of file too, and the write that follows draws `EPIPE`, or
`SIGPIPE` where the driver has not suppressed one. The terms are §3's drain
case's: a driver may suppress the signal or may treat termination by it as the
failing outcome, and this document does not choose.

Nothing here obliges a server to wait for end of file. It has the declared
length and reads that many octets, so waiting would add a round trip to every
exchange, and §2's check on octets beyond the declared length is a SHOULD it
need not make exhaustive.

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

A receiver MUST accept a message whose `ludsa_version_minor` differs from its
own, in either direction, and MUST skip an unrecognised **optional** tag without
error, including one that appears more than once. The skipping obligation is
absolute only for a minor version the receiver does not implement; §4 lets it
reject instead where it holds the schema that proves the tag illegitimate.

A receiver MUST accept a message whose declared length is exactly the minimum
for the version, which a `PING_REQUEST` carrying no fields is, and a field whose
value is exactly its §3.1 maximum. It
MUST report an enumerated value it does not recognise as that enumeration's 1
(§3.4) and MUST NOT report it as an affirmative outcome, so a `result` of 200 is
`AUTHN_UNKNOWN` and never `OK`, and so is a `result` of 0.

A receiver MUST skip an unrecognised optional tag whose field number equals that
of a tag the message type does define, and read the defined field from its own
whole tag. §3.3 matches a tag whole and never by field number. `0x0242` placed
before `0x02C4` in an `AUTHENTICATE_REQUEST` drives this against a server, and
`0x0243` before `0x0281` in an `AUTHENTICATE_RESPONSE` against a client.

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
| `RESPONSE_INCOMPLETE` | a response started and stopped part-way, at end of file or at the client's deadline |
| `RESPONSE_ABSENT` | no response octet arrived, and the server closed |
| `RESPONSE_TIMEOUT` | no response octet arrived, and the client's deadline expired |
| `CONNECTION_FAILED` | the client never established a connection it was willing to use |
| `INVALID_REQUEST` | the caller's own input could not be carried in a conforming request |

A client whose host wants a distinct code per condition MAY subdivide these,
`INVALID_REQUEST_CREDENTIAL_TOO_LARGE` and so on. What it MUST NOT do is report
a code a server sends for a fault it found itself. §5's statuses describe a
request a server received, and a request that was never sent gives them nothing
to describe.

The table above binds a client by its conditions, not by these names. Where it
gives a status for a case marked `either`, a client detects the same condition
and reports the response name beside it here. Where a case is marked `client`,
the status column already holds one of these names, and the same latitude
applies to it.

A code that carries no direction serves both sides. A client reports
`VERSION_UNSUPPORTED` for a response major version it does not implement, and
relays `UNAVAILABLE`, `INTERNAL_ERROR`, `RATE_LIMITED` and `PEERCRED_DENIED` as
the server sent them.

A client that reads no response octet reports `RESPONSE_ABSENT` where the server
closed and `RESPONSE_TIMEOUT` where its own deadline expired. One that reads any
octet and then gets neither the rest of the response nor a decision reports
`RESPONSE_INCOMPLETE`, whichever of the two ended it. It MUST NOT report any of
them as a decision.

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
turn any failure into an admission. The test is what the stack does, not what
one entry point returns: the `PAM_SUCCESS` this appendix gives
`AUTHN_DENIED_POLICY` at `pam_sm_authenticate` admits nobody by itself, because
the same outcome denies at `pam_sm_acct_mgmt` and the rule below binds a client
that relies on it.

`CONNECTION_FAILED` and `PEERCRED_DENIED` describe one condition from opposite
ends, a client and a server that a configuration does not permit to talk. A
client that offers an override SHOULD apply one setting to both, so an
administrator who changes how a peer refusal behaves changes all of it.

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
| `AUTHN_UNKNOWN` | `PAM_AUTHINFO_UNAVAIL` | `PAM_AUTHINFO_UNAVAIL` |
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
`user` it has no mapping for. That condition used to reach either code, and the
two sit on opposite sides of the split above, so the same request failed the
`auth` phase against one conforming server and passed it against another. A
deployment missing its `account` line was then exploitable or not depending on
which server it met, which is not a property a deployment can test for.