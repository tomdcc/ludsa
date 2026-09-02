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

The transport is local and messages are bounded at 32KB, so a healthy exchange
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

**The read direction latches too, and there the latch is right.** After the read
deadline fires, `io.CopyN(io.Discard, conn, ...)` returns
`os.ErrDeadlineExceeded` having read nothing, so the §3 drain never runs. Do not
reset the read deadline to make it run. §3 lists that deadline among the drain's
terminators, so wherever it has expired no drain was owed: the server writes its
status and closes. Resetting it lets the one client that already used its whole
deadline hold the connection for a second one.

Every other early rejection is settled while the deadline still runs, so the
drain there is a normal read and `io.CopyN` behaves. The tell is that a server
"fixing" the latch drains on exactly the exchange with least claim to it.

**A shorter deadline for the version word is for a refusal reached at accept and
for nothing else.** Setting one on an ordinary exchange, and answering in your
own major version when it fires, reads as a courtesy to a loaded server and is
the one shape that makes the latch wrong: the short deadline expires, the read
direction latches, and §2's request deadline is still running, so a drain owed
on a later rejection cannot run. A refusal reached at accept owes no drain (§3),
so §5's peer refusal and §2's concurrency refusal may each bound their wait, and
there the latch is harmless. Where you cannot afford to wait at all, answer in
your own major version at once or stop accepting, both of which §2 gives you.

**Keep the request deadline and the dependency deadline apart.** They are
sequential phases of one exchange and must not share a `context`. One
`context.WithTimeout` threaded through the socket read and the backend call
reports a slow dependency as `REQUEST_TIMEOUT`, which §5 defines as the request
not having arrived. The request deadline ends when the request is complete, and
the dependency call takes a fresh one whose expiry is `UNAVAILABLE`. On a
request that never completes because the server rejected it early, that deadline
does not end at all: §3 has it bound the drain.

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

**Either row is a complete answer for a reader shaped like this one.** A server
that reads the declared octets and then parses reports the truncation, and §4
lets it. One that validates as it reads may instead answer a fault it found in
the prefix, and §4 lets that too. Neither owes the other's answer, so there is
nothing to reconcile: do not add a parse of the prefix here to chase a status
the specification does not ask for.

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
a 16KB `credential` spans several segments. `io.ReadFull` is the loop, and the
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

That indistinguishability is also what §7's half-close case exploits from the
other side. A driver cannot see which one a client did either, so it writes the
response and lets the write report it. A client that used `close()` where §2
asks for `CloseWrite()` fails there and nowhere earlier.

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
too small, and the obvious fixes for it are all unnecessary. These rules
combine:

- §4.1 freezes field maxima, so `authn_id` can never exceed 512 in major
  version 1. Nothing a client already stores can grow.
- A minor version can therefore only add fields, and §4.1 makes an added field
  optional.
- §3.3 makes an optional field skippable, using bit 6 and, when clear, bits
  5-4 to know how many octets to skip. A receiver skips it knowing nothing
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
`RESPONSE_TOO_LARGE`, which §7 reserves for a declared length above `MAX_LENGTH`
and not for one above the buffer. Read and discard what the buffer does not
hold, and do not interpret anything past the declared length. §3's constraint is
on what a receiver interprets, not on holding a message whole.

`MAX_LENGTH` still bounds the discard path, so a client parsing a 32768-octet
declared length reads it to completion. That costs wall time rather than memory,
and the scratch it discards through can be a few hundred octets reused in a
loop.

A client that has **rejected** the message is not parsing it, and §3 relieves it
of reading the rest: report and close. The distinction is worth coding as a
distinction, because the two look alike in a read loop and only one of them has
an answer already determined. It is also the whole of a client's obligation
here. A server owes a drain because its peer may still be writing a credential;
nothing is waiting on a client's read, so a client owes none.

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
response, then read and discard until the earliest of several conditions. It is
the one path where a server reads from a connection it has already given up on,
and the obvious Go call and the obvious hand-rolled loop behave differently at
EOF.

`io.CopyN(io.Discard, conn, 2*MaxLength-alreadyRead)` returns `io.EOF` when the
peer has half-closed, which is the normal case under §2, and that is the
condition that ends the drain rather than an error. The subtraction is part of
the rule: §3 counts the bound from the whole connection, "counting the envelope
and everything already read before it rejected", so a bare `2*MaxLength`
overshoots by whatever the server had consumed before it gave up.

Count what came off the socket rather than what the parser consumed. A server
that read the request through a `bufio.Reader` has octets sitting in userspace
that the parser never saw, and the section below on octets beyond the declared
length is about exactly those. Leaving them out of `alreadyRead` makes the
subtraction too small and the drain runs past the bound §3 sets. A hand-rolled
`for { n, err := conn.Read(b) ... }` that does not treat `n == 0 && err ==
io.EOF` as terminal spins at full CPU until the deadline, which is a remote
unauthenticated CPU burn reachable without authenticating.

The response goes out before the drain starts, so a client blocked in `read`
gets its answer without waiting for the drain to finish.

**The sketch above never consults the declared length, and that is fine.** §3
makes it a permission to stop early rather than one of the drain's terminators,
and one a server holds only where it may rely on the length, which it may not on
the two paths where the length itself was the fault. On the paths where it may,
draining past it costs a little time under a deadline that is already running
and reads octets a conforming peer never sent. The loop that ignores it drains
at least as long, which is the direction this section's purpose points.

**What the drain is for.** It keeps a client that is still writing a large
`credential` from taking `EPIPE` before it gets round to reading. On `AF_UNIX` a
response already written sits in the peer's receive queue and survives the
server's `close()`, so the drain is not protecting the response itself, and no
reset discards it: the transport delivers queued data before it reports
anything, measured on Linux and macOS on 2026-09-02. What Linux does report
after the data, wherever the closer held octets it had not read, is `ECONNRESET`
in place of end of file. A drain that stops at its bound therefore leaves a
Linux client seeing an error after a response it already holds, which is one
more reason a client does not read on for the end (§2).

The status is the one determined when the message was rejected. A drain that
runs to the deadline still answers `REQUEST_TOO_LARGE`, not `REQUEST_TIMEOUT`.

**The deadline the drain runs under is §2's request deadline**, which is still
running because the request never completed. A server need not set a second one,
and MAY set a shorter one where it would rather not spend the remaining budget
on a peer that already has its answer. What it MUST NOT do is answer again when
the deadline expires mid-drain: the response has gone, and the expiry means
close.

## Writing to the socket

**A server closing while a client is still writing is expected, not
exceptional.** §2 lets a server close without replying before it has read a
version word, on a refusal reached at accept, and where writing fails, which its
response deadline expiring is. §3 has it close when the drain ends, which
narrows that window rather than removing it. A client sending a large
`credential` will meet this. Every client therefore has to decide what a failed
write does to it.

**What the failed write does differs by runtime**, which is why the
specification says nothing about it and this document says a good deal. In C the
kernel raises `SIGPIPE` and the default action terminates the process. In Go the
runtime raises it only for file descriptors 1 and 2, so a write to a closed
socket returns `EPIPE` as an ordinary error and nothing dies. A runtime that
maps signals to exceptions gives you a third shape. The cause is one sequence
and the manifestation is a property of the language, which is why §2.1 binds the
outcome and says nothing about the mechanism: a client MUST NOT let its host
application terminate because a write to the socket failed. Everything below is
how to satisfy that in C, and in Go the rule costs nothing except that an
unrecovered panic on a write error breaks it.

**Suppress `SIGPIPE` per write, and leave the host's disposition alone.**
Writing to a socket whose peer has closed raises `SIGPIPE` in C, whose default
action terminates the process. A client is a guest inside `sshd` or a PostgreSQL
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

None of that reaches the Go client, which gets an error where the C one gets a
signal. What does reach it is the paragraph below: the write failing is not the
end of the exchange, and the response may already be waiting.

**Offer a way to write the request in pieces, for the test suite only.** §7's
F10 asks whether a client can be configured to bound the octets it writes before
flushing, and to pause between writes. Nothing at runtime wants that, and one
write is the right default. It exists because no request this protocol permits
is large enough to leave a client mid-write against a default socket send
buffer, so without the switch a driver cannot make the write fail and §2.1's
rule goes untested against you. A build-time or configuration-only knob is
enough, and a client that offers none is still conforming.

**A short write and an `EINTR` are resumable.** Both are routine inside a host
that installs handlers without `SA_RESTART`, which is the host's prerogative and
not something a guest library may change. Resuming on the same connection is not
the retry §2 forbids; that rule is about opening a new connection. A client that
treats either as fatal fails an authentication because an unrelated child
exited. An `EINTR` from `connect()` is the other way round. POSIX has the
connection go on being established on that descriptor. So the retry §2 permits
closes the interrupted descriptor and connects a fresh one, rather than calling
`connect()` on it again.

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

**A client is in the same position.** §2 delivers a message at its declared
octets, so a client holding them holds the response, and a deadline expiring
afterwards changes nothing: report the `result` you parsed, never
`RESPONSE_TIMEOUT`. Do not read on to end of file to finish the exchange.
Excess-octet detection on a client is the same best-effort check as the
server's, a buffered-data test or a non-blocking read, and never a blocking
read: the server may still be draining, and on Linux its eventual close arrives
as `ECONNRESET` rather than end of file wherever it left octets unread. A client
that requires end of file fails a conforming refusal on that platform and passes
it on macOS, which is the divergence a developer on this machine never sees.

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

**Set the umask; do not inherit one.** §2 bounds the mode from above only, so a
socket that comes out `0600` under an inherited `0077` conforms, and the server
starts, publishes and serves nobody. Nothing on the server side reports it. What
reports it is every client, at once, with `EACCES` and `CONNECTION_FAILED`, so
the fault is loud in the wrong log. A server that sets the umask itself before
`bind()` cannot inherit the problem, and one taking the second route sets the
mode outright and is unaffected.

Measured on Linux on 2026-08-23, Debian 12 on kernel 7.0.12: with umask `0117`
under a `02770` directory, a bound socket comes out `0660` with the directory's
group and no setgid bit. `S_ISGID` propagates to newly created subdirectories,
not to files or sockets, so the first route produces a socket §2 accepts.

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
`(*net.UnixListener).File()` dups the descriptor, which leaves a second one to
close. Before Go 1.11 it also put the original into blocking mode, which
disabled the listener's deadline handling for the rest of the process's life.
`SyscallConn()` reaches the same descriptor without either.

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
`Perm()` masks to `0777`, which drops the three bits `07000` that §2 forbids
just as firmly. A server that omits the mask refuses to start on
a socket it has just created correctly.

## A socket already at the path

§2 requires any socket already at the published path to be gone before the
server publishes its own, and requires the is-a-socket test before anything at
that path is removed or replaced. It requires no liveness test, so staleness is
not a property you have to establish for conformance: whatever socket is there
goes. The strategies below are about what a deployment wants to notice when a
second instance starts by mistake.

**`rename()` does not perform that test for you.** Measured on Linux on
2026-08-23: renaming a socket over a regular file succeeds silently and frees
the replaced inode, and renaming over an existing bound socket succeeds too,
leaving the old server holding a descriptor no new client can reach. Only a
directory at the target refuses, with `EISDIR`. A server publishing by
`rename()` therefore destroys whatever is at its configured path unless it has
already applied §2's test.

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
the server opens itself: it unlinks its own leftover socket before binding, and
an attacker with write permission can bind at the free path during that window.
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
structurally broken, which case 26 catches through its own combination.

## Emitting fields

§3.2 requires ascending tag order, and at most one of each tag in every version
1.0 message type.

**Go: map iteration is randomised.** An encoder that collects fields into a
`map[uint16][]byte` and ranges over it emits them in an order that changes
between runs of the same binary on the same input. A byte-exact vector
comparison then fails intermittently rather than consistently, which reads as
flakiness rather than a conformance bug, and a two-field response passes half
the time. Sort the tags, or build a fixed-order slice.

**One field name can carry two tags.** §6.4 gives `server_desc` `0x0143` in a
`PING_RESPONSE` and `0x0243` in a `STATUS_RESPONSE`, because `PING_RESPONSE` has
no required field to number ahead of it. A tag belongs to the pair of message
type and field, never to the field alone, so an encoder's constants are
per-schema exactly as a decoder's are. A single `TagServerDesc = 0x0243` shared
by both response builders emits a non-conforming `PING_RESPONSE`, and a
hand-authored vector catches it only if someone wrote a `PING_RESPONSE` vector.
This is the mirror of the decoder hazard below, where one field number carries
two tags.

**Do not echo `ludsa_version_minor`.** §3 makes a response's minor version the
server's own for that major version, never the request's. The natural Go shape
produces the breach without anyone choosing it: decode the envelope into a
struct, hand it to the handler, and reuse it to build the response. A `wire`
suite cannot see the fault, because vectors compare a codec against fixed bytes
and the echo happens in the handler above it. §7 drives it, so a conformance run
catches what your own tests will not. Build the response envelope from your own
version rather than from the one you parsed.

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
that wrong. Track only the tags your schema defines: §3.3 has you skip every
occurrence of an unrecognised optional tag, since a later minor version may
assign one that repeats and you hold no schema to judge it by.

After the scan the representation is free. Ordering is a property of the
encoding and not of the decoded message, so handing an application a map is
fine.

## Decoding into a struct

**Key the schema on the whole tag, not on the field number.** The field number
is the tag's high octet, so tags sort by it and §6.4's table reads as though the
number were the identifier. It is not. §3.3 matches a tag whole, and two tags
sharing a number are two different tags.

A `map[uint8]field` keyed by number looks natural and is wrong. `0x0242` is
optional, dynamic, `string`, field number 2, and it passes every shape check;
`credential` in an `AUTHENTICATE_REQUEST` is `0x02C4`, also number 2. Sent in
that order they ascend, so the framing pass accepts both, and a number-keyed
decoder stores the first into `credential`'s slot. `0x0142` before `0x0181` is
the same shape against a client, putting a dynamic `string` where the one-octet
`result` belongs.

No conforming peer produces this: §4.1 requires a later version's field to take
a number higher than every number already assigned in that message type, so
number 2 can never be reassigned in `AUTHENTICATE_REQUEST`. It is what a broken
or hostile peer sends, and §7 drives it against both sides.

**The answer is a rejection, and §4 leaves no choice about it.** Because no
version can give `0x0242` a meaning in an `AUTHENTICATE_REQUEST`, skipping it
would be fail-open against a peer the receiver can prove broken. Compare the
incoming field number against that message type's ceiling (§3.3), the highest
number your own schema assigns there: above it, skip an optional field; at or
below it, reject the message. An unrecognised required tag is malformed at any
number, since a later version's addition is optional (§4.1). A whole-tag decoder
that skips fails §7's case as surely as a number-keyed one that reads, so a
decoder that gets the whole-tag comparison right and the number comparison wrong
still fails.

**A message type your schema gives no field has a ceiling below every field
number**, so every unrecognised optional tag in one is skipped. `PING_REQUEST`
is that type in version 1.0, and it is where the natural implementation goes
wrong: a per-type maximum computed by folding over the schema's field numbers
has no value to return for an empty schema, and code that treats the absent
maximum as "no tag can be above it" rejects a tag §3.3 requires it to skip. That
fails a version 1.1 peer that added an optional field to `PING_REQUEST`, and
nothing in `testdata/` exhibits it, because no version 1.0 peer sends such a
message. Seed the fold with a value below every field number rather than
special-casing the empty schema at the comparison.

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

**The encoder loses the same distinction, and `omitempty` is how.** Everything
above is about decoding, and a Go encoder throws the distinction away before the
octets exist. §3.2 says to omit an optional field that has **no value**;
`json:",omitempty"` and every reflex modelled on it omits a field whose value is
**empty**, which is the other thing. A zero-octet `user` becomes an absent
`user`, and §6.2 reads those as opposite instructions.

It bites the conformance driver first. F3's second decision case sends a
credential the peer accepts with a `user` of zero octets, and an encoder with
that reflex rewrites the case into the absent-`user` path before it leaves the
process. The driver then reports a pass for a case it never sent.

Model an optional field so absence and emptiness are distinct in the type, a
pointer or an explicit presence flag, and have the encoder emit on presence
rather than on non-emptiness. The same applies to `max_age`, where absent and
zero are opposite instructions in the other direction.

**`SERVICE_UNSUPPORTED` is one protocol name at two numbers**, `result` 7 and
`serving` 4: one condition, answered in whichever enumeration the response type
carries. §3.4 fixes numbers and not names, since the number is what travels, so
a language that scopes constants inside their type has nothing to do. Go gets
`ResultServiceUnsupported` and `ServingServiceUnsupported` and the question does
not arise.

**C has one namespace for enumeration constants, so qualify by enumeration.**
The compiler stops a second declaration, so the trap is not two constants but
one: `result` 4 is `AUTHN_DENIED_POLICY`, which Appendix B answers with
`PAM_SUCCESS` at `pam_sm_authenticate`, so a single constant taken from the
`serving` table and compared against a `result` matches the appendix's one
fail-open cell while reading as "the configured `service` is wrong". Declare
`LUDSA_RESULT_SERVICE_UNSUPPORTED` and `LUDSA_SERVING_SERVICE_UNSUPPORTED` and
neither aliases the other.

**A field absent from the wire decodes as zero, and §3.4 is what makes that
safe.** §5 has an `ERROR` message carry no field but `status` and
`error_detail`. A struct field no tag populated holds 0, and §3.4 leaves 0
unassigned in every enumeration, so a receiver treats it as that enumeration's 1
and MUST NOT read it as affirmative. A `result` no response carried is
`AUTHN_UNKNOWN` and never `OK`, and a `serving` no response carried is
`SERVING_UNKNOWN` and never `SERVING`.

Do not rely on that alone. It closes the enumerations, and the shared-struct
hazard below reaches fields it does not cover.

§6 makes establishing `msg_type` before reading any field a MUST for a client,
and it is what keeps this shut. Dispatch on it into a per-type struct rather
than filling one shared struct and branching afterwards. A shared struct is what
makes the hazard reachable, because the field exists in it whether or not the
message carried one.

§6.4 numbers each group from 1, so a tag is unique within its group and not
across the table. `status` and `result` are both `0x0181`; `error_detail` and
`authn_id` are both `0x0243`. A decode-then-dispatch client reading an `ERROR`
as an `AUTHENTICATE_RESPONSE` therefore finds a tag that group does define and
reads `status` into `result`, rather than meeting an unrecognised required tag
and aborting. The numbering does not stop it and is not there to.

§3.4 does. It holds the affirmative outcome at 2 in every enumeration that has
one and leaves 2 unassigned in `status`, which has none, so no status value read
as a result is `OK` and the confusion misreports rather than granting. It
survives a decoder which correctly requires the field to be present, which the
absent-field hazard above does not.

**One case has neither defence, and it is the one to build against.** `serving`
is `0x0181` in `STATUS_RESPONSE` and `result` is `0x0181` in
`AUTHENTICATE_RESPONSE`, and `SERVING` and `OK` are both 2. A client that reads
a `STATUS_RESPONSE` into an `AUTHENTICATE_RESPONSE` without checking `msg_type`
reads a health check as a grant. §3.4 says so itself: two enumerations put an
affirmative outcome at 2, so that section cannot separate them.

What separates them is the `msg_type` check §6 makes a MUST, and §4's order,
which rejects a response type the request does not permit at step 6, before any
tag is compared. Treat that check as the whole of the defence rather than as the
first of two, and dispatch into a per-type struct so the wrong field has nowhere
to land.

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

**The same reflex reaches `user`, and there it is a role-name differential.**
§3.2 binds a client to send the octets its host supplied, for every field. A PAM
module that trims or case-folds `PAM_USER` asks the server about one name while
its host admits another, and the server's `OK` licenses a role nobody requested.
§7 drives it: a driver hands the client a `user` with leading and trailing
whitespace and reads the request off the wire. A host that hands you a name you
cannot encode, a NUL in a `string` for instance, is `INVALID_REQUEST` and no
connection (§3.3), never a name you repair.

**An over-long optional field is the same answer, and the tempting one is
wrong.** A 300-octet `PAM_RHOST` against a 256-octet maximum looks like a case
for dropping the field: it is optional, it is diagnostic, and failing a login
over it seems disproportionate. §3.1 forbids it. Dropping asks a question the
host did not ask, and a policy that turns on `rhost` then sees a request with
none. Report `INVALID_REQUEST` without connecting and let the caller decide
whether to retry without the field. The judgement is the host's to make, and it
cannot make it if your library already made it.

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
and then trims has satisfied the SHOULD and broken the MUST. The report goes to
the server's own log, and `error_detail` is not the place to reach for: a
credential that fails validation is a decision, so the client receives an
`AUTHENTICATE_RESPONSE`, which has no field to carry it.

## Populating `error_detail`

§5 requires `error_detail` to carry no control character, meaning U+0000 to
U+001F, U+007F and U+0080 to U+009F, which is what "safe to log verbatim"
amounts to once `utf8` has supplied valid encoding and no NUL. §3.1 has a server
name the bound when it rejects a field for length, and lets it name the field.
§5 has it describe a condition and never the octets. A server that embeds
anything from the request anyway meets those rules awkwardly, because §3.3
forbids it from rejecting control characters or invalid UTF-8 on input in the
first place.

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
server that overruns emits a message a client using `error_detail` reports as
malformed (§5, §7). The consequence of getting it wrong lands on the
*client*: a PAM module writes `error_detail` to `pam_syslog` on the strength of
§5's guarantee, so a newline the server passed through is a log-forging
primitive in the host's authentication log.

`server_desc` carries the same obligation, for the same reason: §6.1 makes it a
diagnostic, so it reaches a log by the same route.

**On the receiving side, check at the point of use and check code points.** §5
requires a receiver to establish that one of these values is valid UTF-8 and
free of control characters before it logs it, returns it, or acts on it, and to
report the response as malformed where it fails. A receiver that ignores the
field does nothing, so a PAM module pays nothing on the success path.

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

**Everything that arrives on the socket is a peer's choice.** The protocol
bounds what a peer may legally send and says nothing about why it sent it, so a
conforming value can still be hostile. That is plain for `user`, which §6.2
marks as entirely the remote party's. It holds for `error_detail` too, which a
server wrote but may have built out of something a remote party gave it.

Treat every received value as attacker-influenced at the point of use, whatever
guarantee its field carries. A guarantee about a value's content is not a
guarantee about what the value may safely be passed to. The specification does
not enumerate the second, and this document does. The two sections below are the
instances that have bitten.

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

**The client's logging hazard is the opposite one, and it is worse.** A server
guarantees `error_detail`, `server_desc` and `authn_id` are free of control
characters and valid UTF-8 (§5), so a client has nothing to escape. What it has
instead is a value it may be tempted to hand straight to a logging call.

`pam_syslog(pamh, LOG_ERR, fmt, ...)` is `printf`-shaped, so

```c
pam_syslog(pamh, LOG_ERR, error_detail);        /* wrong */
pam_syslog(pamh, LOG_ERR, "%s", error_detail);  /* right */
```

`%` is neither a control character nor invalid UTF-8, so §5's guarantee does not
exclude it. That guarantee covers the octets and not what a receiver hands them
to. §5 also has a server embed text from elsewhere in the
field, so the `%` need not come from a hostile server to arrive.

The same holds for `server_desc` and `authn_id`, and `authn_id` is the one that
reaches furthest. `error_detail` arrives only in an `ERROR` message, so the
exposed call is on a failure path. `pam_ludsa` logs the resolved identity, so
`authn_id` is logged when everything went right.

The consequence is not the same as the server's. `pam_ludsa` is loaded into an
`sshd` running as root. The server may be an unprivileged sidecar. So a format
string turns "a compromised server can admit users" into "a compromised server
runs code as root".

Do not rely on the toolchain for this. `FORTIFY_SOURCE` blocks `%n` in a
writable format string on most distributions, but not on all of them, and `%s`
against an arbitrary pointer reads regardless.

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

For a server that holds each message whole, the budget that matters at §2's
connection bound is `MAX_LENGTH` multiplied by the concurrency limit, not one
message. §3 fixes the first, so the concurrency limit is the only one such a
server sets.

Nothing requires holding a message whole. §3 forbids allocating from a declared
length before it is checked, and forbids allocating for the body of a message
being rejected at all, so a server that validates as it reads and discards on
the drain path holds a field at a time rather than 32768 octets. Its budget is
its own to work out, and it is far below the product above.

## Mapping to PAM

`SPEC.md`'s Appendix B carries the three mappings, and `pam_ludsa` uses them
unchanged. What follows is what the appendix leaves to a module.

**Why `REQUEST_INCOMPLETE` is not `PAM_SYSTEM_ERR`.** A peer that crashed
mid-write is a deployment problem, and reporting it as a protocol violation
sends someone hunting a codec bug that is not there. `VERSION_UNSUPPORTED` goes
the other way for the same reason: no retry resolves a major version mismatch,
and someone has to upgrade a package.

`REQUEST_UNSUPPORTED` means a message type a server does not implement, and
nothing else. An unrecognised `service` used to share it and now has
`SERVICE_UNSUPPORTED` in the response's own enumeration, `result` for
AUTHENTICATE and `serving` for STATUS. Both still reach `PAM_SYSTEM_ERR`, since
a retry improves neither, but an operator can now tell a version skew from a
typo in `service=` without reading `error_detail`.

**Do not map `SERVICE_UNSUPPORTED` onto an authentication failure**, however
naturally a `result` other than `OK` falls into the deny branch. The `service`
came from the module's own arguments and no user chose it, so `PAM_AUTH_ERR`
blames the wrong party and hides a configuration error behind a login that
merely fails.

`REQUEST_MALFORMED` covers a field above its §3.1 maximum as well as a defect in
the framing, and `error_detail` is again what separates them. A `credential`
above 16384 octets is the instance to expect, and §3.1 has a client refuse to
send one rather than let a server discover it, so a server seeing one is looking
at a client that skipped a check it owed.

**The `account` line is a requirement the appendix places on a client, and
`pam_ludsa` has to document it.** An authorisation outcome returns `PAM_SUCCESS`
from `pam_sm_authenticate` by design, which is the emphasised cell in Appendix
B's result table, so a stack without an `account` line lets a
`AUTHN_DENIED_POLICY` admit the caller. The module cannot detect the omission at
run time, so the manual page carries it and says why.

The exposure depends on the server. `AUTHN_DENIED_POLICY` is one of the codes a
server names a reason with, and §6.2.1 makes every one of those optional, so a
server that discloses nothing answers `AUTHN_DENIED` throughout, which Appendix
B maps to `PAM_AUTH_ERR` and which needs no `account` line at all. A deployment
cannot know which kind of server it will meet, so the requirement stands either
way.

**`pam_sm_acct_mgmt` returns the cached verdict's code for anything but `OK`.**
A credential outcome usually stops the stack at the `auth` phase, so the
question looks academic, and a control flag that does not stop the stack reaches
it. Returning `PAM_SUCCESS` there because "the credential failure was already
reported" is the fail-open, and Appendix B's rule that no cached verdict fails,
with `PAM_SYSTEM_ERR` as the code to prefer, is the same rule: `PAM_SUCCESS`
comes from `OK` in this transaction, for this user, and from nothing else.

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

That check is worth only as much as the server's consistency, which is why §6.2
asks a server to carry the same optional fields on every AUTHENTICATE response
within one `service` that carries the same `result`. A server that populates
`authn_id` from its upstream and omits it on a cache hit passes the check and
fails later, on the path the check never took. The cache is the usual instance
and not the only one: any second route to the same `OK` that drops the field
defeats the check just as completely.

**Do not read that as a rule to keep the field set constant across answers.** A
denial resolved no identity and has none to send, so a denial carrying no
`authn_id` is the field meaning what it says. Emitting a zero-length `authn_id`
on a denial to keep the shape uniform is the mistake here: §3.2 makes a
zero-octet value a value rather than an absence, so it says the server resolved
an identity of no octets.

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

**A uniform denial code is not uniform behaviour.** §6.2 offers a server that
will not disclose the option of answering `AUTHN_DENIED` for every denial, and a
server that takes it while returning in a microsecond for an unknown user and
thirty milliseconds for a policy lookup has disclosed the same thing the code
was withholding. A production server may want to equalise the two, by deferring
the answer to a fixed floor or by doing the work either way.

How far to take that is a deployment question and this document does not
prescribe it. The floor costs latency on every denial, the work costs load, and
which matters depends on the traffic. What is worth stating is that choosing
`AUTHN_DENIED` for its discretion and then leaking through timing buys nothing.

## The socket path

§2 requires an absolute path, forbids truncating it, and has each side establish
that its own path fits before it binds or connects. The size of the array the
path travels in is a platform fact:

| platform | `sun_path` | usable |
|---|---|---|
| Linux, Solaris | 108 | 107 |
| macOS, FreeBSD, NetBSD, OpenBSD, DragonFly | 104 | 103 |
| AIX | 1023 | 1022 |

Read from Go's generated type files on 2026-09-01, which are produced from each
platform's own headers. Only the macOS row was exercised on this machine, where
a path of 103 bound and one of 104 did not.

**Derive the maximum rather than hardcoding it.** `sizeof(((struct sockaddr_un
*)0)->sun_path) - 1` in C, `len(syscall.RawSockaddrUnix{}.Path) - 1` in Go, both
compile-time constants needing no cgo and no `unsafe`. The spread above is why:
a two-value assumption is wrong on AIX and on Solaris, and the next platform is
not a value anyone has.

**A C string literal naming the number is available, at a cost.** The
preprocessor cannot stringify `sizeof`, since it runs first, so
`STR(sizeof(...))` yields the text of the expression. What works is a literal
per platform with a `_Static_assert` against `sizeof`, which fails the build
where the assumption is wrong. That gives `ludsa_strstatus()` a message naming
the maximum with no allocation, and it costs a platform list to maintain. An
accessor returning the derived value costs nothing and lets the caller format;
for the client that caller is `pam_ludsa`, which owns `pam_syslog`.

**Neither language reports the condition for you.** `strncpy` truncates with no
signal at all. `snprintf(addr.sun_path, sizeof addr.sun_path, "%s", path)`
returning `>= sizeof addr.sun_path` is the detection idiom where you have not
checked in advance. Go does not truncate, which is the safe half, but returns
`EINVAL`, surfacing as `bind: invalid argument` with no length in it. That is
the whole reason §2 makes the check prior: the syscall cannot tell an operator
what went wrong.

**Treat `EINVAL` from `bind` or `connect` as authoritative anyway.** A binary
compiled against one platform's headers can run under another's kernel:
FreeBSD's Linux compatibility layer and illumos LX zones both do this, and a
Linux-built client there believes 107 while the kernel may enforce 103. No
compile-time technique catches that, because both the `_Static_assert` and the
`sizeof` accessor were correct against the headers they saw. Check for the
message, and let the syscall have the last word. **Unverified**: that these
layers enforce the host's limit rather than translating to the guest's has not
been measured here, and belongs in the Docker harness beside the
`PAM_MAX_RESP_SIZE` figures.

**Relative paths and the working directory.** There is no `bindat` on Linux and
`net.Listen` takes a path and nothing else, so `chdir` is the only route to
directory-relative semantics, and the working directory is process-global. A
library that changes it is mutating state shared with the host process and
racing every other thread, which is the objection that keeps
`signal(SIGPIPE, SIG_IGN)` out of `libludsa` too.

§2 settles what each side does rather than resolving one: a server fails to
start and a client refuses to connect, reporting `CONNECTION_FAILED`. The test
is on the configured string before any call, since a leading `/` is the whole of
it, and neither `bind` nor `connect` reports the condition afterwards.

## Peer credentials

§2 binds a client that was configured with an expected server identity. It must
confirm the peer before it writes, and close and report `CONNECTION_FAILED`
where it cannot. A client configured with none owes nothing.

Nothing binds a server. Filesystem permissions are the access control the
protocol relies on. A server that checks anyway is worth the trouble in most
deployments, and §5 gives it a status to report the refusal with.

**Check at accept, before you read anything.** §5 makes it a MUST. `SO_PEERCRED`
is readable there and costs nothing to keep, so nothing is gained by waiting and
a server that waits has parsed octets from a peer it could already have refused.

**Then choose one of three answers, and the choice is about diagnosis rather
than about correctness.** Close without replying, which leaks nothing and costs
the client its diagnosis. Answer `PEERCRED_DENIED` at once in your own major
version, which costs nothing and reaches a client configured for that version.
Or wait a bounded interval for `ludsa_version_major` and answer in the version
the client declared, which reaches its logs whatever version it speaks and holds
a connection open for a peer you have already refused. Appendix B maps the
status to `PAM_AUTHINFO_UNAVAIL`, the same code a wrong socket mode draws, so
the stack falls through and the log carries the diagnosis.

**Whatever arrives on that interval, the answer is `PEERCRED_DENIED`.** A
version you do not implement is not `VERSION_UNSUPPORTED` here, and an interval
that expires with nothing read is not `REQUEST_TIMEOUT`. Both describe something
the peer did after you had already refused it, and either would send an operator
to the wrong configuration file. The version word chooses an encoding and
decides nothing, which is the only reason reading it is safe.

**Default to the third answer, and fall back to the first under load.** Waiting
for `ludsa_version_major` and replying in the client's own version is what puts
a usable diagnosis in the client's log, and reading and discarding while you do
it keeps a client that is still writing a credential from taking `EPIPE`. Make
that the normal path.

Under load the same behaviour works against you. Every refused connection then
costs an interval and a discard loop, and an unauthorised peer opening
connections in bulk is what produces the load in the first place. Set a
threshold past which a refused connection is closed at once, with no wait and no
discard. §5 permits that without qualification, and a server with no such
threshold has no way to shed the cost.

The concurrency refusal is the same shape and §3 now says so: a refusal reached
at accept owes no drain on either ground. Do not run a discard loop on the
`RATE_LIMITED` path by analogy with the rejection path, which does owe one. The
bound there is `2 x MAX_LENGTH`, so reading a refused connection to that bound
costs up to 65536 octets at the moment you have least to spend them on. A server
that sheds load by draining has built the amplifier it was shedding to avoid.

**A peer rule that turns on `service` is a second gate, not a later check.** The
credentials check still runs at accept. What waits is which services this peer
may reach, and `PEERCRED_DENIED` answers that too. Look the name up before the
peer table: an unheld `service` is `SERVICE_UNSUPPORTED` for every peer (§5), so
a deny-by-default allowlist keyed on names answers the selector first and gates
only the names the server holds. Such a server meeting a request too malformed
to yield a `service` cannot settle the second gate at all, and answers the
framing status instead.

**Do not defer the credentials check to compare process ids.** Reading
`SO_PEERCRED` at accept and keeping it costs nothing, so the uid check runs
immediately and the pid comparison below runs later against the same stored
value. The comparison needs the request; the allowlist check does not.

**Reading the version word and then closing in silence is conforming only for a
refusal reached at accept.** It is the shape a peercred check acquires by
accident: read the envelope, decide the peer is not allowed, return early. §5
permits that for a peer it refused, and §2 forbids it for everything else, since
a server that has read the version word and refused nothing at accept answers. A
client cannot tell a silent close from a crashed server, which is why §5's other
two answers are the better ones even where silence is allowed.

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
may compare it against the `pid` field of an AUTHENTICATE request, and log the
disagreement. It MUST NOT refuse on one. §6.2 makes `pid` logging-only, and a
mismatch is not evidence of an attack: forging the field requires access that
already permits forging the whole request. A strict mode would buy nothing, and
it would cost a deployment valid logins the day its namespace assumption stopped
holding.

A server that logs both should log the claimed pid and the attested pid as
separate fields, rather than substituting one for the other. Otherwise the
logged value means different things in different deployments.

**A client has more to lose than a server does, which is why §2 binds it.** A
socket an attacker planted receives the credential and then answers whatever it
likes. That is an authentication bypass rather than only a disclosure.

§8 leaves the mechanism unspecified, because no single check works everywhere.
§2 keys the obligation to configuration instead. §7 drives it with F5, a driver
socket bound, listened on and owned by a uid the client was not configured to
expect, and asserts the client writes no octet. Where a supervisor binds the
socket and hands the descriptor to another process (§1), a peer-credential check
reports the supervisor's uid, and that is the uid a deployment configures the
client with.

These are the checks a client can be configured for. It should support whichever
its deployment allows:

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

§2 compares uids and says nothing about names, so `libludsa` takes a uid and
does no lookup. A host that wants to configure its server by name resolves the
name and hands the library the number. That is not only a layering preference:
`getpwnam_r` inside an `sshd` PAM stack can `dlopen` further NSS modules and
block on a network directory, and its buffer has no derivable size, since
`sysconf(_SC_GETPW_R_SIZE_MAX)` is permitted to return `-1`. A library that
allocates nothing has neither a size to use nor a retry loop it can bound, and
the lookup would sit on the authentication path. A host doing it at
configuration time has none of those constraints.

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
