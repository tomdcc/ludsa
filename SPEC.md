# LUDSA protocol specification

**L**ocal **U**nix **D**omain **S**ocket **A**uthentication

Status: **draft v0.1**. Protocol version 1 is not frozen; field tags may still
be renumbered. Everything marked *open* at the bottom is undecided.

This document states what conforming implementations must do. Rationale is kept
to the minimum needed to stop a rule reading as arbitrary.

## 1. Purpose and actors

LUDSA is a request/response protocol by which a process that must make an
authentication decision defers that decision to another process on the same
machine, reached over a unix domain socket.

It is deliberately backend-agnostic. The protocol says nothing about what a
credential is or how it is checked; a conforming daemon may validate JWTs,
query a directory, or read a flat file.

| role | description |
|---|---|
| **client** | The process holding a credential it cannot itself validate. A PAM module, or a PostgreSQL OAuth validator module. |
| **daemon** | The process that decides. Holds whatever upstream connections, caches and policy the decision requires. |

Known clients: `pam_ludsa` (PAM module), `pg_ludsa` (PostgreSQL OAuth
validator, planned), `ludsa-probe` (diagnostics), `ludsa-conformance` (test
driver).

Known daemons: `ludsa-k8s` (Kubernetes TokenReview), `ludsa-file` (reference
implementation, flat file).

## 2. Transport

**Unix domain stream sockets only.** TCP MUST NOT be used, including on
loopback. The credential crosses this channel in cleartext, and filesystem
permissions are the only access control the protocol relies on.

**One request per connection.** The client connects, writes exactly one
request, reads exactly one response, and closes. There is no pipelining and no
connection reuse, so there is no desynchronisation state to reason about.

**Half-close after the request.** Having written the request, a client MUST
shut down its write direction (`shutdown(SHUT_WR)`, `CloseWrite()`), leaving
the read direction open for the response.

The requirement is asymmetric because the situations are. A client holds the
connection open to read the reply, so without an explicit half-close the server
has no end-of-request signal beyond the length prefix, and a client that dies
mid-write leaves it blocked until its read deadline. The half-close turns that
into an immediate error. A server needs no equivalent rule: it closes after
replying, which produces EOF at the client anyway.

Two consequences a receiver MUST observe:

- **EOF on the read direction is not a reason to abandon the exchange.** The
  connection remains writable, and a response is still owed. Treating a
  half-close as a closed connection is the obvious way to implement this
  wrongly.
- **The read deadline still applies.** EOF is an optimisation for the
  conforming case, not a replacement for the timeout — a client that omits the
  half-close must degrade to the deadline rather than hang the server
  indefinitely.

EOF arriving before the declared length has been read is `MALFORMED`.

**Octets beyond the declared length.** A receiver MUST NOT process anything
past the declared length, and SHOULD reject the exchange as `MALFORMED` if it
observes any — the same principle as trailing octets after `TAG_END`, applied
one level up. A sender and receiver disagreeing about where a message ends
means one of them has a framing bug, and that is worth failing loudly on.

The check MUST be non-blocking. Reading again and requiring EOF would deadlock
against a client that omitted its half-close: it waits for the response while
the receiver waits for EOF, until the deadline. A non-blocking read that finds
data rejects, and one that yields `EAGAIN` proceeds. This makes the check
best-effort — excess octets may not have arrived yet — which is acceptable
because it detects implementation bugs rather than enforcing a security
boundary.

**Connect-time retry.** A daemon restart leaves a brief window in which the
socket path exists but nothing is listening. A client MAY retry, subject to:

- Retry only `ECONNREFUSED` and `ENOENT`. A timeout means the daemon is up and
  struggling, so retrying adds load while consuming the caller's authentication
  budget; protocol errors and denials are answers, not transient faults.
- Never retry once any part of the request has been written. Retry belongs
  before the first byte, or not at all.
- The configured timeout is an **overall deadline** covering connect, retries
  and the exchange — not per attempt, which would multiply.
- Keep the budget to a few hundred milliseconds.

This covers the bind race only. A full daemon restart — exec, config load, key
fetch, bind, plus any supervisor backoff — outlasts any sane retry budget, and
the correct response to that is for callers to reconnect.

**Socket permissions.** The daemon MUST create the socket with mode `0660` or
stricter. Deployments are expected to place client and daemon in a shared
group.

Restrict the containing **directory** as well, not only the socket. `bind()`
creates the inode with default permissions and a separate `chmod()` follows, so
there is a window in which the socket is more permissive than intended;
directory permissions close it. A daemon MUST also unlink a stale socket before
binding — the inode survives a crash, and `bind()` would otherwise fail with
`EADDRINUSE`.

**Peer credentials.** The daemon SHOULD check `SO_PEERCRED` and refuse
connections whose peer uid is not in a configured allowlist. Note that the peer
*pid* reported by `SO_PEERCRED` is meaningless across PID namespaces (it will
generally be 0 when client and daemon are in separate containers); only the uid
is dependable there.

**Timeouts.** Both sides MUST apply read and write deadlines. A client's
deadline must be comfortably inside whatever authentication timeout its host
application enforces.

A server MUST apply a deadline spanning **accept to complete request**, and a
separate deadline on writing the response. On expiry it closes without
replying: a request that never completed leaves nothing useful to say, and a
client that is not reading will not receive it anyway. The client observes EOF
and reports a transport error.

One total deadline is sufficient; separate idle tracking is not needed. Messages
are bounded by `MAX_MESSAGE_SIZE` and the transport is local, so a peer that has
not finished within a few seconds is stuck rather than slow, and there is
nothing to be gained by trickling.

A server SHOULD also bound concurrent connections. Note this is robustness
rather than defence: anything able to reach the socket is already an
allowlisted uid, so the concern is a wedged client consuming capacity and
blocking legitimate authentications, not an adversary.

### 2.1 Separate containers is a primary deployment

Client and daemon running in **different containers of the same pod**, sharing
only the volume that carries the socket, is a required and first-class
deployment — not a degraded variant of running side by side. The same
constraints apply to unrelated processes on one host under different users, so
nothing below is Kubernetes-specific.

Consequences, all normative:

- **Nothing may be passed by reference.** The two sides share no filesystem
  beyond the socket's own directory, no memory, and no process namespace. Every
  value the daemon needs MUST travel in the message. No field may carry a path,
  a file descriptor, or a shared-memory identifier — now or in any later
  version.
- **Peer pid MUST NOT inform any decision.** Separate containers mean separate
  PID namespaces, in which `SO_PEERCRED` reports pid 0. This is why `pid` in
  AUTHENTICATE is self-reported and marked logging-only; it is not laziness but
  the only thing that can work.
- **The peer uid allowlist MUST be configurable.** Client and daemon run as
  different users. A daemon that compares the peer uid against its own is
  correct only when co-located, and MUST NOT do so by default.
- **Lifecycles are independent.** Either side may restart without the other. A
  daemon MUST remove a stale socket before binding. A client MUST NOT retain a
  connection or descriptor between requests. Startup ordering is a deployment
  concern and is not guaranteed by the protocol.

## 3. Framing

All integers are **big-endian**, regardless of host architecture. Test vectors
depend on this.

```
message := length header field* end

length  := u32          octets following this field, excluding it
header  := u32 version  protocol version (1)
           u32 action   action code

field   := u16 tag
           u32 len      octets of value
           u8  value[len]

end     := u16 0xFFFF   TAG_END; no length or value follows
```

A reader MUST read `length`, then read exactly that many octets, then parse
within that buffer only. It MUST NOT read past the declared length even if the
socket has more data.

`MAX_MESSAGE_SIZE` is 65536 octets. A reader MUST reject a declared length
exceeding it without reading the body.

### 3.1 Field maxima

A whole-message bound is not sufficient. Implementations receive into
fixed-size storage, so a field limit must be part of the protocol rather than a
private choice — otherwise a conforming daemon may emit a value that a
conforming client is obliged to reject, which is an interoperability failure
wearing the costume of a protocol error.

| field | maximum octets |
|---|---|
| `user` | 256 |
| `service` | 64 |
| `authtok` | 32768 |
| `rhost`, `ruser`, `tty` | 256 |
| `authn_id` | 512 |
| `error_detail` | 512 |

A field exceeding its maximum is `MALFORMED`. Receivers MUST NOT truncate an
identity to fit: shortening `authn_id` merges two principals into one, which is
the same collision hazard as truncating a role name, and silently grants
whatever the shortened identity maps to.

For scale, a Kubernetes identity is `system:serviceaccount:` plus a namespace
of at most 63 octets and an account name of at most 253, so 339.

### 3.2 Deterministic encoding

Required, so that conformance vectors compare byte-exact:

- Fields MUST be emitted in ascending tag order.
- Each tag MUST appear at most once.
- Absent optional fields are omitted entirely, never sent empty.

### 3.3 Critical and skippable tags

**Odd tags are critical. Even tags are skippable.**

A receiver encountering an unrecognised **odd** tag MUST abort the exchange and
return `MALFORMED`. An unrecognised **even** tag MUST be skipped without error.

This exists because skip-unknown is a fail-open evolution mode, which is
unacceptable in an authentication protocol. If a future version adds a field
meaning "the audience check was not performed", an older peer that skipped it
would read a qualified answer as an unqualified approval. Such a field is given
an odd tag, and old peers are forced to refuse rather than misinterpret.

Additive changes that are genuinely safe to ignore take even tags and need no
version bump.

## 4. Versioning

`version` appears in the fixed header of every message, before any field, so
mismatch is detectable before parsing anything else.

A receiver MUST reject a message whose version it does not implement. There is
no negotiation and no compatibility shim.

Note that the two ends are *not* deployed together — under §2.1 they are
separate images on separate release cadences, so skew is a normal operational
state. Rejecting outright makes an incompatible pair fail immediately and
identically rather than misparse into a wrong authentication decision.

Which versions may be combined is therefore a matter for release documentation
on both sides. The Go module's semantic version is **not** the compatibility
contract. Only the protocol version is.

## 5. Messages

### 5.1 PING — action `0x01`

Liveness, and version compatibility checking at startup.

Request: no fields.

Response:

| tag | field | type | |
|---|---|---|---|
| 2 | daemon_version | string | skippable |
| 4 | uptime_seconds | u64 | skippable |

Nothing is critical: the header version is the substance of the exchange.

### 5.2 AUTHENTICATE — action `0x02`

The primary operation.

Request:

| tag | field | type | | source | trust |
|---|---|---|---|---|---|
| 1 | user | string | critical | client application | **claim — must be verified** |
| 3 | service | string | critical | host configuration | trusted |
| 5 | authtok | bytes | critical | end user | the subject of validation |
| 2 | rhost | string | | host observation | trusted |
| 4 | ruser | string | | host application | claim |
| 6 | tty | string | | host application | claim |
| 8 | pid | u32 | | client's own `getpid()` | self-reported; logging only |

`user` is the identity the connecting party is *requesting*. In the PostgreSQL
case it comes straight from the client's startup packet and is entirely under
the remote party's control. A daemon MUST bind it to the credential and MUST
NOT treat a valid credential alone as sufficient.

`service` identifies which policy applies. In the PostgreSQL case it derives
from the `pamservice=` option on the matched `pg_hba.conf` line and is not
influenced by the remote party, so it is safe to route on.

`pid` is for log correlation only — it lets a daemon's log join to a
PostgreSQL log carrying `%p`. It is self-reported and MUST NOT inform any
decision.

Where a shared PID namespace makes `SO_PEERCRED`'s pid meaningful, a daemon MAY
compare the two, but MUST NOT reject on mismatch by default — forging this
field requires access that already permits forging the whole request, so it is
not a security boundary. An opt-in strict mode is permitted, documented as a
correctness check rather than a defence.

Daemons SHOULD log the claimed and attested pids as separate fields rather than
substituting one for the other, so the logged value does not mean different
things in different deployments.

Response:

| tag | field | type | |
|---|---|---|---|
| 1 | authn_result | u32 (§6) | critical |
| 3 | authz_result | u32 (§6) | critical |
| 2 | authn_id | string | skippable |
| 4 | error_detail | string | skippable |

Two verdicts are returned from one round trip. `authn_result` answers "is this
credential valid"; `authz_result` answers "may this identity act as `user`".
PAM requires both — `pam_authenticate()` then `pam_acct_mgmt()` — and the
client is expected to cache `authz_result` between the two rather than make a
second request.

`authn_id` is the canonical identity the daemon resolved, for example
`system:serviceaccount:app:web`. A client that has somewhere to put it (the
PostgreSQL validator API's `authn_id`, feeding `system_user`) uses it; a client
that does not (PAM, which has no channel for it) ignores it.

`error_detail` is for the daemon's log and the host's log. It MUST NOT be
returned to the remote party.

### 5.3 STATUS — action `0x03`

Health, for readiness probes and diagnostics.

**STATUS carries no credential.** It is gated only by the socket's own access
controls, so anything permitted to query health is also permitted to
authenticate. That is deliberate: its contents are low-sensitivity — health,
counters, and configuration deliberately exposed for diagnostics — and in the
intended deployments no consumer needs one without the other. Probes run inside
the daemon's own container at its own uid, and metrics scraping uses a separate
HTTP endpoint.

Request:

| tag | field | type | |
|---|---|---|---|
| 2 | check_upstream | u8 | skippable |
| 4 | service | string | skippable |

`check_upstream` absent or zero means report cached health. Nonzero requests a
live upstream check; daemons SHOULD rate-limit this.

`service` selects which policy to describe. A daemon may serve several
services with different configuration — different expected audiences, different
mappings — so there is generally no single answer. Absent, the daemon reports
whatever is global to it and omits per-policy detail. Present and unrecognised,
the daemon MUST respond `UNSUPPORTED` rather than silently reporting a
different policy's configuration.

Response:

| tag | field | type | |
|---|---|---|---|
| 1 | serving | u8 | critical |
| 2 | upstream_ok | u8 | |
| 4 | upstream_last_success | u64 | |
| 6 | upstream_last_error | string | |
| 8 | cache_entries | u64 | |
| 10 | cache_hits | u64 | |
| 12 | cache_misses | u64 | |
| 14 | daemon_version | string | |
| 16 | detail | detail-map (§5.3.1) | |

`serving` is the readiness signal: can this daemon authenticate a *new*
principal right now. It is not simply `upstream_ok` — a daemon whose cache can
still cover a brief upstream outage remains serving.

Everything backend-specific goes in `detail` rather than being given a tag.
The protocol must not acquire fields that only one daemon can populate.

#### 5.3.1 detail-map encoding

```
detail-map := entry*
entry      := u16 keylen, u8 key[keylen], u32 vallen, u8 value[vallen]
```

Keys and values are UTF-8. Entries MUST be sorted by key, ascending
bytewise, and each key MUST appear at most once — the same determinism rule as
top-level fields, for the same reason.

Keys are conventional, not specified. `ludsa-k8s` is expected to report at
least:

| key | why |
|---|---|
| `identity` | which principal the upstream believes the daemon is; catches an RBAC binding on the wrong service account |
| `audience` | the expected audience for the requested `service` |
| `ca_source` | `in-cluster`, `kubeconfig`, or an explicit bundle path |
| `ca_not_after` | earliest expiry in the trust bundle |
| `ca_fingerprint` | SHA-256 of the bundle, to confirm which CA is loaded |
| `pid_namespace` | `isolated` or `shared`; explains why peer pid is unusable, and flags a weakening of the isolation the deployment assumes |
| `jwks_last_refresh` | when the signing keys were last successfully fetched; a daemon that can no longer refresh keeps working until the cluster rotates them, and then stops entirely |

`audience` is reported deliberately despite being configuration: it is the
setting whose misconfiguration fails *open*, so reading it back from a running
daemon is worth more than the disclosure costs. It is not secret, and only
permitted uids can reach the socket.

### 5.4 AUTHORIZE — action `0x04`, reserved

For PAM stacks where another module handled `auth` and a LUDSA client serves
only `account`. Not required by PostgreSQL, which always calls both on one
handle. Reserved, unspecified in v1.

## 6. Result codes

| code | name | meaning | PAM mapping |
|---|---|---|---|
| 0 | `OK` | permitted | `PAM_SUCCESS` |
| 1 | `DENIED_CREDENTIAL` | credential invalid, expired, or wrong audience | `PAM_AUTH_ERR` |
| 2 | `DENIED_POLICY` | credential valid; identity may not act as `user` | `PAM_PERM_DENIED` |
| 3 | `USER_UNKNOWN` | no mapping exists for the resolved identity | `PAM_USER_UNKNOWN` |
| 4 | `UNAVAILABLE` | daemon could not determine an answer | `PAM_AUTHINFO_UNAVAIL` |
| 5 | `MALFORMED` | protocol violation | `PAM_SYSTEM_ERR` |
| 6 | `UNSUPPORTED` | action not implemented | `PAM_SYSTEM_ERR` |

An enum rather than a boolean because the distinctions are load-bearing:
`UNAVAILABLE` versus `DENIED_CREDENTIAL` changes stack semantics for a generic
PAM module, and `DENIED_CREDENTIAL` versus `DENIED_POLICY` separates "this
token is bad" from "this token is fine but that identity may not be that role".

When `authn_result` is not `OK`, `authz_result` SHOULD also be non-`OK`.

## 7. Conformance

A conforming daemon MUST reject, with `MALFORMED` where a response is possible:

1. A version it does not implement.
2. A declared length exceeding `MAX_MESSAGE_SIZE`.
3. A field length extending beyond the declared message length.
4. A duplicate tag.
5. Tags out of ascending order.
6. An unrecognised **odd** tag.
7. A missing critical field for the action.
8. A message not terminated by `TAG_END`.
9. Trailing octets after `TAG_END` within the declared length.
10. Octets beyond the declared length, where observed by the non-blocking
    check of §2.

It MUST skip, without error, an unrecognised **even** tag.

`ludsa-conformance` exists to send each of these. Test vectors in `testdata/`
are hand-authored from this document and MUST NOT be generated by dumping an
implementation's output — otherwise the implementation, and its bugs, become
the specification.

## 8. Non-goals

- Confidentiality or integrity on the wire. The socket's permissions are the
  security boundary; a peer able to write to it can already assert any verdict.
- Authentication of the daemon to the client, beyond filesystem permissions.
- Streaming, pipelining, or multiplexing.
- Carrying a PAM conversation. LUDSA cannot drive prompts, so it cannot support
  interactive flows. This is deliberate: the target is non-interactive
  credential presentation.

## 9. Open

Two, both additive and neither blocking. Deciding either later costs nothing,
which is why neither holds up freezing version 1.

**Whether `AUTHORIZE` (§5.4) is ever specified, or stays reserved forever.** A
client invoked only in PAM's `account` phase has never seen a credential, so it
could answer on `user`, `service` and `rhost` alone — which for a
credential-validating backend means nothing at all. Settled by the appearance of
a backend whose authorisation decision needs no credential. Until then,
reserved.

**Whether `error_detail` (§5.2) warrants a machine-readable companion code**
beyond the coarse results of §6. A daemon already classifies its own failure
reasons for its own metrics, so this buys nothing daemon-side; what it would add
is letting the *client* classify, and so alert from the host application's logs
rather than the daemon's. Settled by someone wanting that.
