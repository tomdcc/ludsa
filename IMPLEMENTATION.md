# LUDSA implementation guidance

How to satisfy the rules in `SPEC.md`. Nothing here is normative. An
implementation that meets the specification by other means conforms, and where
this document and the specification disagree, the specification wins.

`RATIONALE.md` covers why the rules are what they are. This document covers the
strategies that satisfy them, and the traps in each. Several of the traps below
are specific to Go, because the reference server is written in it and its
standard library makes some of these rules awkward to express correctly.

## Deadlines

The specification sets no values. These are a coherent starting set, and all of
them should be configurable.

The transport is local and messages are bounded at 64KB, so a healthy exchange
completes in single-digit milliseconds. PostgreSQL's `authentication_timeout`
defaults to 60 seconds, which is the ceiling a client's overall deadline has to
sit inside. That suggests a client overall deadline in the high hundreds of
milliseconds, of which the connect retry budget is a few hundred, and server
deadlines of a second or two.

**Go: a fired deadline latches.** `net.Conn.SetDeadline` sets read *and* write,
so the expiry that ends a read also kills the response write. Worse, even
`SetReadDeadline` alone is not enough to recover: once a deadline has passed,
every subsequent operation in that direction fails with `os.ErrDeadlineExceeded`
until the deadline is reset. The handler that reads

```go
if errors.Is(err, os.ErrDeadlineExceeded) { writeStatus(conn, REQUEST_TIMEOUT) }
```

sends nothing, and the server believes it conforms while never once emitting a
`REQUEST_TIMEOUT` frame. Call `SetWriteDeadline` afresh after the read deadline
has fired and before writing, which is what §2's separate response deadline is
asking for anyway.

The fault is invisible under `net.Pipe`, which has its own deadline
implementation, and invisible in any test where the deadline never fires.

**Keep the request deadline and the dependency deadline apart.** They are
sequential phases of one exchange and must not share a `context`. One
`context.WithTimeout` threaded through the socket read and the backend call
reports a slow dependency as `REQUEST_TIMEOUT`, which §5 defines as the request
not having arrived. The request deadline ends when the request is complete, and
the dependency call takes a fresh one whose expiry is `UNAVAILABLE`.

Both map to `PAM_AUTHINFO_UNAVAIL`, so the outcome is identical and only the
diagnosis differs. That is why the mistake survives testing, and why it costs an
operator a search of infrastructure that is working.

## Reading a request

**Go: `io.ReadAll` reports a truncated request as a complete one.**
`io.ReadAll(io.LimitReader(conn, n))` returns `(short, nil)` at EOF, because it
treats `io.EOF` as normal termination. §7 case 15 then goes silently
unimplemented and the server parses a truncated buffer.

`io.ReadFull` is the call that distinguishes them: it returns
`io.ErrUnexpectedEOF` for a partial read and `io.EOF` for a zero-byte one.

**Which of those means silence depends on where you are.** §2 starts the
obligation to answer at `ludsa_version_major`, so:

```
either error reading ludsa_version_major  no version to answer in, close quietly
io.EOF from any later read                REQUEST_INCOMPLETE (§7 case 15)
io.ErrUnexpectedEOF from any later read   REQUEST_INCOMPLETE
```

**Read the version word on its own, not the whole envelope.** A server that
takes the envelope in one 8-octet `io.ReadFull` gets `io.ErrUnexpectedEOF` from
a one-octet request. The table above would answer it. §2 says the opposite:
below two octets there is no version to answer in, so the server closes without
replying. Splitting the first read at two octets puts the threshold where §2
puts it.

The client's threshold is one octet, not two, and §7 case 24 pins it. A client
reports `RESPONSE_INCOMPLETE` for the same truncation that leaves a server
silent, because a client is reporting rather than answering.

A client that writes exactly the 8-octet envelope declaring `length = 100` and
then half-closes gives `io.EOF` from the *body* read. Treating that as silence
closes on a natural case-15 vector, which is the case this section exists to
stop going unimplemented.

**A single `Read` does not fill the buffer.** `conn.Read(buf)` returns whatever
has arrived, with no error and nothing to say the buffer is short. Every vector
in `testdata` is a few hundred octets and arrives whole, so a server built on
one `Read` passes the entire suite and fails on the first production JWT, where
a 32KB `credential` spans several segments. `io.ReadFull` is the loop, and the
envelope needs it as much as the body does.

Writes need no equivalent. `net.Conn.Write` returns an error if it writes fewer
octets than it was given, so it loops internally.

**Never cast into the buffer to read an integer.** `msg_type` sits at offset 0
of a message, a tag at 4 and its `len` at 6, so every `u32` after the first is
misaligned. A value of arbitrary length leaves every later field header wherever
it falls.

`*(uint32_t *)p` is undefined behaviour whatever the hardware tolerates. It also
passes on x86, and on a big-endian host, in exactly the wrong way. §3's byte
order already requires assembling from octets, which is alignment-independent by
construction:

```c
uint32_t v = ((uint32_t)p[0] << 24) | ((uint32_t)p[1] << 16)
           | ((uint32_t)p[2] <<  8) |  (uint32_t)p[3];
```

`binary.BigEndian.Uint32` is the equivalent in Go, and is safe at any offset.

**Go cannot tell a half-close from a close.** `(*net.UnixConn).Read` returns
`io.EOF` identically for a peer's `shutdown(SHUT_WR)` and its `close()`. There
is no notification and no affordance. §2 requires the server to proceed to write
regardless and find out from `EPIPE`, so any handler shaped around "read
returned EOF, the client is gone, return early" is wrong, and it reads as
reasonable code.

## Sizing a client's buffer

Appendix A gives the per-message figures. No version 1.0 response exceeds 534
octets, and `pam_ludsa`, which receives only an `AUTHENTICATE_RESPONSE` or an
`ERROR`, tops out at 527. `MAX_LENGTH` is 32768 and no client needs a buffer
that size.

There are two strategies:

| strategy | buffer | version skew |
|---|---|---|
| parse incrementally, discard what you do not recognise | 534, fixed | absorbs any minor version, no reconfiguration |
| read the declared length whole, then parse | `MAX_LENGTH`, 32768 | also correct, 61 times larger |

**The fixed buffer is permanently correct, not correct for now.** That is worth
spelling out, because the obvious worry is that a later minor version makes 534
too small, and the obvious fixes for it are all unnecessary. Four rules combine:

- §4.1 freezes field maxima, so `authn_id` can never exceed 512 in major
  version 1. Nothing a client already stores can grow.
- A minor version can therefore only add fields, and §4.1 makes an added field
  optional.
- §3.3 makes an optional field skippable, using bit 14 and, when clear, bits
  13-12 to know how many octets to skip. A receiver skips it knowing nothing
  else about it.
- §3 says a receiver need not retain the whole declared length, provided it
  reads all of it and interprets nothing beyond.

So the bytes a client retains are bounded by its own compiled-in schema, for
every version it will ever meet. The thing that grows is the thing it throws
away. A client needs no configuration knob for the buffer size, no fallback to
dynamic allocation, and no recompile against a newer peer.

The framing supports this directly. At every point you hold a tag, and for a
dynamic field a `u32 len`, so you know whether to copy the next run of octets
into your struct or discard it. Tag order is one comparison against the previous
tag, and required-field presence is a bitmask. No rule in §3 or §4 needs the
message resident.

**A small buffer does not license rejecting a larger message.** §3 requires
accepting any declared length up to `MAX_LENGTH`, so a client sized at 527 has
to cope with a response longer than that rather than answering
`REQUEST_TOO_LARGE`. Read and discard what the buffer does not hold, and do not
interpret anything past the declared length. §3's constraint is on what a
receiver interprets, not on holding a message whole.

`MAX_LENGTH` still bounds the discard path, so a client must read a
32768-octet declared length to completion before it can report anything. That
costs wall time rather than memory, and the scratch it discards through can be a
few hundred octets reused in a loop.

**The extent check overflows before the allocation does.** §3 forbids
allocating from an unchecked length, and the check that guards it is the earlier
hazard:

```go
if off+int(l) > len(buf) { return ErrMalformed }   // wrong
```

With `l` a `uint32` near 2^32 and `int` 32 bits wide, `off+int(l)` goes
negative, the check passes, and the slice expression panics instead. That panic
is recovered and reported as `INTERNAL_ERROR`, on an input that is plainly
`REQUEST_MALFORMED`. Compare without the addition:

```go
if l > uint32(len(buf)-off) { return ErrMalformed }   // right
```

## Draining a rejected message

§3 requires a server that rejects before reading the whole message to send its
response, then read and discard until the earliest of four conditions. It is the
one path where a server reads from a connection it has already given up on, and
the obvious Go call and the obvious hand-rolled loop behave differently at EOF.

`io.CopyN(io.Discard, conn, 2*MaxLength)` returns `io.EOF` when the peer has
half-closed, which is the normal case under §2, and that is the condition that
ends the drain rather than an error. A hand-rolled `for { n, err := conn.Read(b)
... }` that does not treat `n == 0 && err == io.EOF` as terminal spins at full
CPU until the deadline, which is a remote unauthenticated CPU burn reachable
without authenticating.

The response goes out before the drain starts, so a client blocked in `read`
gets its answer without waiting for the drain to finish.

**What the drain is for.** It keeps a client that is still writing a large
`credential` from taking `EPIPE` before it gets round to reading. On `AF_UNIX` a
response already written sits in the peer's receive queue and survives the
server's `close()`, so the drain is not protecting the response itself; there is
no reset to discard it, which is a TCP behaviour and §2 forbids TCP.

The status is the one determined when the message was rejected. A drain that
runs to the deadline still answers `REQUEST_TOO_LARGE`, not `REQUEST_TIMEOUT`.

## Writing to the socket

**Suppress `SIGPIPE` per write, and leave the host's disposition alone.**
Writing to a socket whose peer has closed raises `SIGPIPE`, whose default action
terminates the process. A client is a guest inside `sshd` or a PostgreSQL
backend, so the default action is the host dying in the middle of an
authentication.

The two halves of the rule fail differently, and only the first is obvious:

- **Missing suppression** passes every PostgreSQL test, because PostgreSQL
  already sets `SIGPIPE` to `SIG_IGN`, and then kills an `sshd`.
- **`signal(SIGPIPE, SIG_IGN)` at library init** fixes the crash by taking
  global state a library must not hold, and changes the behaviour of every other
  write in the host process.

`MSG_NOSIGNAL` on `send()` covers Linux, which is effectively all non-testing
use. `SO_NOSIGPIPE` as a socket option covers macOS and the BSDs, where only
`libludsa` is built, since `pam_ludsa` needs Linux-PAM and cannot be built there
at all. Between them every port under consideration is covered, so the portable
route, blocking `SIGPIPE` with `pthread_sigmask` and consuming a pending one
with `sigtimedwait` after a failed write, is not needed here. It is what someone
reaches for when aiming at a portability this project does not want.

**A short write and an `EINTR` are resumable.** Both are routine inside a host
that installs handlers without `SA_RESTART`, which is the host's prerogative and
not something a guest library may change. Resuming on the same connection is not
the retry §2 forbids; that rule is about opening a new connection. A client that
treats either as fatal fails an authentication because an unrelated child
exited.

**Read the response even when the write failed.** §3 has a server that rejects a
message early send its response and then discard up to `2 x MAX_LENGTH` octets
before closing, so a client whose write returned `EPIPE` may still have a
`REQUEST_TOO_LARGE` waiting for it. Reporting the transport error
without attempting the read throws away the only explanation of it.

## Detecting octets beyond the declared length

§2 requires rejecting an exchange in which excess octets are observed, and says
detection need not be exhaustive. The reason for that allowance is that the
obvious implementation deadlocks: reading again and requiring EOF blocks against
a client that omitted its half-close, since the client waits for the response
while the server waits for EOF. A non-blocking read avoids that, rejecting when
it finds data and proceeding when it does not.

**Go has no non-blocking read, and the two obvious substitutes both fail.**

`conn.SetReadDeadline(time.Now())` followed by `conn.Read` checks the deadline
*before* attempting the syscall, so it returns `ErrDeadlineExceeded` without
ever touching the socket. The check therefore reports "nothing pending"
unconditionally, including when the socket is full of excess octets, and it
latches the read direction as above.

`conn.SyscallConn()` with `RawConn.Read` does reach a genuinely non-blocking
descriptor, but the callback's return value inverts the intuition: returning
`false` means "not ready, park me on the poller", which blocks. The callback
must always return `true`, leaving `EAGAIN` from the raw `syscall.Read` to be
interpreted by its caller.

**And a buffered reader hides them completely.** If the request was read through
a `bufio.Reader`, the excess octets are already in userspace, where no
descriptor-level check can see them. The real check is `br.Buffered() > 0`, and
a correct server needs both that and the descriptor-level one.

Because §2 makes the check best-effort, a server that never detects anything
still conforms, which is exactly why a broken check stays undiscovered.

**A client needs none of this.** §2 has the server close after it responds, so a
client checks for excess octets by reading once more and requiring end of file.
The deadlock the machinery above works around is the server's problem, and it
exists because a client that omits its half-close leaves the server nothing to
wait for. Reading to EOF on the client side is safe only because the server's
close is required; without that rule it would hang until the client's overall
deadline.

## Creating and publishing the socket

§2 states a visibility property: the socket must never be observable at its
published path with the wrong mode or group. Two routes satisfy it.

**Setgid directory and umask.** Set the umask to `0117` before `bind()`, so the
socket is created `0660` outright, and let the directory's setgid bit supply the
group. The socket is created with the right mode and group, so there is no
`chmod`, no `chgrp`, no `rename`, no window, and no privilege needed beyond
group membership. It needs a directory the deployment has prepared, which the
section below describes.

**Temporary name, then publish.** `bind()` to a temporary name in the same
directory, `chgrp`, `chmod`, then `rename()` into place. `rename()` is atomic
within a filesystem and a bound socket follows its inode, so the socket never
appears at its published path in a wrong state. This works with no special
directory, at the cost of the server needing to be in the target group or hold
`CAP_CHOWN`.

Verify either on the target platform before relying on it. Umask behaviour on
socket inodes is not uniform.

**Go: a few problems with the second route.**

`net.Listen("unix", path)` exposes no mode option and binds using the process
umask. `syscall.Umask` is process-wide, not per-goroutine, so the window in
which it is changed applies to every other goroutine touching the filesystem,
including log rotation at startup.

`net.Listen` also sets `unlinkOnClose`, and `Close()` unlinks *the address it
bound*, the temporary name. After the rename, a clean shutdown unlinks a path
that no longer exists and leaves the published socket behind, so every ordinary
restart looks like a crash. Calling
`(*net.UnixListener).SetUnlinkOnClose(false)` and unlinking the final path
explicitly avoids that.

And reaching the descriptor to `fchmod` it is a trap of its own:
`(*net.UnixListener).File()` dups the descriptor and puts the *original* into
blocking mode, which disables the listener's deadline handling for the rest of
the process's life. `SyscallConn()` is the call that does not.

**Check the mode as a set of bits, and mask the file type off first.** §2's
bound is any of `07777` set that `0660` does not:

```c
(st.st_mode & 07777) & ~0660        /* C */
```
```go
sys := fi.Sys().(*syscall.Stat_t)
sys.Mode & 07777 &^ 0o660           // Go
```

Two traps, and the second is worse. `mode > 0660` admits `0606`, since `0o606`
is 390 and `0o660` is 432, and that socket is world-writable and inaccessible to
its own group. But `st.st_mode & ~0660` is non-zero for *every* socket including
a correct one, because `st_mode` carries the file type and `S_IFSOCK` is
`0140000`. Go's `os.FileMode` carries `ModeSocket` the same way, and its
`Perm()` masks to `0777`, which drops the setgid bit the setgid-directory route
above can leave on the socket. A server that omits the mask refuses to start on
a socket it has just created correctly.

## Detecting a stale socket

§2 requires unlinking a stale socket before binding, and never unlinking a path
that is not a socket. It does not say how to tell stale from live, because that
is a detection strategy rather than an observable property.

- **Unconditional**, after the is-a-socket check. What nearly everyone writes,
  and the only thing that works after a `SIGKILL`. Its cost is that a second
  instance started by mistake silently takes the path: the first keeps its bound
  inode and keeps accepting on a socket no client can reach, so authentication
  fails against a process that looks healthy and logs nothing.
- **`connect()` probe.** Cheap, and distinguishes the common cases. A
  wedged-but-live server still accepts, so it refuses to start exactly when
  taking over would have been right.
- **Lock file.** The robust answer, at the cost of a second path to configure.

## The socket's directory

§2 states no rule about the directory, because no decidable one exists. Root
writes every directory, so a literal test excludes the layout the design
assumes; holders of `CAP_DAC_OVERRIDE` sit the same way; testing only the
immediate directory misses an ancestor that can be renamed; and testing every
ancestor asserts a property the server confirms once and cannot hold.

The hazard is real all the same. A user who can write to the directory unlinks
the socket, binds their own at the same path, and receives the credential the
next client sends. So the deployment carries it, and two layouts satisfy it by
construction:

- **systemd `RuntimeDirectory=`.** The unit creates `/run/ludsa` owned by the
  service user, beneath `/run` at `0755 root:root` and `/` at `0755 root:root`.
  Nothing in the chain is writable by anyone else, and nothing has to be checked
  at run time.
- **A pod's `emptyDir`.** Only the pod's own containers reach it, so the
  question reduces to which of them mount it writable.

Search permission on the directory is not a concern. A process that can traverse
it but is not in the socket's group can `stat()` the socket and nothing more,
because `connect()` on a unix socket requires write permission on the socket
inode.

The sticky bit is what makes a shared directory such as `/tmp` survivable, since
it stops another user unlinking an existing socket. It does not close the window
the server opens itself: it unlinks its own stale socket before binding, and an
attacker with write permission can bind at the free path during that window.
Prefer a directory the server owns.

A server that inspects the directory anyway is doing something useful, and Go
makes it awkward. The check needs the directory's uid, gid and mode, which means
`os.Stat` and then an unchecked `fi.Sys().(*syscall.Stat_t)` assertion that
panics wherever `Sys()` returns something else, and needs build tags to compile
off Unix at all. There is no `bindat`, and `net.Listen` takes a path and nothing
else, so there is no way to bind relative to a directory descriptor already
inspected. The check-then-bind window cannot be closed, only re-checked after.

## The accept loop

**Go: the loop everyone writes fails §2 three ways.**

```go
for { c, err := l.Accept(); if err != nil { continue }; go handle(c) }
```

It bounds nothing. On `EMFILE` or `ENFILE` it busy-spins at full CPU, which is a
self-sustaining outage: the server is pinned at 100% precisely when it has no
descriptors left to authenticate with. `net/http`'s `Server.Serve` carries an
explicit backoff for this and a hand-rolled loop inherits nothing;
`net.Error.Temporary()` is deprecated and returns nothing useful, so the test is
`errors.Is(err, syscall.EMFILE)` against the unwrapped `*net.OpError`.

And the obvious bound, a buffered-channel semaphore acquired *after* `Accept()`
returns, starts §2's accept-to-complete-request deadline ticking while the
connection waits in the queue, so overload is reported to the client as
`REQUEST_TIMEOUT`, which §5 defines as the request not arriving. Acquire before
`Accept()` instead, which leaves connections in the kernel backlog, or answer
`RATE_LIMITED` deliberately.

**Every connection goroutine recovers.** An unrecovered panic ends the process,
not the goroutine, so a decoder bug turns any of §7's malformed cases into a
remote unauthenticated crash. Nothing else produces `INTERNAL_ERROR` either: a
server with no `recover` defines a status it can never send.

The recovery path has to know whether anything has gone out. If no response
octet has been written it sends `INTERNAL_ERROR`; if some has, appending to it
produces a malformed response, so it closes instead.

`recover` is not the whole defence, and the opposite-direction hazard is worth
holding alongside it. `make([]byte, n)` on an unchecked `n` does not panic, it
allocates, and a runtime out-of-memory fatal error cannot be recovered at all.
§3's rule against allocating from an unchecked length is what covers that one.

## Validation order

§4 puts schema-independent framing at step 4 and `msg_type` at step 5, and the
line between them is finer than it looks. Tag shape, ascending order, field
extents and `TAG_END` hold without a schema. Whether a tag may repeat does not:
only the message type says, and step 7 is where that is decided.

So a message with both a repeated tag and an unimplemented `msg_type` is
`REQUEST_UNSUPPORTED`, and one with a repeated tag in a type the server does
implement is `REQUEST_MALFORMED`. §7 cases 22 and 5 are the pair that pin it.

Run the schema-independent pass over the whole message first, then dispatch,
then apply the schema. A server that dispatches on `msg_type` before checking
tag shape or extents answers `REQUEST_UNSUPPORTED` to a message that is
structurally broken, which case 23 catches through its own combination.

## Emitting fields

§3.2 requires ascending tag order, and at most one of each tag in every version
1.0 message type.

**Go: map iteration is randomised.** An encoder that collects fields into a
`map[uint16][]byte` and ranges over it emits them in an order that changes
between runs of the same binary on the same input. A byte-exact vector
comparison then fails intermittently rather than consistently, which reads as
flakiness rather than a conformance bug, and a two-field response passes half
the time. Sort the tags, or build a fixed-order slice.

**The same warning applies on decode, for the opposite reason.** A map catches
a repeated tag for free, which is what makes it tempting, and it discards the
ordering §7 case 6 requires. The check has to happen while scanning, comparing
each tag against the one before, because a map built first cannot be validated
afterwards.

One comparison still catches both, but the two answers arrive at different
steps. `<` fails ordering at step 4 and is `REQUEST_MALFORMED` whatever the
message type. `==` is a repeat, which step 7 judges against the schema, so a
scanner should record it and let the dispatch decide rather than reject on the
spot. For every version 1.0 type the answer is the same; for a message type the
server does not implement it is `REQUEST_UNSUPPORTED`, and rejecting early gets
that wrong.

After the scan the representation is free. Ordering is a property of the
encoding and not of the decoded message, so handing an application a map is
fine.

## Decoding into a struct

**Absent and empty are different, and a struct field cannot hold both.** §3.2
makes a value of zero octets a value rather than an absent field, so `""` from a
decoder means one thing for a required field and two for an optional one.

For a **required** field, have the decoder reject a missing one as
`REQUEST_MALFORMED` before it returns. Application code then never meets the
ambiguity, and `req.User == ""` correctly means present and empty, which §3.2
leaves the application to judge.

For an **optional** field, absence is data and the type has to carry it.
`max_age` is the case that bites, and it is worse than the string cases. The
two readings are opposite instructions: absent means the server may answer from
data of any age, and zero means it MUST check before answering. So
`req.MaxAge == 0` turns "any staleness is acceptable" into "check right now",
which surfaces as load rather than as a wrong answer.

`user` is the other case, and the dangerous one. Absent asks the server not to
decide; present and empty asks it to decide about the empty-named user, which
comes back as one of §6.2.1's denials. `if req.User == ""`
sends the second down the first's path, so the server skips its binding check
and answers `OK` on any valid credential, while the client that sent the empty
`user` believes it was checked. Nothing on the wire distinguishes that from a
correct answer.

**A field absent from the wire decodes as the affirmative value.** §5 has an
`ERROR` message carry no field but `status` and `error_detail`, so a `result`
decoded from one was never sent, and its zero value is `OK`, meaning permitted.
The same applies to `serving`, where a readiness probe reads `SERVING` off a
response that carried no such field.

§3 makes establishing `msg_type` before reading any field a MUST for a client,
and it is what keeps this shut. Dispatch on it into a per-type struct rather
than filling one shared struct and branching afterwards. A shared struct is what
makes the hazard reachable, because the field exists in it whether or not the
message carried one.

§6.4 numbers the response fields so that no tag names two different fields, and
a decode-then-dispatch client is what that numbering protects. Such a client
reading an `ERROR` message as an `AUTHENTICATE_RESPONSE` finds `0x8101`, which
is not a tag `AUTHENTICATE_RESPONSE` defines, so it skips an optional field and
comes out with no `result` at all. §6.2 then denies for want of one.

Had `status` and `result` shared a tag, the same client would have read a
protocol rejection as a decision, and value 0 is where that bites: §5 leaves 0
unassigned in `status`, so §3.4 requires a receiver to treat it as
`STATUS_UNKNOWN`,
while read as a `result` it is `OK`. Unlike the absent-field hazard above, that
survives a decoder which correctly requires the field to be present.

None of which licenses skipping the `msg_type` check. §3 makes it a MUST, the
numbering is a second line rather than the first, and `0x4302` is still
`server_desc` in two response groups, so a client that dispatches on tag alone
can still take a PING's release string for a STATUS one.

## Carrying the credential

§6.2 forbids normalising `credential` on either side: no trimming, no stripping
a trailing newline, no case folding. It is the rule in this protocol with the
worst failure mode, because it fails silently and self-consistently. An
implementation that trims on both sides matches everything it stores and passes
every test it owns, and only the conformance matrix, run against another
implementation, shows anything wrong.

**Keep it as `[]byte` from the socket to the backend.** `strings.TrimSpace` on a
value that arrived as text is the reflex, and it is invisible in a round-trip
test because the same code trims both directions.

**`ludsa-file` is where this bites twice.** `bufio.Scanner` strips the line
terminator from the *stored* credential, and `strings.Fields` or `TrimSpace`
strips it from the *parsed* one. Such a server agrees with itself completely and
disagrees with `ludsa-k8s` on any credential carrying trailing whitespace. Make
the file length-prefixed or explicitly escaped rather than whitespace-delimited,
so the storage side cannot lose an octet either.

**The asymmetric case is the dangerous one.** A client that trims and a server
that does not is a parser differential in an authentication path, because a
signature covers exact bytes. It is also the case a single implementation's
tests cannot reach.

§6.2's SHOULD about detecting leading or trailing whitespace is the diagnostic,
not the fix. The two are easy to conflate: a server that reports the condition
and then trims has satisfied the SHOULD and broken the MUST.

## Populating `error_detail`

§5 requires `error_detail` to carry no control character, meaning U+0000 to
U+001F, U+007F and U+0080 to U+009F, which is what "safe to log verbatim"
amounts to once `utf8` has supplied valid encoding and no NUL. §3.1 pushes a
server toward naming the offending value when it rejects a field for length.
Those two rules meet awkwardly, because §3.3 forbids the server from rejecting
control characters or invalid UTF-8 on input in the first place.

**Go: `fmt` will pass the bytes straight through.** `%s` and `%v` on a `string`
copy it untouched, with no escaping, no validation and no error. So the most
ordinary line in the file violates a MUST:

```go
fmt.Sprintf("user %s exceeds 256 octets", user)   // wrong
fmt.Sprintf("user %q exceeds 256 octets", user)   // right
```

The example names `user` deliberately. The same line with `credential`
substituted writes a bearer token into `pam_syslog`, and out through every log
shipper behind it. §5 forbids it, and no test catches it, because the
conformance suite compares the encoding of `error_detail` and never its content.

`%q` renders invalid UTF-8 as `\xNN`, and `utf8.ValidString` is the check.

**`%q` alone overruns the field.** It expands each control octet to four
characters, so a 256-octet `user` of control characters renders to 1026 against
`error_detail`'s maximum of 512. §5 binds the server to fit, so render into a
budget rather than formatting and hoping: shorten the embedded text, keep the
prose that names the bound, and never let the result exceed the maximum. A
server that overruns emits a message its client treats as having no
`error_detail` at all (§3.1). The consequence of getting it wrong lands on the
*client*: a PAM module writes `error_detail` to `pam_syslog` on the strength of
§5's guarantee, so a newline the server passed through is a log-forging
primitive in the host's authentication log.

`server_desc` carries the same obligation, for the same reason: §6.1 makes it a
diagnostic, so it reaches a log by the same route.

**On the receiving side, check at the point of use and check code points.** §5
requires a receiver to establish that one of these values is valid UTF-8 and
free of control characters before it logs it, returns it, or acts on it, and to
treat a value that fails as absent. A receiver that ignores the field does
nothing, so a PAM module pays nothing on the success path.

The check is one forward pass, allocating nothing, and the control-character
test comes out of the same pass that validates the encoding. Do not scan octets
for `0x80` to `0x9F`:

```
U+0100 -> C4 80      a byte scan rejects this
U+4E2D -> E4 B8 AD   and this
```

Those are ordinary continuation octets. A byte scan discards every value with a
Latin Extended, Greek, Cyrillic or CJK character in it, and on §6.2's
`user`-absent path that silently removes the identity a client depends on.
`0x00` to `0x1F` and `0x7F` never appear as continuation octets, so the newline
check, which is the log-injection vector, is safe on raw bytes.

## Logging received values

§3.3 forbids rejecting control characters or invalid UTF-8 in `user`, `ruser`,
`tty` and `rhost`, and §6.2 marks `user` as entirely under the remote party's
control. The server is the side with a logger and the side receiving those
bytes, so this obligation lands on the server alone.

A `user` of `web\nlevel=info msg="authentication succeeded"` forges a log line.
Rejecting the input is not available, because a role name arrives from
PostgreSQL's startup packet as bytes and validating it rejects legal input, so
the fix belongs in the logger.

**Each log format has a hole the other does not**, which is why fixing one and
testing only that one leaves the other open:

- **Text** lets a client forge a line with a newline. JSON escapes it for free.
- **JSON** requires UTF-8, and Go's marshaller coerces invalid bytes to U+FFFD
  without erroring, so two distinct role names collapse into one logged string.
  Text has no such problem.

Escape control characters in the text handler. Check `utf8.ValidString` in the
JSON one and render anything invalid as base64 or hex in a distinctly named
field. The property to preserve is byte fidelity: the bytes must stay
recoverable even when their encoding is unknowable. Display correctness is not
achievable without an encoding nobody has, and byte loss is the half that can be
avoided.

## Rate-limiting STATUS

§6.3 lets a client set `max_age` to zero, which obliges the server to check its
dependency before answering. A readiness probe misconfigured that way asks for a
check every few seconds.

A server whose dependency is expensive should set a floor on how often it
checks, and respond `RATE_LIMITED` when a request would breach it. The
specification requires that refusal rather than a stale answer, so the client
learns its freshness requirement was not met instead of believing a reading that
predates whatever it was investigating.

Set the floor from what the dependency tolerates, and document it. A client that
wants only cheap health should omit `max_age` rather than send a large value,
since an absent `max_age` places no obligation on the server at all.

The budget that matters for §2's connection bound is `MAX_LENGTH` multiplied by
the concurrency limit, not one message. §3 fixes the first, so the concurrency
limit is the only one a server sets.

## Mapping to PAM

`SPEC.md`'s Appendix B carries the three mappings, and `pam_ludsa` uses them
unchanged. What follows is what the appendix leaves to a module.

**Why `REQUEST_INCOMPLETE` is not `PAM_SYSTEM_ERR`.** A peer that crashed
mid-write is a deployment problem, and reporting it as a protocol violation
sends someone hunting a codec bug that is not there. `VERSION_UNSUPPORTED` goes
the other way for the same reason: no retry resolves a major version mismatch,
and someone has to upgrade a package.

Two statuses straddle the code-versus-environment reading. `REQUEST_UNSUPPORTED`
covers a message type a server does not implement and a `service` it does not
recognise; the second is a configuration mismatch. `REQUEST_TOO_LARGE` covers an
oversized message and a field above its §3.1 maximum; a `credential` above 16384
octets may be a token a client passed through exactly as it was given. Both stay
`PAM_SYSTEM_ERR`, because a retry improves neither and both need someone to
look. Which one it was lives in `error_detail`.

**The `account` line is the appendix's one requirement, and `pam_ludsa` has to
document it.** An authorisation outcome returns `PAM_SUCCESS` from
`pam_sm_authenticate` by design, so a stack without an `account` line lets a
`AUTHN_DENIED_POLICY` admit the caller. The module cannot detect the omission at
run time, so the manual page carries it and says why.

The exposure depends on the server. `AUTHN_DENIED_POLICY` is one of the three
optional codes (§6.2.1), so a server that discloses nothing answers
`AUTHN_DENIED` throughout, which Appendix B maps to `PAM_AUTH_ERR` and which
needs no `account` line at all. A deployment cannot know which kind of server it
will meet, so the requirement stands either way.

**A defective `authn_id` now costs an authentication.** §5 has a client check
any of the three `utf8` values it uses. Any defect is `RESPONSE_MALFORMED`, and
Appendix B maps that to `PAM_SYSTEM_ERR`. `pam_ludsa` logs the resolved
identity, so it uses `authn_id` and checks it. A server emitting a bad one fails
every authentication, grants included. That is the intended reading: it is a
server bug and it should be loud.

A client that omits `user` and cannot read the credential itself needs a server
that returns `authn_id`, and §6.2 gives it no way to require one. Check that at
deployment rather than at run time: a client in that position which meets a
server that omits `authn_id` can only fail, and failing every authentication for
a configuration mismatch is worth catching before it is load-bearing.

`pam_ludsa` always sends `user`, because PAM always has one: it is the account
being authenticated, and there is no version of the module's job that does not
involve it. The optional path in §6.2 is for a client that holds a
self-identifying credential and its own policy, and it takes on the obligation
the server would otherwise carry. A client on that path that treats `OK` as
permission has moved the fail-open somewhere no server can see.

## Not leaking through timing

Two channels, and the second undoes the first if it is missed.

**Compare credentials with `crypto/subtle.ConstantTimeCompare`.** `ludsa-file`
is the one server here that holds the credential it checks against and answers
by comparing the two, so `bytes.Equal` leaks the length of the matching prefix
through response time. Compare the lengths first, since `ConstantTimeCompare`
returns 0 for unequal lengths and the length is not the secret. It is a
reference server rather than a production one, which is the reason to get it
right: it is what someone reads before writing their own.

**A uniform denial code is not uniform behaviour.** §6.2 has a server unwilling
to disclose answer `AUTHN_DENIED` for every denial, and a server that does that
while returning in a microsecond for an unknown user and thirty milliseconds for
a policy lookup has disclosed the same thing the code was withholding. A
production server may want to equalise the two, by deferring the answer to a
fixed floor or by doing the work either way.

How far to take that is a deployment question and this document does not
prescribe it. The floor costs latency on every denial, the work costs load, and
which matters depends on the traffic. What is worth stating is that choosing
`AUTHN_DENIED` for its discretion and then leaking through timing buys nothing.

## Peer credentials

The specification does not require either side to check the other, because
filesystem permissions are the access control the protocol relies on. Checking
anyway is worthwhile in most deployments, and §5 gives a server that does a
status to report it with.

**Where the check goes is now a choice rather than a constraint.** At accept, a
server has nothing to answer in and closes without replying, which costs it the
diagnosis and leaks nothing. After reading `ludsa_version_major` it can answer
`PEERCRED_DENIED`, which reaches the client's logs and Appendix B maps to
`PAM_SYSTEM_ERR`, sending an operator to the allowlist. Before the status
existed the first was the only conforming option, because §2 obliges a server to
answer once it holds a version and §5 had nothing that fit.

A server can read `SO_PEERCRED` and refuse a connection whose peer uid is not in
a configured allowlist, subject to these constraints:

- The peer pid from `SO_PEERCRED` is not usable. It is reported in the namespace
  of the process that asks, so it is generally 0 when the client and the server
  are in separate containers. The uid is usable wherever the two share a user
  namespace, which is the common case; where they do not, it is translated or
  comes back as the overflow uid, and it is no more dependable than the pid.
- The allowlist must be configurable, and a server must not compare the peer uid
  against its own by default. The client and the server usually run as different
  users, and a server that assumes otherwise works only when the two are
  co-located.

Where a shared PID namespace makes the `SO_PEERCRED` pid meaningful, a server
may compare it against the `pid` field of an AUTHENTICATE request. It should not
reject on a mismatch by default, since forging that field requires access that
already permits forging the whole request. An opt-in strict mode is reasonable,
documented as a correctness check rather than a defence.

A server that logs both should log the claimed pid and the attested pid as
separate fields, rather than substituting one for the other. Otherwise the
logged value means different things in different deployments.

**A client has more to lose than a server does, and §8 leaves it unaddressed.**
A socket an attacker planted receives the credential and then answers whatever
it likes, which is an authentication bypass rather than only a disclosure. The
protocol says nothing about it because no single check works everywhere. Two do
between them, and a client should be configurable for whichever its deployment
supports:

- **The peer's uid**, via `getpeereid` or `SO_PEERCRED`, compared against a
  configured value. Read it after `connect()` and *before* writing any part of
  the request; a check made after the credential is on the wire has already
  lost. Compare numerically rather than by name, because the server's uid need
  not resolve in the client's passwd database across a container boundary.
- **Ownership and permissions** of the socket and its directory, where the uid
  is not meaningful. Weaker, since it describes a path rather than the process
  actually connected, and it races.

libpq is the worked example, as `requirepeer` in `fe-connect.c`. It calls
`getpeereid` on the connected socket rather than stat-ing the path, so there is
no check-then-connect window. It sits immediately before the transition to
`PGRES_POLLING_WRITING`, so the check precedes the startup packet. Its
documentation names the threat directly: a socket in a publicly writable
location can be claimed by any user's server.

Its name comparison is the part not to copy. It resolves the peer uid with
`pg_fe_getusername` and compares the resulting name, which needs the server's
user to exist in the client's passwd database.

It fetches the gid and never uses it, which is right and worth stating. With
mode `0660` the socket's group is the set of permitted clients, so every
legitimate client is a member and could set that group on a socket of its own.
The group does not discriminate. The owner uid does.

## Library versioning

A library that implements this protocol has two version numbers, and they are
unrelated. The protocol version travels in the header of every message. The
library's own release version, a Go module semantic version for instance, states
source compatibility for programs that import it.

Neither can be derived from the other. Two releases of a library that differ in
major version can speak the same protocol version, and a library can gain
support for a new protocol version without changing its own major version.

Which library releases interoperate is a matter for release documentation. The
protocol version alone decides whether two running processes can talk, and §4 of
the specification makes it the only value either side may test.

## Deployment

**Put the client and the server in a shared group.** The socket is mode `0660`,
so group membership is what grants the client access.

**A supervisor may bind the socket** and hand a descriptor to the process that
serves. §1 makes the two together the server, so the socket obligations fall on
whichever of them binds. A server handed a descriptor performs none of the
startup checks, because the supervisor already did.

**Client and server in different containers of the same pod is a first-class
deployment.** They share only the volume that carries the socket: no filesystem
beyond that directory, no memory, and no process namespace. An implementation
that follows §2.1 works there without further accommodation, which is the point
of the rule. The same constraints apply to unrelated processes on one host under
different users, so nothing about this is specific to Kubernetes.

Startup order is not guaranteed by the protocol. A client that starts first sees
`ENOENT` and retries within its budget; a server that starts first waits.
