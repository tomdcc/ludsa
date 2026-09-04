# LUDSA design rationale

Why the protocol is shaped the way `SPEC.md` says. Nothing here is normative: if
this document and the specification disagree, the specification wins.

Sections mirror `SPEC.md`, so a rule and its reasoning carry the same number.

Some of the rules below prevent a failure an implementation can't detect on its
own. They look unnecessary until two implementations meet, and the conformance
vectors are what expose them. Each one says so where it applies.

## 2. Transport

**Why the socket path must be absolute and must not be truncated.** §2
legislates the path in detail and said nothing about either, which the ninth
round found. Both produce the same failure, and it is the one §2's peer-identity
rule exists to close: a client connects successfully to something other than the
server.

A relative path resolves through ordinary pathname lookup against the calling
process's working directory, since the socket calls have no directory argument.
That directory belongs to the host process a client is loaded into. A PostgreSQL
backend has moved to its data directory and a daemon has usually moved to the
root, so the same configured string names different sockets on different hosts,
and can name different ones in one process either side of a `chdir`.

Truncation is the same failure by another route. The path travels inside a fixed
array in the address structure, and the natural C spellings shorten it without
saying so. A shortened path is still a valid path: it names nothing, which is
harmless, or it names a different socket, which is not, and no error
distinguishes either from success. The rule is therefore that a side establishes
the path fits before it calls, rather than only that it must not truncate.
Neither `bind` nor `connect` reports the condition afterwards.

The rule states a property rather than a number, for the reason §2's socket mode
states a bound rather than a value. The size is a platform fact and it varies
more than two ways: Linux and Solaris carry 108, the BSDs and macOS 104, and AIX
1023. A normative 108 would make an implementation on a smaller platform
conforming and broken. It is not even a single number within a platform, since
portable practice leaves room for a terminator and the usable length is one
less.

The two sides configure their paths independently and need not share a string. A
volume mounted at different points gives one socket two names, and neither side
reads the other's, so each validates its own path against its own platform. That
also disposes of any cross-platform concern: a socket is a kernel object, so
both peers are talking to the same kernel and see the same limit, and they would
not care if they did not.

The ninth round wrote both rules and equipped only one. Truncation got a party
on each side, a report name and a case. Absoluteness got a MUST on the
configuration, which binds nobody: a configuration is not a party, so nothing
said what a server does with a relative path or what a client reports for one,
and two implementations could both claim conformance while one refused to start
and the other published a socket at a name that meant something different to
every client. The tenth round gave it the same equipment, on the ground that the
two rules guard one failure and a rule with no party is one a conforming
implementation may ignore.

`CONNECTION_FAILED` rather than `INVALID_REQUEST`, for the same reason
truncation takes it: §7 gives `INVALID_REQUEST` to the caller's own input, and a
socket path is the deployment's configuration rather than anything the caller
supplied. It carries the fall-through that Appendix B gives
`PAM_AUTHINFO_UNAVAIL`, which the truncation rule already carried, so nothing
new turns on it here.

**Why a client must survive a write failing, after four rounds of declining
it.** Every earlier proposal was a rule about signals, and every decline
answered one: a specification reaching into process-global state is specifying a
library, and the same sequence raises `SIGPIPE` in C, returns `EPIPE` in Go, and
takes a third shape in a runtime that maps signals to exceptions, so a rule
would be written for one and wrong for the rest.

The ninth round's proposal is about the outcome instead. It names no signal, no
errno and no runtime. A C client satisfies it with `MSG_NOSIGNAL` or
`SO_NOSIGPIPE`, a Go client by doing nothing, since its runtime raises the
signal only for descriptors 1 and 2, and a runtime that maps signals to
exceptions by catching. Where a client already gets it right the rule binds
nothing, and a rule that is vacuous somewhere is not a rule that is wrong there.
Go is not wholly exempt either: an unrecovered panic ends the process, so a
client that panics on a write error breaks it.

**The boundary is neither of the two this entry tried.** Observability failed
because §5 binds a client's dealings with its host application twice with
nothing on the wire to show for it. Provenance failed because §2 binds a client
to apply deadlines, which concerns no value the protocol delivered. What
separates this rule from allocator behaviour and threading is that the protocol
authorises the condition: §2, §3 and §5 each let a server close a connection a
client is still writing to, so the write failing is a conforming exchange rather
than a fault.

**The fourteenth round tested that boundary against §2's deadlines and the rule
stayed where it is.** §2 lets a server take as long as it likes reaching a
decision, so the boundary reaches the deadline as well as the write, and in C
the textbook way to bound a blocking call is `alarm()` with a `SIGALRM`
disposition, which cancels the alarm `sshd` arms for its login grace period.
What separates it from the write is that no conforming peer creates the
condition: a client applies a deadline because §2 tells it to, and it chooses
how. Nothing on the wire distinguishes `alarm()` from a `poll()` loop, and F7
cannot, since both report the same outcome. So the hazard is real and it belongs
in the guidance, which now names it.

That also answers the objection that would otherwise sink it, which is the one
that removed the format-string MUST: naming one condition implies the rest are
safe. The rest are not conforming exchanges. No conforming server sends a
malformed response, so a client dying on one is a library bug outside this
document, where a client dying on a close this document permits is the document
producing the crash.

The client is not a peer that owns its own process. §1 has it loaded into a host
application, and a PAM module is loaded into an `sshd` running as root rather
than forked from it. A protocol that specifies that relationship and then
authorises the peer to kill it has said something incomplete.

**Why unix sockets only, and pathname sockets at that.** The credential crosses
this channel in cleartext, and filesystem permissions are the only access
control the protocol relies on. TCP on loopback would put it on a channel with
no such control.

Linux's abstract namespace is the same objection in quieter form. An abstract
socket has no inode, so it has no owner, group or mode, and the only thing
between a caller and the socket is the network namespace boundary. It vanishes
when the last descriptor closes, so there is never a stale socket to unlink,
which is a rule §2 would otherwise be glad to lose. But that is one property
seen from two sides. Nothing on the filesystem means no cleanup and no
permissions, and §8 makes permissions the whole model.

**Why one request per connection.** There is no desynchronisation state to
reason about. Pipelining would introduce that class of bug for no gain, given
the transport is local and the exchange is a single round trip.

**Why both sides half-close, and why their reasons differ.** A client holds the
connection open to read the reply. Without an explicit half-close the server has
no end-of-request signal beyond the length prefix, and a client that dies
mid-write leaves it blocked until its read deadline. The half-close turns that
into an immediate error.

The server's side is stated separately, and the tenth round made it
unconditional: it shuts down its write direction as soon as it has written the
response, before any drain it owes rather than after. Deferring it until the
drain finished left a client that had chosen to read for the end waiting on a
server holding the connection open exactly as §3 tells it to, while the same
paragraph of §2 promised that a client could read to end of file without risk of
blocking. Half-closing costs a draining server nothing, since a drain only reads
and §2 allows no second response. The eleventh round then took the end out of
delivery altogether, which is the next entry, so nothing depends on the ordering
any more. The rule stays because it costs nothing, §7 observes it, and it keeps
that promise true for a client that reads on anyway.

**Why a message is delivered at its declared octets, and the end is not part of
it.** The tenth round asked what a client holds when it has parsed a complete
response and its own deadline then expires on the read that follows. The
document had no answer: §7's names covered a response that never started and one
that stopped part-way, and this one did neither. The two available answers were
the outcome it parsed and a failure, and they reach `PAM_SUCCESS` and
`PAM_AUTHINFO_UNAVAIL` on identical wire input.

The tenth round's ruling was that a transfer is its declared octets and the end
of the sender's side of it, so that a receiver holding every octet and no end
had received nothing and had no outcome to weigh against the failure. The
eleventh round reversed it, for two reasons found separately. All three of its
reviewers read the rule as binding a server to wait for the client's half-close
before it had received the request, which §4's order, §7 and the guidance all
contradicted. A client that never half-closed drew `OK` from one conforming
server and `REQUEST_TIMEOUT` from another. And a measurement showed that Linux
reports a peer's close as `ECONNRESET` wherever the closer held unread octets,
which every refusal reached at accept and every bounded drain produces. A client
reading for the end therefore met an error on the paths §5 permits, and could
not tell it from a connection that was actually reset. The author's ruling was
to drop the end from delivery. Scoping it to one side and naming the platform
behaviour were the two remedies on the table, and the ruling went under both.
The tenth round's argument was wrong on its own terms: the end proves nothing
the declared length had not already proved, since a sender that declared a
length has nothing more it could have meant to send. What the client in the
tenth round's question had done was block on an exhaustive excess-octet read,
and the answer to that is not to block. A message is delivered at its declared
octets on both sides, nothing that happens afterwards changes what was
delivered, and a deadline expiring after delivery cannot take a message away.
The transport supports it: `AF_UNIX` never discards queued data on close, which
the same measurement confirmed on both platforms. The client-local codes went
back to splitting on how much arrived, which the entry on the fourth code
records, and the round-ten case that expected `RESPONSE_TIMEOUT` for a complete
response held open inverted rather than died: it now expects the `result`, and
fails a client that waits for the end. The platform behaviour went to the
guidance, where it belongs, and its normative traces are §2's clause on the form
a close is reported in, and `RESPONSE_INCOMPLETE` and §3's drain terminator each
naming, alongside end of file, an error that ends the read.

**Why reading on for excess octets does not wait.** The eleventh round left a
server that holds a complete request and reads on for excess octets with no
answer to its own deadline firing. The delivery rule said it holds the request;
§2's timeouts paragraph said an expiring request deadline is `REQUEST_TIMEOUT`.
Two conforming servers answered `OK` and `REQUEST_TIMEOUT` on identical octets,
which is the defect the reversal was meant to remove, surviving one paragraph
away from it. The deadline also names a span, accept to complete request, that
has ended by then. The twelfth round bounded the read and said the expiry is not
`REQUEST_TIMEOUT`.

The thirteenth round removed the wait, which is what the bound had been
managing. A server that waits defers a response it already owes, and nothing in
that response depends on octets the delivery rule puts outside the message. The
check was best-effort in any case, since §2 concedes the excess may not have
arrived and obliges nobody to wait for it, so the bound bought a racy diagnostic
at the price of the whole request budget. §3's drain observes anything later for
nothing, after the response has gone, and a server there can log what it finds.
Deleting the wait deletes the seam with it: where there is no bound there is no
expiry, and the qualification the twelfth round added has nothing left to
qualify.

**Why EOF is not a reason to abandon the exchange.** Treating a half-close as a
closed connection is the obvious way to implement this wrongly, and it produces
a server that works only against clients that omit the half-close. That is, only
against non-conforming ones.

**Why the half-close finally got a case.** For six rounds §2 stated it as a MUST
and §7 reached it with nothing. Both server-side rules describe tolerating its
absence, so the suite could not have failed a client that never half-closed at
all, and the seventh round said so.

The case has two assertions because §2 has two rules and one of them is passed
by a client that breaks the other. A client that closes the connection outright
gives a driver end of file exactly as a half-close does, so a case checking only
for end of file admits it. What separates them is the write that follows: a
half-closed connection is still readable by the client and takes the response,
and a closed one draws `EPIPE`. That is the mechanism §3's drain case turns on
as well. The half-close case takes F6 for the wait alone: a client that never
half-closes holds the driver until its own deadline ends the exchange, and the
driver has no other bound to wait to.

Nothing obliges a server to wait for end of file, and the case does not create
such an obligation. A server has the declared length and reads that many octets,
so waiting would put a round trip on every exchange to strengthen a SHOULD that
§2 already says need not be exhaustive. The obligation sits on the client, which
is where the cost of meeting it is nil.

**Why the obligation to answer begins at the version word.** Silence tells a
client nothing. It looks the same as a crash, a wrong socket path, or a server
that closed for any other reason, so whoever is debugging starts from scratch.
That argues for answering always, and the rule stops one step short of it.

`ludsa_version_major` sits at offset 0, so a server holding two octets knows
which version to answer in, and the obligation starts exactly there. Below it
nothing has been said. A bare connect carries no request, and the common thing
that produces one is a port scan, so an answer would go to something that asked
nothing.

The sentence below the threshold used to say flatly that a server closes without
replying, which the paragraph after it contradicts by letting the same server
answer in its own major version. It now says nothing obliges it to reply, and
points at the permission.

**Where the excess-octet check sits in §4's order.** It sits nowhere in it, and
the ninth round had to say so. §4 opens by promising that a message failing more
than one check takes the outcome of the earliest, which gave a reviewer a
question the document could not answer: does a server that notices stray octets
after reaching a decision still answer `REQUEST_MALFORMED`?

Most of what looks unsettled there was settled on purpose. Detection is not
required to be exhaustive, and §7 gives the case no expected outcome, so a
server that never looks and one that looks early already answer differently and
both conform. What was left was narrower, and the window is small: §3 has a
server send one response to a connection, so the question lives only between
reaching an outcome and writing it.

The answer follows from where the octets are. §3 puts them outside the message,
so the check is about the connection rather than about the message, which is the
same reason §5's peer check is not one of the steps either. A check that is not
part of the order does not slot into it: a receiver that observes them before it
has answered rejects on them whatever the order had produced, and one that
observes them afterwards has already answered.

**Why detecting excess octets is optional but rejecting them is not.** These are
different questions. Whether a client or server notices octets past the declared
length depends on timing, because they may not have arrived yet. What it does
once it has noticed depends on nothing: a client and a server disagreeing about
where a message ends means one of them has a framing bug, and that is worth
failing loudly on.

**Why the specification sets no deadline values.** It says which deadlines exist
and what each expiry produces, which is what an implementer needs to be correct.
Numbers are a tuning decision belonging to a deployment, and freezing them into
a normative document would make a server under load unable to defend itself
without becoming non-conforming.

**Why a bound on waiting for the version word belongs to a refusal reached at
accept and to no other server.** Earlier drafts had a server allow a short
interval for it before answering in its own major version, so that a client
about to send got an answer in its own version. The ninth round removed the
interval and left the request deadline as the only bound, and the eleventh
returned a bound to a refusal reached at accept, as the end of this entry
records.

The interval was a second deadline that was not one of §3's drain terminators,
and that is what made it costly. When it expired, §2's request deadline was
still running, so a drain was owed on a connection whose read side a server had
just given up on. In Go that is not merely awkward: the read deadline latches,
so the drain cannot run at all, and the guidance told a server not to reset it.
A server following both documents skipped a drain it owed, and §7 cannot catch
that, since whether a drain ran is not observable from outside.

Removing it costs nothing, because the wait it bounded is not a real wait. A
client holds everything it will send before it connects, the transport is a unix
socket, so writing involves no network, and the version word is two octets that
always fit in a socket buffer. Nothing but scheduling delay can hold it up, and
a client too starved to produce two octets will not produce a whole request
within the deadline either. So the request deadline bounds the same set of
clients the interval did, and its expiry is already a drain terminator, which is
what makes the in-flight question disappear rather than get answered.

A server that cannot afford to wait has two answers that were always there: it
may answer in its own major version at once, or stop accepting and leave
connections in the backlog. Neither needs a second deadline.

§5's peer refusal keeps a bound of its own, and that is consistent rather than
an exception. The hazard was never a shorter bound; it was a shorter bound where
a drain is owed. A server refusing the peer owes none, so nothing is left in
flight for the bound to strand.

The twelfth round found the neighbouring sentence asserting that a server
deciding to answer without a version word has refused at accept. A server whose
request deadline expired with one octet read has decided to answer and refused
nothing. Nothing observable turned on it, since that server has met the
terminator anyway, but the sentence claimed a population where it meant to scope
a permission, and it now scopes the permission.

The eleventh round found §2 still giving the drain as the reason no shorter
bound exists, after the tenth had made every refusal reached at accept owe none.
A server that decides to answer before it has read a version word has refused at
accept or has met the deadline that would have bounded a drain, so the reason
was false for every server the sentence bound, and the concurrency refusal was
denied the bound the peer refusal had on ground the two now share. §2 grants it
to any refusal reached at accept. The rule for an ordinary exchange is
unchanged: a server that has refused nothing waits for the version word under
the request deadline, and the guidance still says why a shorter one there
strands a drain.

**Why a server answers `UNAVAILABLE` as soon as it knows.** The obligation is on
promptness rather than on holding a timer. A backend that returns an error at
200ms should not leave the server sitting on the connection until some deadline
fires, and requiring the deadline instead of the promptness would permit exactly
that.

**Why a connection limit has no required behaviour.** Answering `RATE_LIMITED`
is the more diagnosable option, since the alternative reports overload as a
client-side timeout with nothing on the wire. But letting the kernel backlog
absorb it is a legitimate resource choice, the client's own deadline bounds the
wait either way, and a server knows its own constraints better than this
document does.

**Why `RATE_LIMITED` covers both a declined check and a declined connection.**
§5 used to define it as work a request would have required, which describes
§6.3's producer and not §2's: the concurrency limit is reached at accept, so
nothing has asked for anything. Two statuses was the alternative and buys
nothing. Both map to the same PAM code, both mean wait and try again, and no
client branches on which limit it met. What a client needs from the name is that
capacity ran out and a later attempt may fare better, and that is true at either
end.

The twelfth round found the prose reading as though the two producers were the
whole of it, against a definition covering any limit a server sets on itself. A
server budgeting backend calls and declining an AUTHENTICATE on that budget is a
third, and it had `RATE_LIMITED` under the table and `UNAVAILABLE` under the
prose. The prose now names the property and gives its producers as instances.

**Why the retry rule states a property rather than a list.** An enumeration goes
stale against conditions the protocol itself produces. §2 advises a server at
its limit to stop accepting and leave connections in the kernel backlog, and a
non-blocking `connect` against a full backlog returns `EAGAIN`: a closed list of
two would recommend an overload behaviour and forbid recovering from it. `EINTR`
is the same shape, arriving from a host that installs handlers without
`SA_RESTART`, which is the host's business and not a guest library's.

§2's gloss on `EINTR` once said that nothing about the connection was
established, which is not what POSIX says of a blocking `connect()`: the
connection goes on being established on the descriptor. The gloss now says the
attempt was interrupted before it was answered, and the guidance says to close
that descriptor before retrying.

What the rule is actually for is forbidding a retry of something a second
attempt cannot change, which is a decision, a timeout, or a protocol fault. Said
that way it stays correct as the list grows.

**Why a timeout, a protocol error and a denial are never retried.** A timeout
means the server is up and struggling, so retrying adds load while consuming the
caller's authentication budget. Protocol errors and denials are answers, not
transient faults.

**Why the deadline is overall rather than per attempt.** A per-attempt timeout
multiplies by the retry count, and the caller's authentication budget does not.

**Why socket permissions are stated as a visibility property.** Requiring the
server to *create* the socket with a given mode is the obvious wording and not
something `bind()` lets anyone do: the mode comes from the umask and a `chmod`
follows. Stating what a client can rely on instead leaves the mechanism open, so
a setgid directory, an atomic rename, or a platform facility that sets both at
creation all satisfy it equally.

It also covers the case where the socket is not the server's to create. Where a
supervisor binds and hands over a descriptor, §1 makes the two together the
server, and the property holds or it doesn't regardless of which of them
established it.

**Why the socket's group is configured rather than derived.** `bind()` gives the
socket the creating process's primary group. A server that lets that stand is
guessing at the client's identity, which it has no way to know, and the failure
is a socket the client gets `EACCES` on with nothing in either document
explaining why. There is no default for the same reason, so a server given no
group fails to start rather than guess.

**Why the mode is stated as a set of bits.** "More permissive than `0660`" has
no total order to compare against. `0606` is numerically below `0660`, so the
obvious `mode > 0660` admits a socket that is world-writable and inaccessible to
its own group, while an implementer comparing the three triples separately
rejects the same socket. Naming the bits removes the choice.

It is a bound rather than an exact value because the two directions fail
differently. A mode tighter than `0660` stops clients connecting, which a
deployment discovers immediately. A mode looser than it is the one nothing
surfaces.

The seventh round raised the permitted narrowing as a finding, which is the
argument for §2 saying it rather than leaving it here. The reasoning had been
settled for rounds and a reviewer still read the upper bound as an oversight,
because nothing in the normative text distinguished a bound chosen from a bound
that had not been finished. A rule stated as a bound should say it is one.

Requiring `0660` exactly was the alternative and would have been the same rule
twice over. The upper bound already forbids every extra bit, so a lower bound
saying the group can connect leaves owner and group at `rw` and everything else
clear, which is `0660` and nothing else. There is no middle setting to choose,
only the freedom to tighten, and tightening is what a deployment whose client
and server share a uid does on purpose.

**Why a server must fail to start rather than widen the mode.** The tempting
fallback for a server that cannot set the group is to open the mode until things
work. That trades a startup failure nobody can miss for a socket anyone on the
box can authenticate through, and nothing would ever surface it.

**Why a socket at the path must be cleared, and why only a socket.** The inode
survives a crash, and `bind()` would otherwise fail with `EADDRINUSE`.
Unqualified, though, that rule deletes whatever is at the configured path. Point
it at a regular file by mistake and the server removes it at startup. Refusing
to start turns a destructive misconfiguration into a loud one.

The rule states a property rather than a mechanism, because the two publishing
routes reach it differently. One unlinks the socket already there and binds in
place; the other binds to a temporary name and renames over it, which frees the
replaced inode just as an unlink would. An earlier draft required the unlink,
which the second route does not perform, so a recommended implementation
violated a MUST on a literal reading.

The word "stale" was doing work the rule never asked for. It reads as a property
a server has to establish, and a unix socket outlives the process that bound it,
so nothing about a path holding one says whether a server is running. The tenth
round replaced it: any socket already at the path goes, and no liveness test is
required. What a deployment does to notice a second instance started by mistake
is `IMPLEMENTATION.md`'s business, not a conformance question.

**Why there is no rule about the socket's directory.** A user who can write to
the directory can unlink the socket and bind their own at the same path. The
client connects by path, reaches that socket, and hands it the credential, and
no permission on the original socket prevents it, because by then the original
is gone. Unlinking a stale socket opens a second window on the same hazard,
since the path is briefly free at every restart.

The hazard is real and a rule about it was not. Every directory is writable by
root, so a literal test excludes the deployment this protocol assumes, where
`/run/ludsa` belongs to `root` and the server does not run as root. Holders of
`CAP_DAC_OVERRIDE` sit the same way, and counting them leaves no directory that
passes at all. Testing only the immediate directory misses an ancestor that can
be renamed, and testing every ancestor states a property the server confirms
once and cannot hold.

PostgreSQL checks nothing here and rests the whole weight on authentication.
LUDSA cannot do that, because the socket's permissions are the only gate on
STATUS. So the layout belongs to a deployment. `IMPLEMENTATION.md` describes it,
and describes what a client examines when it satisfies itself about a server by
permissions rather than by peer credentials.

Search permission is the intuitive thing to restrict, and is not restricted. A
process that can traverse the directory but is not in the socket's group can
`stat` the socket and nothing more, because `connect()` on a unix socket
requires write permission on the socket itself.

**Why a server may answer without a version.** The rule used to be that a server
below `ludsa_version_major` closes without replying, on the ground that it has
no version to answer in. That is half true. It has no version the *client*
chose, but it always has its own, and §4 already puts that to work: a server
meeting a major version it does not implement replies `VERSION_UNSUPPORTED` in
its own. Answering when it has not been given a version is the same move, and
refusing to make it cost the two cases below their diagnosis.

Waiting for the version word is still the better answer where a server can
afford it, because a version the client chose beats one the server picked. What
the ninth round removed was the short bound §2 used to put on that wait. It was
a second deadline and not one of §3's drain terminators, so its expiry left a
drain owed on a connection the server had just stopped reading, and §2's request
deadline bounds the same set of clients without that consequence. The eleventh
round returned that bound to a refusal reached at accept, which owes no drain,
and the entry above on the version-word bound records why.

**Why §2 stopped enumerating when a server may say nothing.** It used to close
the list: having read the version word, a server closed without replying "only"
when writing failed, or when a fault after it had begun writing left it unable
to finish. §5 then let a server that refused the peer on its credentials say
nothing at all, and read and discard so as not to cut a client off mid-write,
which reads the version word on the way. The tenth round found the two in
contradiction on the path `IMPLEMENTATION.md` recommends.

Extending the list was the obvious repair and it was the wrong one. The reasons
a server says nothing are not the protocol's to hold: one may decline to spend
anything further on a caller it has already ruled out, and one shedding load may
decline to wait for a version word it has no capacity to act on. Those are a
server's judgement and a deployment's policy, and a client cannot separate them
from each other or from a breach. It sees no response and a closed connection,
which is `RESPONSE_ABSENT` whatever produced it.

So the enumeration went and the obligation stayed. A server answers once it can,
and what still binds it is the far end: having begun writing a response it
finishes it. That much a client can act on, because a response that stops
part-way is a fault it can name. Which is the same lesson as the counts rule
elsewhere in this document, arriving at a MUST instead of at a number: a closed
list is a claim about every case, and this one had already been outgrown twice
before anyone noticed it was a claim at all.

The eleventh round found that the tenth round's statement had gone a step too
far. "What binds a server is the other end of it" read, to two of its reviewers
independently, as making the answer itself optional after the version word,
against §6's MUST, §6.3 and §7, so a server that read a full request and dropped
it under load conformed under one section and not under three. The two examples
the tenth round gave of judgement, a caller already ruled out and a load-shedder
declining to wait for a version word, both sit at accept, and §3 had already
named that property for the drain. So the MUST is back with its exceptions
stated as a kind of case: a refusal reached at accept may say nothing, and a
server that began writing finishes unless writing fails. A third refusal reached
at accept would fall under it without anyone editing §2, which is what separates
a property from the list that went.

**Why the request deadline needed qualifying.** The unqualified sentence told a
server to answer `REQUEST_TIMEOUT` whenever the deadline expired, including
before two octets had arrived, which the paragraph above it forbade. §7 case 19
already carried the qualification. §2 and §7 disagreeing about the same
condition is the shape of defect this document keeps producing at section seams.

The case needed a second qualification in the thirteenth round. A stalled
message has not ended, so the alternative to `REQUEST_TIMEOUT` is not
`REQUEST_INCOMPLETE` and §7's truncated-and-faulty exclusion does not reach it.
A driver that stalls behind a prefix carrying a fault gets `REQUEST_MALFORMED`
from a server validating as it reads, under the permission §4 grants, and
records a failure against a conforming server. The case now says the octets
carry no fault.

**Why `RATE_LIMITED` reads the version word where it can.** The concurrency
limit is reached at accept, before anything has arrived, so answering it was the
one status the protocol asked a server to send with nothing in hand. Reading the
version word costs two octets against the cost of having accepted the
connection, and it keeps one rule about when a server may speak rather than a
rule with an exception. A server at its limit may also answer in its own major
version without waiting, which is the same permission every answer without a
version has, and it is the one to take when the limit is what it is defending
against.

The eleventh round settled what the refusal answers when the wait turns up
something else. It is settled at accept on the same terms as §5's peer refusal:
a version the server does not implement, read while it waits, is `RATE_LIMITED`
in the server's own major version, which `VERSION_UNSUPPORTED` does not
displace, and an expiry while it waits is `RATE_LIMITED` in its own major
version or a close. Before that, §4 and §2 gave a server at its bound two
answers for a future-major client, and they map to different PAM codes. The
trade is the one §5 already made: the client loses the version diagnosis on the
exchanges a loaded server refuses, and gains the answer that tells it a later
attempt may fare better, which is the one true thing the server knows.

**Why a client has a name for a request it cannot send.** §3.1 forbids a sender
emitting a field above its maximum and forbids anyone truncating one to fit, and
for four rounds it said nothing about what a client does when its own caller
hands it 20000 octets of credential. The obvious name is barred: §5 keeps
`REQUEST_` codes for a server's statement about a request it received, and a
request that was never sent gives one nothing to describe.

A rule with two MUSTs and no way to report its own violation gets implemented as
truncate and hope, which is the identity-collision hazard this document warns
about for `authn_id` arriving by another road. `INVALID_REQUEST` closes it, and
a client whose host wants finer codes may subdivide it, since the client-local
names are a SHOULD and one using its own is conforming. The weight sits on the
prohibition rather than the name.

An earlier round ruled the same thing and the ruling never reached the
specification, which is how the gap survived a rename that would have
contradicted it.

**Why an over-long optional field is refused rather than dropped.** For four
rounds the rule said nothing about the case, and the two readings differ by
whether an authentication succeeds. §3.2's omit rule reaches an optional field
that has no value, and this one has a value the client cannot legally send.
Dropping it asks a different question: `rhost` is a host observation a policy
may turn on, so a request without it may draw a different decision than the one
the host application asked for.

The deeper reason is layering. Truncating a value, or deciding a diagnostic
field matters less than the login, is a judgement about what the request means,
and the host application is where that judgement belongs. A client library that
makes it silently has substituted its own policy for its caller's, and the
caller cannot see that it happened. Refusing puts the choice back where it can
be made knowingly: the client reports, and the host retries without the field if
that is what it wants.

**Why one half of that sentence is a MUST NOT and the other is not.** It said a
client "reports the condition §7 names `INVALID_REQUEST` to its host
application, without opening a connection", in the indicative, next to two
prohibitions. The eighth round read the whole thing as an obligation nothing
tests, and the halves are not alike.

Not opening a connection is about the socket, which is what this document
governs, and it is the half a driver could observe. It is a MUST NOT.

Reporting to the host application is a rule about a client's dealings with its
own caller, over a value the protocol never delivered, since the host handed it
in. Promoting this sentence to a MUST would bind a client on something no peer
can observe and that no conforming exchange produces, which is the line
objection 5 was eventually accepted on: the protocol requires a client to
survive what it authorises a peer to do, and a host handing its own client an
oversized value is not that. It stays in the indicative, which is what it always
was and now visibly on purpose.

**Why a client's own deadline is `RESPONSE_TIMEOUT`.** It had two candidate
statuses and §5 supported both readings. The rename settled half of it:
`REQUEST_TIMEOUT` is the server's request deadline and a client cannot report
it, and `UNAVAILABLE` lost its client clause when the response codes were split
out. What was left was which client code covers it, and the three that existed
then were all defined by end of file, which a deadline expiry is not.

A fourth code keeps a distinction worth keeping. The tenth round made a transfer
include its end, so that what ended the wait became the whole question and how
much arrived separated only the two the server ended. The eleventh round took
the end out of delivery again, which §2's entry records, and the codes reach
only a wait that ended short of the declared octets: a client holding them holds
the response and none of the three applies. Within that, how much arrived
separates the two the server ended, nothing at all and part of a response, and
the client's own deadline names the third whatever had arrived, because the
client is still waiting and holds no fault the server committed.

All of them map to `PAM_AUTHINFO_UNAVAIL`, so the splits earn their keep in a
log rather than at the stack, which is the trade `RESPONSE_ABSENT` and
`RESPONSE_INCOMPLETE` already made.

**Why a server closes without replying when the response deadline expires.** The
channel is the thing that is blocked, so there is nothing to send it.

**Why the client's check is conditional rather than encouraged.** A socket an
attacker planted receives the credential and then answers whatever it likes,
which is a bypass rather than a disclosure. The protocol said nothing about it
for three rounds, while §2 spent three MUSTs on the server's half of the same
threat, constraining what appears at the socket path.

Two forms were declined before this one. A MUST naming `SO_PEERCRED` is
unsatisfiable where the two sides do not share a user namespace. A bare SHOULD
is satisfiable everywhere and enforceable nowhere: no conformance case can fail
a client that simply did not check.

**Why the configured identity is a uid and not a name.** §2 offered both for
several rounds, and only one of them is a protocol matter. Comparing uids is
numeric equality against what the kernel reports. Comparing names needs a name
service, and §2 never said whose: across a container boundary the client's
`passwd` and the server's differ, so one configured string means two things. The
tenth round found the normative text offering a form `IMPLEMENTATION.md` tells
an implementer not to build, in the libpq example whose name comparison is
called the part not to copy.

The C cost decided it. Resolving a name means `getpwnam_r` inside an `sshd` PAM
stack, which can `dlopen` further NSS modules and block on a network directory,
and whose buffer has no derivable size, since `sysconf(_SC_GETPW_R_SIZE_MAX)`
may return `-1`. A library that allocates nothing has no size to use and no
bounded retry, and the lookup would sit on the authentication path.

So the protocol compares uids and says nothing else. A deployment that knows its
server by name resolves the name to a number, and whether the client does that
itself or makes its host do it is the client's own affair. Removing the
underspecified resolver beat specifying it, and it costs a deployment one lookup
at configuration time, where none of the constraints above apply.

Keying the obligation to configuration fixes both. A client nobody configured
owes nothing, so the rule is satisfiable in every deployment. A client that was
configured is bound, and a driver can see it: accept the connection, and a
conforming client sends no octet. The case it closes is the one that matters,
a client told what to expect that sent the credential anyway.

**Why an unconfirmed peer fails rather than proceeds.** Not confirming has two
causes and the safe answer is the same for both. A peer that is someone else is
the attack. A peer the client cannot identify means the deployment asked for a
check its platform will not support, which is a configuration error, and
answering it with a working authentication hides the error until someone
exploits it. Failing is loud at deployment time, which is where a configuration
error should surface.

**Why bound concurrent connections.** Robustness rather than defence: anything
able to reach the socket is already an allowlisted uid. The concern is a wedged
client consuming capacity and blocking legitimate authentications, not an
adversary.

## 2.1 Independence of the two sides

The two sides share no filesystem beyond the socket's own directory, no memory,
and no process namespace. That is what forces every value into the message and
rules out passing anything by reference.

The rule holds for every later version, not only this one, because a field that
carried a path would work in testing and fail in the deployment the protocol
exists to serve. `pid` in AUTHENTICATE is self-reported for the same reason. It
is not laziness: in separate PID namespaces it is the only thing that can work.

## 3. Framing

**Why network byte order.** Big-endian is the convention for wire formats, and
the conversion helpers for it are portable: `htonl` and `ntohl` are POSIX. There
is no portable equivalent for little-endian, where glibc, the BSDs and macOS
each expose different names in different headers, so choosing it would put a
portability shim into a library that depends on nothing but libc. The saving it
would buy is a few byte swaps per authentication, against an upstream round
trip. Note that this machine and most others are little-endian, so a host-order
bug passes locally and fails only against a conforming peer.

**Why the version comes first, and why almost nothing else is fixed.** Rejecting
an unimplemented major version is only implementable if the receiving side can
trust its own reading of the major version. Anything placed before it, including
a length prefix, would have to be frozen too, and freezing it assumes a future
major wants it in the same place and the same width. There is no reason to
assume that. Putting `ludsa_version_major` at offset 0 commits two octets and
leaves everything else, including whether there is a length prefix at all, to
the version that declares itself.

**Why a request and a response have the same shape.** The alternative is to
give a response a status where a request carries its type. That leaves a
response unreadable on its own: the type selects the schema, so a reader has to
remember what was sent before it can parse what came back. A client always can,
which is why the arrangement survives, but `ludsa-probe`, a packet capture and a
log cannot.

**Why a request type and its response type are separate message types.** Sharing
one code between them keeps direction in the schema lookup, and a message whose
schema needs direction is not self-describing after all. Separating them is what
makes `msg_type` sufficient on its own.

Two rules then stop needing to be written. A response of the wrong type is a
`msg_type` the request does not permit, rather than a special comparison against
what was sent. And a response type arriving as a request is a type the server
does not accept, rather than a case about direction.

§6's two tables are frozen together by §4.1. Which types exist is no use without
which responses each request permits, and a later version free to add a response
type to an existing request could turn a client's rejection into an
interoperability failure.

**Why the `msg_type` codes carry no structure.** Interleaving each request with
its response makes the table readable, and reading a pattern into it is the
hazard. A code space where a response is the request with a bit set invites
`if (type & 0x80)`, which is the same decomposition §3.3 forbids for a tag and
for the same reason: the value means what the table says it means, and a later
version may assign one that fits no pattern.

It also leaves the schema of a failed response unstated. A status of
`INTERNAL_ERROR` is not a message type, so nothing says which fields such a
response may carry, and `error_detail` ends up assigned within AUTHENTICATE
alone. A failed PING then carries a reason its client is required to skip.

Making the status a message type answers both. `ERROR` is `msg_type` `0x00`, it
has a schema like every other type, and a receiver selects that schema the same
way it selects any other. Nothing binds two header words together, so there is
no invariant to police, and a status the peer does not recognise reaches the
same §3.4 rule as any other unrecognised enumerated value.

`0x00` because `ERROR` is not a request type. It sits outside the `0x01` to
`0x06` run of request and response pairs rather than in the middle of it, `0x07`
onward stays free for a later pair, and a message of nothing but zeroes reads as
an error rather than as anything affirmative. §3.4 later generalised that last
property to every enumeration.

**Why a response echoes the major version but not the minor.** A server that
implements several major versions must answer in the one it was addressed in.
Answering in its own preferred major version would fail an exchange in which
both sides understood each other. The minor version is the opposite case: its
purpose is to report what this server supports, so echoing the client's would
report nothing.

**Why the length bounds interpretation rather than reading.** Stated as a rule
about reading, it forbids the buffered reader every implementation uses, and it
makes §2's excess-octet check unimplementable: a receiver cannot observe an
octet it may not read. Stated as a rule about interpretation, both work. What
matters is that nothing past the declared length reaches a decision, not how
many octets happened to arrive in one call.

**Why a server drains at all.** Closing straight after the response leaves a
client that is still writing to meet `EPIPE` on its next write, and `SIGPIPE`
with it where the client has not suppressed it. The response itself is never at
risk: on a unix socket what the server already wrote sits in the client's
receive queue and survives the close, and no close discards it. What a Linux
client may see after the response is `ECONNRESET` in place of end of file,
wherever the server closed holding octets it had not read, which is why nothing
has a client read on for the end.

The window is not the socket buffer. A client that issues one write of a legal
request fits it in the buffer and never blocks, whatever the server does. A
client that writes the envelope and the body separately has a gap between the
two, and no buffer size closes it.

**Why a refusal reached at accept owes nothing.** §3's obligation follows what
the closing side owes the writer. A server that rejected a message had asked for
that request, so cutting the peer off mid-write is its own doing. A server that
refused the connection asked for nothing, and the ground does not reach it.

The tenth round found the document saying that in one place and implying the
opposite in another. §5 had a concurrency decline reached at accept, "so nothing
has asked for anything yet". §2 reached "owes no drain" by way of the deadline
being a drain terminator, which is a reason that presupposes a drain. §3's own
lead-in held the answer attached to the wrong sentence. The clause about a
server shedding load hung off the credentials case, so a reader looking for the
load-shedding rule could find it only by misreading the one beside it. Both
reviewers who raised this read it that way.

What the wrong reading cost is not ambiguity alone. Draining a refused
connection means reading to `2 x MAX_LENGTH` on the path where the server has
least to spend. A peer opening connections in bulk is what produces the load in
the first place, so the reading that owed a drain turned the refusal into the
amplifier it exists to prevent.

The rule now sits once, in §3's lead-in, with §2 and §5 pointing at it. Reading
the version word to choose an encoding does not attach the obligation on either
ground, since the refusal was settled before the read.

**Why a rejection on excess octets sits outside the drain obligation.** The
obligation binds a server that rejects before it has read the whole message, and
a server rejecting on excess octets has read all of it. §3's principle says a
server that rejected a message it asked for drains, so the case looked like an
omission on the wrong side of it. It is not: the peer still writing there is
writing octets the protocol forbids it to send, so nothing the server asked for
is being cut off. The permission to discard stays, since it costs a server
little and spares a broken client an `EPIPE`.

**Why it is required rather than encouraged.** The protocol creates the
condition. §2 has a server close after it responds, and §3 has it respond before
it has read the request, so the specification itself arranges for a client to be
writing into a socket the server is about to close. A hazard the protocol
manufactures is one the protocol should mitigate, rather than leaving each
server to choose and each client to guess.

A weaker rule would give a client the worst of both. It would still have to
handle `EPIPE`, since a SHOULD guarantees nothing, and it would gain no case it
could rely on. The value of the drain is that the ordinary path stays ordinary:
the client's write completes, it half-closes, it reads. Making that conditional
on the server it happens to meet buys nothing and costs predictability.

`VERSION_UNSUPPORTED` is the case that settles it. A newer client meets an older
server, neither is at fault, and it is also the exchange where the server can
least afford to guess at sizes, for the reason the count below gives.

Note what the rule does not remove. The drain narrows the window rather than
closing it, because the server closes when the drain ends, so a client must
still handle `EPIPE` and must still read the response after a failed write.

Both of those are `IMPLEMENTATION.md` guidance and neither is normative, by
decision rather than by omission. How a client deals with its host application
is between the two of them, and no peer can observe it. The protocol's job there
is to make the safe implementation the easy one, which is what the guidance and
the suggested client codes are for.

**How a MUST nobody can see gets tested.** The response is the same whether a
server drains or closes, so the drain looked untestable. The socket is not the
same, though, and a driver can probe it: write again after the response arrives,
and a draining server consumes it while a closed one gives `EPIPE`.

`VERSION_UNSUPPORTED` is the exchange to do it in. §3 already puts the declared
length out of play there, so the case tests the drain rather than which length a
server may rely on.

**Why stopping at the declared length is a permission and not a terminator.**
It sat in the list of things the drain stops at, headed "the earliest of these",
and the guidance told a server it could ignore it. Both reviewers reached that
bullet in the ninth round from opposite sides: one that its counting basis was
unstated where the bullet beside it stated one, the other that the guidance
contradicted a MUST.

The guidance was right and the rule was wrong. The drain exists to keep a peer
from being cut off mid-write, so reading longer serves its purpose rather than
defeating it, and three bounds already stop it: end of file, `2 x MAX_LENGTH`,
and §2's deadline. The declared length adds no bound the drain needs. It saves a
server some time on the paths where it may rely on the number, and a list headed
"the earliest of these" was the wrong place to say so.

Stating the counting basis costs a clause and settles the other half. Once
stopping early is optional, a server reading either way conforms and both are
bounded, so the ambiguity stops mattering; a server that does stop still needs
to know when.

**Why that phrase now has a definition.** It governed when a drain may stop
early and appeared three times unexplained, with one worked instance. The eighth
round asked the question it could not answer: is a length that failed the bounds
check one a server may rely on? A client declaring 3 octets and then writing
four kilobytes draws a three-octet discard and a close from one server and a
full drain from another, and the first leaves a peer taking `EPIPE` mid-write,
which is the one thing the drain exists to prevent.

So a length is worth acting on where the server read it under a major version it
implements and where it passed the bounds check. A number that failed that check
is the sender's own fault reported back as a fact, and honouring it turns the
drain against its purpose. The driver is a version 1
implementation claiming a later major version, so the answer comes back in a
version it can parse.

What the case cannot settle is a server whose drain bound is shorter than the
round trip. It stopped draining legitimately and looks exactly like one that
never started, which is why the bound is a fixture rather than an assumption.

Nor can it separate a draining server from one that shut down its write
direction and never read: both accept the driver's write, since a kilobyte fits
in a receive buffer nobody is emptying. What the case detects is the close, and
the close is the hazard, because it is the one that reaches a client
mid-credential as `EPIPE`.

**Why only a server drains, and why the asymmetry is not an oversight.** §3's
read-exactly rule used to bind every receiver with no exception, while the drain
had a server stop short of the declared length and §4 let any receiver report a
fault without waiting for the rest to arrive. A client that found a fault at the
twentieth octet of a response declaring thirty-two kilobytes had to breach one
rule or the other. The ninth round scoped the rule to a receiver that is parsing
the message, which is what it was always about: do not interpret past the
declared length, and do not under-read and take the next message's octets for
this one's. A receiver that has rejected is not parsing.

What relieves the two sides differs, and it should stay that way. A server
drains because the client may still be writing a credential of up to sixteen
kilobytes and would be cut off mid-write. A client rejecting a response has
nothing of the sort to protect: the server has already written its answer, and
§2 has a response that was written survive the close. The most a server takes is
`EPIPE` on a write it had not finished, which is exactly the cost the drain
spends effort sparing a client. So the client gets a permission to close and no
obligation to drain, and a later round proposing to give it one should read this
paragraph first.

The ninth round added a third grading, and a lead-in above the drain rule naming
all of them. The same purpose was being stated in three places at three
strengths, and three statements are three chances to drift. A server that
rejected a message drains. A server that refused the connection owes nothing and
may read anyway. A client owes nothing. What the lead-in does not do is restate
the MUST, which stays in one place below it. The tenth round widened the middle
grading from the credentials refusal alone to any refusal reached at accept,
which is the entry above.

**Why the count is twice `MAX_LENGTH`.** §4.1 freezes `MAX_LENGTH` within major
version 1, but a later major version may raise it, and the exchange where that
lands is `VERSION_UNSUPPORTED`. A server answering it has read only
`ludsa_version_major`. §3 puts everything after that word inside the major
version it declares, including whether there is a length prefix at all, so the
server knows neither the message's length nor its shape. It cannot bound the
drain by anything the message says, and the octet count is all it has.

A budget of one `MAX_LENGTH` would cut off a client of a later major version
part-way through a request that version permits. That client would take `EPIPE`
on the one exchange whose purpose is to tell it to upgrade, and would lose the
answer explaining why. Doubling leaves room for a larger message from a version
this server does not implement.

The bound still has to exist, because an unbounded drain is a peer-controlled
read with no ceiling. A sender above twice this version's ceiling is cut off,
and that is the compromise the constant states: generous enough for a later
major version, bounded enough that nothing holds the connection open for ever.

**Why §2's deadline bounds the drain in time.** §3 used to say the drain runs
after the response is written, so neither deadline in §2 is running by then, and
to require a server to set a bound of its own. Half of that was true. The
response-write deadline has finished; the accept-to-complete-request deadline
has not, because on this path the request never completes. So §2's deadline was
running, and §2 says an expiry means answering `REQUEST_TIMEOUT`, which on a
connection already answered is an instruction no server can carry out.

Letting it run and saying what its expiry means is better than ending it. One
number then bounds the whole connection from accept, rather than a server being
held for a request deadline and a drain bound in succession, and the guarantee
that the drain terminates comes from a deadline §2 already requires rather than
from a knob §3 had to insist on. Its expiry during a drain means close: the
status was settled when the message was rejected, and a server sends one
response to a connection.

A server's own drain bound survives as a MAY. It never carried the termination
guarantee, and what it does carry is a server's freedom to stop draining early,
which §7 concedes in F4 and cannot test for.

**Why an interrupted read is not the error that ends a drain.** The terminator
said "an error on the read direction", which a C server takes from `EINTR`
inside a host that installs handlers without `SA_RESTART`, and a non-blocking
one takes from `EAGAIN`. Neither says the peer has stopped writing, so a server
stopping there closes on the client the drain exists to protect, while
conforming to the letter. The terminator now names an error that ends the read,
and excludes the two that do not.

**Why a rejection whose terminator is already met drains nothing.** That follows
from the rule rather than being chosen, and it is worth stating because it looks
like an omission. A rejection reached while every terminator is still unmet owes
a drain and the remaining budget pays for it. The timeout path is settled *by*
the deadline expiring, so the budget is spent and the earliest terminator is met
before the response goes out.

This used to name `REQUEST_TIMEOUT` as the only such rejection, and the tenth
round found that false. `REQUEST_INCOMPLETE` covers a message that ended before
its declared length, which is to say a message that ended at end of file, and
end of file is the first terminator §3 lists. So that path writes and closes
discarding nothing on exactly the same ground. Naming the property instead of
counting the instances is what stops the sentence going stale the next time a
terminator is reachable early.

The client is not left without an answer. A response already written sits in the
peer's receive queue and survives the close, so what a slow client loses is only
the `EPIPE` it would have been spared. Spending a second deadline to spare it
would give the one peer that already used a whole deadline another, which is the
exchange with the least claim to it.

Go arrives at the right behaviour by accident here, because a fired deadline
latches the direction and its drain call reads nothing. The seventh round
reported that as a defect, and under the rule it is not one.
`IMPLEMENTATION.md` says so, because an implementer reading only the Go
behaviour would reset the deadline and reintroduce the double-holding.

**Why nothing is allocated from a wire value before it is checked.** Every
length and count in a message is a number a peer chose. Sizing anything from one
before bounding it against the structure that contains it is where a
memory-safety bug in this protocol would live, and it is one rule rather than
four because the hazard is identical for the message length, a field's length, a
map's entry count and an entry's key and value lengths.

**Why the allocation rule names the discard rather than the body.** It used to
forbid allocating storage for the body, which a server that checked the declared
length against `MAX_LENGTH` and then allocated has already done before it
rejects anything. Whether that breached the rule turned on reading it as being
about timing or about the drain, and no case can observe either. Saying what it
may not allocate for, the octets it is about to throw away, settles it in the
direction the rule was always about.

The fourteenth round added the offset to the same sentence, because the offset
is reached first. A check written as `off + len > buflen` wraps where the two
are the same width, passes, and the read then runs off the end, with nothing
allocated at any point. The rule was always about arithmetic on a length a peer
chose, and it had named only the second thing that arithmetic reaches.

### 3.1 Field maxima

**Why field maxima are in the protocol at all.** A whole-message bound is not
sufficient. Implementations receive into fixed-size storage, so a field limit
must be part of the protocol rather than a private choice. Otherwise a
conforming server may emit a value that a conforming client is obliged to
reject. That is an interoperability failure, not a protocol error, although it
presents as one.

Every dynamic field therefore has a maximum, the diagnostic ones included. A
field without one leaves a C client sizing a buffer with no sanctioned number to
use. Fixed-width fields need none, because their width is in the tag.

**Why `MAX_LENGTH` is not configurable downwards.** A server free to accept less
can answer `REQUEST_TOO_LARGE` to a message this document permits, which makes
"legal" and "acceptable" different predicates with nothing bridging them. PING
carries no field advertising the difference, and `server_desc` must not be
parsed, so a client cannot discover it and has no recourse if it could.

A floor short of the maximum does not help. §4.1 lets a minor version add
optional fields, so the largest legal message grows within major version 1 while
any floor computed today does not. Under version 1.0 the largest AUTHENTICATE
request is 17520 octets against a constant of 32768, and that headroom is the
room for those later additions.

What a server gives up is small. The number that threatens it is the accepted
maximum multiplied by its concurrency limit, and it still sets the concurrency
limit. The deadline ruling does not carry across: a server refusing an unbounded
wait defends against something it cannot predict, and a server refusing 32KB per
connection in flight does not.

**Why the constant bounds the declared length rather than the message.** Named
for the message it would exclude the eight envelope octets from nothing, and a
buffer sized from it would overflow by exactly that much. Bounding `length`
alone makes the constant the size of the buffer a receiver needs, and settles
whether a declared length of exactly 32768 is legal, which a size named for
something it is not measured against did not.

**Why `authn_id` is `utf8` where `user` is `string`.** The two look alike and
arrive from opposite directions. `user` comes off PostgreSQL's startup packet as
bytes, under an encoding that predates `client_encoding` negotiation, so
requiring an encoding of it rejects legal input. `authn_id` is constructed by
the server, from a backend that already knows what it holds, so an encoding can
be required of it.

That matters because of where the value goes. A PAM module logs the resolved
identity on success, which makes `authn_id` the one server-produced string a
client writes to a host's authentication log without any stated protection. A
newline in it forges a line there. Every other string a client logs carries a
producer obligation, and this one carrying none read as an oversight rather than
a decision.

**Why a defect in the three server-sent strings kills the response.** There was
an exception here until the fourth review round. An over-long `error_detail` or
`server_desc` was treated as absent, on the principle that a diagnostic a
receiver cannot use must not cost it the diagnosis, while an over-long
`authn_id` rejected the message because a client may act on it.

That split did not survive contact with the other half of the same rule. §5
already had a receiver treat an `authn_id` with a control character as absent,
which left a client holding the same `OK` and the same nothing. The graceful
path went to the more suspicious input, and one field's handling was split
across two sections that disagreed.

The rule now covers every one of those fields and every defect. A server is
bound to emit none of them, so a client meeting one has found a bug rather than
a value to work around. Reporting it is what gets the bug fixed, and a defect
that is quietly absorbed is a defect nobody fixes. What a client does beyond
reporting is its own business, and the specification does not say.

The cost is real and was accepted. A server emitting a bad `authn_id` fails
every authentication against a client that reads the field, grants included.
`pam_ludsa` is such a client, because it logs the resolved identity.

**Why a value must not be truncated to fit.** Shortening `authn_id` merges two
principals into one, the same collision hazard as truncating a role name, and
silently grants whatever the shortened identity maps to.

**Why a server that cannot carry a value answers `INTERNAL_ERROR`.** §3.1 bound
both parties against emitting an over-long field and against truncating one. It
then gave the escape hatch to the client alone. §5 gave the server one instance,
for `error_detail`. That works because a server composes the value and can embed
less of it. `authn_id` has no such route: the identity is the value.

The eleventh round gave the rule a case. Every other new MUST of the last two
rounds had one, or a note saying why not, and this one is driveable. F3
supplies, where a backend can, a credential whose identity is longer than the
maximum, and the case expects `INTERNAL_ERROR`. A truncating server and an
omitting server each answer `OK`, in shapes the driver can tell apart. Most
backends will take the concession, since a Kubernetes service-account identity
cannot reach the maximum, and a flat-file server is the likely one to run it.
The alternative, a note that nothing a driver sends reaches the rule, would have
been false.

The tenth round first proposed letting the server omit the field, on the ground
that every field a server sends is optional in its schema. The author overruled
it, and the distinction is the one that matters here. Optional in the schema
says only that some deployments extract no identity. It says nothing about a
deployment that does, whose client is waiting for one. Such a server, omitting
the field because it did not fit, tells that client nothing it can act on. §6.2
leaves a client that sent no `user` unable to grant on a bare `OK`. The
deployment fails and nothing says why.

The permission to omit is therefore about having no value to send, never about
having one that will not fit. The rule says so outright, since leaving it
implied would ask a reader to work out which sense a MAY carries.

No maximum here is set anywhere near an ordinary value, so a server meeting one
has a fault rather than an answer. That is what `INTERNAL_ERROR` names: waiting
will not change it and someone needs to look. Appendix B sends it to
`PAM_SYSTEM_ERR`, the code that fetches a person.

**Why 512 for `authn_id`.** A Kubernetes identity is `system:serviceaccount:`
plus a namespace of at most 63 octets and an account name of at most 253, so
339.

**Why `credential` is capped below PostgreSQL's own limit.** PostgreSQL allows
65535 (`PG_MAX_AUTH_TOKEN_LENGTH`, enforced on the PasswordMessage). LUDSA caps
it at 16384, so there is a band from 16385 to 65535 that PostgreSQL will deliver
and LUDSA will refuse.

The cap is set from what credentials actually are, not from what the framing
allows. The harness measured a Kubernetes service-account token at 1170 octets
on 2026-08-12, so 16384 is fourteen times the target. It is also double the
practical ceiling the token ecosystem designs against: no RFC bounds a JWT, but
nginx, Apache httpd and Tomcat all cap a header at roughly 8KB by default, so
anything meant to travel as a bearer token is built to sit under that with
margin. Node's 16KB default does not lift the ceiling, because a deployment has
to clear the lowest limit in the path.

Nothing common comes close to the cap. Entra ID access tokens are the largest
in ordinary use at 1 to 4KB, and Entra drops the groups claim past roughly two
hundred groups rather than emit a larger token. The credentials that genuinely
exceed 16384 are not JWTs at all: a Kerberos ticket carrying a large PAC is the
real instance, and Windows defaults `MaxTokenSize` to 48000.

The band is therefore theoretical for the deployments this protocol targets, but
it is better as a stated limit than a discovered one, which is why the refusal
must name the bound it hit.

### 3.2 Deterministic encoding

The conformance vectors compare byte-exact, so ascending order and at-most-once
tags are load-bearing. An encoder that violates either still round-trips against
its own decoder, and only the vectors catch it.

**Why at-most-once belongs to the message type and not to the framing.** It
looks like a framing rule and was one until the fifth round. It is not: whether
a tag may repeat is a property of the message type's schema, and holding that
schema is the only way a receiver knows the answer. A receiver that rejects a
repeat before it has read `msg_type` is asserting something about a message type
it may not implement, which breaks §4.1's promise that a new message type is a
minor bump.

No version 1.0 message type permits a repeat, but that is a fact about the
schemas this version defines rather than about the framing. §4.1 freezes only
the loosening: a tag assigned as non-repeatable stays that way, because a peer
that predates the change holds a schema for the tag and would meet duplicates it
has no rule for. A later minor version may assign a field that is repeatable
from the start, which a predating peer never adjudicates at all, because it does
not recognise the tag.

That promise had already been rescued once, when tag recognition and
required-field presence moved after `msg_type` for the same reason. The repeat
check was the third case and was missed both times.

Ascending order stays in the framing pass, because it binds every message
whatever the type. What moved is the strictness. A receiver still compares each
tag against the one before while scanning, and still may not build a map first,
since a map loses the ordering §7 case 6 requires. It just cannot reject on
equality until it knows the type. Ordering is a property of the encoding and not
of the decoded message, so a decoder that has verified it may hand fields to an
application in any form it likes.

**Why a sender emits no tag its version does not define.** The rule went in in
the seventh round. Until then §5 forbade it for `ERROR` and nothing forbade it
anywhere else, so a version 1.0 sender padding an AUTHENTICATE request with an
unassigned tag was arguably conforming, and a driver had an asymmetry to
hard-code with no reason behind it.

Nothing breaks at the time. Every receiver skips an unrecognised optional tag,
which is what makes the padding invisible. What breaks is later: §4.1 lets a
minor version assign any tag number not yet used, so the padding a 1.0 peer
emits today is a field some 1.1 peer reads tomorrow, and it reads it as a value
its schema says is there. The sender is long deployed by then and the fault
presents in the new implementation.

It is the emission-side counterpart of never reusing a field number. Both rules
exist because a tag that once meant nothing must not quietly start meaning
something, and neither failure appears in any test the implementation that
causes it can run.

The rule binds a sender to its own version rather than to 1.0, which is what
lets a later version emit at all, and §5's `ERROR` sentence became an instance
of it rather than the only statement of it.

**What a receiver may do about a breach.** The question arrived the day the rule
went in: should a receiver that knows the sending version reject a tag that
version does not define? The first answer was no, on three grounds, and it was
too tight. The rule now turns on the tag's field number against that message
type's ceiling (§3.3): at or below it the receiver MUST reject, and above it the
receiver skips, with a MAY to reject instead where it implements the declared
version.

The three grounds, and what happened to them. *A receiver could apply the check
only against a version it already implements, so it polices some peers and not
others.* That is a statement about coverage rather than about correctness. A
check that fires where it can be sure and declines where it cannot is ordinary,
and the case it declines is exactly the case where the receiver has no schema to
judge with. *It turns a skipped tag into a failed authentication.* Under a MAY
it does so only where a deployment chose that, and forbidding a deployment to be
strict about input in an authentication path is a strange thing for this
protocol to do. *It makes `ludsa_version_minor` load-bearing for parsing.* It
does not. Parsing is still by tag: the version settles whether the sender had
any business emitting the tag, not what the tag means or how to read it.

What the MAY does not touch is the case that matters for evolution. A receiver
meeting a minor version it does not implement MUST skip, because §4.1 leaves a
later version free to define any tag not yet assigned, so a receiver rejecting
there would fail a conforming peer. That is the whole of skip-unknown's job and
it is untouched.

Neither outcome is preferred, so §7 excludes it as a case rather than choosing,
on the same ground as a message both truncated and faulty: the protocol has no
stake in the answer and the input comes only from a sender that is already
broken.

The other direction needs no rule of its own. A peer implementing a later
version may receive a message declaring an earlier one that carries a tag the
later version defines. It recognises the tag, so §3.3 has it read the field.
Parsing by tag answers that without a second rule.

**Why absence and emptiness are different.** An earlier wording told an emitter
never to send a field empty, which conflated *I have no value for this* with *my
value is zero octets*. The first is an absent optional field, which is omitted.
The second is a value, and whether it is acceptable is a question about
credentials and role names rather than about framing. An empty `user` reaches
the policy layer as a name like any other, so it comes back a denial (§6.2.1),
whether the server validated the credential first or rejected the name before
looking. Which code names that denial turns on whether the empty name resolves
for that deployment. An empty credential fails
validation for any real backend. Both fail closed without the protocol having an
opinion about the value itself.

### 3.3 Tags

**Why the tag encodes its own handling.** A receiver has to apply the right rule
to a tag it has never seen, which rules out carrying the answer beside the tag:
it would have to trust a value from the same message it is deciding about.
Deriving the rule from the tag itself makes it available before anything else is
parsed, and the sender cannot misstate it without producing a different tag.

The alternative, and the shape this replaced, was a uniform `u16 tag, u32 len`
on every field with the type recorded only in this document. That works for
skipping, but it leaves the type unenforceable, and two independent reviews of
the earlier draft both found the same two consequences first: a rule forbidding
NUL octets "in any field" made every integer field unencodable, since `result =
OK` is four zero bytes; and nothing said a four-octet field must arrive with a
length of four, so a short `result` had no defined handling on the path that
carries the verdict. Both were prose scoped by field name, which a decoder
written from the tag table never sees.

**Why an unassigned type is skipped but a contradictory tag is not.** Both are
tags a receiver cannot fully account for, and they fail differently. Bits 5-4
set while bit 6 is set, or a type whose table entry says dynamic on a tag whose
bit 6 is clear, describe a field that cannot exist. Rejecting those is what
keeps a tag comparable as one 16-bit value, which §3.2's ordering check depends
on.

A type value the document has not assigned is not impossible, only unfamiliar,
and rejecting it would break the promise §4.1 makes about an optional field a
later version adds. Skipping needs bit 6 and, when clear, bits 5-4, and never
the type, so a receiver can skip a field of a type it has never heard of. The
consistency check cannot reach an unassigned type for the same reason it is safe
to skip one: nothing says whether it is fixed or dynamic.

**Why the high-octet layout needed a conformance case to defend it.** Moving the
field number into the high octet made tags sort by number, which is what the
sixth round wanted, and it made a number-keyed decoder look natural for the
first time. §6.4's table now reads as though the number were the identifier. The
rule that it is not did not change, and nothing drove it.

Two tags sharing a number are two different tags, and the framing pass accepts
both, since they ascend. A decoder keyed on the number puts one field's value
where another belongs. No conforming peer sends that, because §4.1 makes a later
version take a higher number than any already assigned in the message type, so
the construction is a broken or hostile peer's. That is enough to want a case:
the numbering in §6.4 is a defence against exactly this confusion, and a defence
nothing exercises is a defence nobody has checked.

**Why the whole tag is validated rather than its parts.** A tag is an identifier
that happens to have structure, not a field number with metadata attached. A
sender that labels `user` as an integer has not sent `user` with a wrong type;
it has sent a different tag, which is unrecognised, and the required bit decides
what happens. There is no third case in which a receiver holds a known field and
a wrong description of it, so there is nothing to cross-check and no opportunity
to consult the wire instead of the schema.

The rule named parsing "according to the type or width the tag claims", and the
skip two paragraphs below requires skipping by exactly the width the tag claims.
Both readings of parsing are available and they instruct oppositely. The
prohibition is about the value: a receiver never reads one out of a field whose
tag it cannot place. The width is what it skips by, and skipping is what it does
instead of reading.

**Why width is encoded and not just fixed-versus-dynamic.** An unknown optional
field of a fixed type would otherwise be unskippable: the receiver knows there
is no length prefix but not how many octets to step over. "Optional" would then
stop meaning skippable for any type added later, which is the guarantee the
scheme rests on.

**Why the parity rule is gone.** Odd tags being critical and even tags skippable
was an encoding of required-ness, described as though it were the concept. The
concept is that a field either must be understood or may be ignored, and the tag
now says so directly.

**Why the distinction exists at all.** Skip-unknown is a fail-open evolution
mode, which is unacceptable in an authentication protocol. If a future version
adds a field meaning "the audience check was not performed", an older peer that
skipped it would read a qualified answer as an unqualified approval. Such a
field is made required, and old peers are forced to refuse rather than
misinterpret.

**Why `bytes` does not forbid NUL and `string` does.** The hazard was always
asymmetric. A NUL in `user` truncating on one side and not the other is
role-name confusion, and the truncating side approves a role nobody asked for. A
NUL in the credential truncating produces a signature that does not verify:
fail-closed, and a validation failure rather than a privilege grant. So the rule
belongs to the type that names things, not to the type that carries opaque
octets.

That also gives the two types a difference in behaviour rather than only in
documentation. Every C implementation handles a name as a string at some point,
so an embedded NUL makes such a field mean two different things depending on
which side reads it. PostgreSQL already guarantees the credential is NUL-free
anyway, since `recv_password_packet` rejects a packet whose content is shorter
than its declared length.

**Why `utf8` is a type rather than a rule about particular fields.** The
obligation applies to values a server originates: `error_detail`, `server_desc`
and `authn_id`. It does not apply to values that pass through the server,
because a role name arrives from PostgreSQL's startup packet as bytes and no
encoding can be known, the startup packet predating `client_encoding`
negotiation. Requiring UTF-8 there would mean either rejecting legal role names
or transcoding with knowledge nobody has.

Making it a type rather than three separate sentences means a decoder enforces
it from the tag, and a field added later inherits the rule by being given the
type. `server_desc` gained the obligation that way, having gone without one
while it was prose.

**Prior art.** Encoding the unknown-handling rule in the identifier is a known
technique, although this arrangement of bits is particular to this protocol. Two
protocols do the same thing:

- **IPv6 option types**, RFC 8200 §4.2. The highest-order two bits of the option
  type "specify the action that must be taken if the processing IPv6 node does
  not recognize the Option Type": skip, discard, or discard and send ICMP. Four
  outcomes rather than two, chosen by the identifier.
- **PNG chunk types.** The case of the first letter of the chunk name is the
  ancillary bit, where uppercase means critical and lowercase means ancillary. A
  decoder meeting an unknown critical chunk "shall indicate to the user that the
  image contains information it cannot safely interpret", while an unknown
  ancillary chunk "can safely be ignored".

The same semantics carried by an explicit flag rather than the identifier appear
in X.509 extensions (RFC 5280 §4.2, `critical BOOLEAN`), the JWS `crit` header
(RFC 7515 §4.1.11) and BGP path attribute flags (RFC 4271 §4.3).

The counter-example is the behaviour most readers expect by default. Protocol
Buffers preserves and ignores every unknown field, unconditionally. That is the
fail-open mode this rule refuses, and it is one reason protobuf was not used for
framing.

### 3.4 Enumerated values

**Why every enumeration has the same shape.** The status, the auth result and
`serving` each needed a rule for a value a receiver does not recognise, and
three copies of one rule is three chances to write it differently. Stating it
once means an enumeration added by a later version has defined unknown-handling
without anyone remembering to give it one.

**Why an enumeration is one octet.** Three enumerations at three widths would
leave §3.4's opening claim true of the value assignments and false of the
encoding, and none of them will approach 255 values. One octet also makes the
three tags identical within their own groups, which §6.4 already permits and
which removes a difference carrying no information.

**Why zero is never affirmative.** A zero value is the one an implementation
produces by accident: Go's zero value, a `memset`, a map miss, a `switch` with
no default. For three rounds `OK` and `SERVING` sat at 0 anyway, on the ground
that the accident is a defect in one implementation and that inverting the
enumerations, putting the unrecognised case at 0 and `OK` at 1, taxes every
correct implementation forever.

That reasoning was answering a proposal nobody ended up making. Leaving 0
unassigned and putting `OK` at 2 keeps success as one specific value and failure
as everything else, which is the convention the decline was defending, and it
invites nobody to test against 0, because 0 means nothing. The argument against
`OK` at 1 never reached it.

What settled it was that the document had already accepted the argument once.
`status` leaves 0 unassigned so a zeroed `ERROR` fails closed, and no reason was
ever given why the verdict enumeration deserved less. The earlier reasoning was
also entirely receiver-side, where the accident that matters is a sender's: an
empty response struct in Go emits a structurally perfect approval, every §3.4
rule that would catch it addresses a receiver, and no conformance case can
provoke a server's internal-failure path.

The rule now holds across the protocol rather than in this section. Field number
0 names no field, `msg_type` `0x00` is `ERROR`, and major version 0 is
implemented by nobody, so a zero-filled buffer cannot express an approval
anywhere.

**Why value 1 is never sent.** It exists so an unrecognised value has a name
rather than a special case. A receiver substitutes it, and its own API can
expose that value like any other. Without it, each implementation invents a
private representation of the same condition and a caller has to handle a case
the type does not mention. A server never sends it, because a server that does
not know what went wrong has `INTERNAL_ERROR` for that.

**Why each enumeration names its own 1.** A single literal `UNKNOWN` shared by
three enumerations is unambiguous in a type system and ambiguous everywhere
else. A caller reads `STATUS_UNKNOWN`, `AUTHN_UNKNOWN` and `SERVING_UNKNOWN` in
one namespace, alongside the codes a client reports itself, and a bare `UNKNOWN`
in a log line says nothing about which of the three it was. §3.4 therefore fixes
the number and leaves each enumeration to name it, which is the right division:
the number is what travels.

**Why it sits at 1 rather than last.** Values added later append at the end, so
a sentinel at the end would move each time one was added. It is the one value
implementations pin into their own APIs, so it is the one that must not move.

**Why `SERVING` is about a new principal.** A server holding a verdict cache can
answer for principals it has already decided, and that is not readiness: the
traffic a readiness probe protects is the traffic the server has not seen. A
server validating against a previously downloaded key set is the other case. It
can check a credential it has never met, so it can serve a new principal, and
what it has lost is revocation rather than validation. Naming the new principal
separates the two, and a probe reading `serving` as liveness would keep routing
to the first.

**Why `serving` names its values rather than meaning zero and nonzero.** A field
where any nonzero value means yes can never gain a third state. A later version
adding one, for a server that is draining or degraded, would find every older
client already reading it as an unqualified yes.

## 4. Versioning

**Why the version splits into major and minor.** §3.3 gives the protocol two
tiers of change, a field an older peer may skip and one it must refuse, but a
single version integer can't express which tier a peer is at. Two u16 fields fit
in the four octets a single `u32` would have spent.

**Why the major version is the whole compatibility test.** §4.1 constrains major
version 1 only. A later major version may reassign a tag or change what a field
means, so recognising a tag under an unimplemented major version proves nothing
about what it means there. Tag-level recognition can't detect semantic
redefinition, and the misparse would fall on a verdict field.

Allowing a peer to parse a higher major version whenever it recognised the
contents would also collapse the major version into the required-field rule,
leaving two mechanisms for one job. It would forfeit the ability to redefine
anything, because no future major version could rely on being rejected. Keeping
the two distinct is what makes redefinition possible: a required field says a
single addition must not be ignored while the rest of the format still holds,
and a major version says the format itself changed.

**Why the status a server is required to send may be unreadable.** §4 has a
server answer `VERSION_UNSUPPORTED` in its own major version, and §3 forbids a
receiver from interpreting anything past a version word it does not implement.
The client the reply is aimed at is one whose version the server rejected, so it
cannot read the status. Only a client implementing the server's version can,
which means a multi-version client or a driver simulating another version.

Nothing is lost, because the conclusion travels twice. §3 fixes
`ludsa_version_major` at offset 0 of every message in every version, so a client
that cannot read the status still reads the version word and reports the
mismatch from that. The redundancy is the reason the version word's position is
the one thing frozen across major versions.

**Why later versions are asked to frame the answer in an older version.** The
redundancy above tells a client *that* the versions differ and nothing else. A
`VERSION_UNSUPPORTED` it can read carries a status, and in a later
version may carry more. The convention costs nothing to state now and cannot be
introduced later: if major version 2 ships without it, a version 2 server has no
version 1 encoder and the option is gone.

The alternative was a frame readable by any peer, independent of version. It
would have to be frozen across every future major version, which is the
commitment §3 holds down to two octets. Widening that to a whole
message buys a diagnostic and spends the freedom a major version exists to
provide.

**Why the minor version exists at all, given no verdict turns on it.** It has a
couple of uses:

- It puts version skew in a log for every authentication rather than only at
  startup, which matters because the two sides ship separately.
- It lets a server tailor what it sends. A later version that adds an optional
  diagnostic field can carry the same information to an older client in a field
  that client already understands, rather than dropping it.

An earlier draft claimed a third: that it lets a client tell an unsupported
field apart from a supported field that is absent. That cannot work. A field a
later version adds is optional, since a required one forces a major bump, and an
optional field may be absent because the peer had no value for it. Absence stays
ambiguous whatever version the peer declares, so no rule about what a declared
minor version implies could rescue the inference. The sixth round deleted the
claim rather than inventing the capability rule it needed, which would have been
a MUST no driver could test.

It must not inform a decision, for the same reason `pid` must not. It is a
peer-supplied value, and under *checked, never consulted* such a value may be
tested against configuration but never used to select behaviour.

**Why the field number, and not the declared version, decides what a receiver
does with a tag it does not recognise.** The declared version is peer-supplied,
so a rule turning on it makes the receiver's answer depend on the sender
reporting its own version correctly. The field number does not. §4.1 freezes a
number's assignment, and requires a later version to take a number higher than
every number already assigned. So a number at or below a receiver's ceiling for
that message type can never acquire a new meaning. A tag carrying such
a number, whose whole value is not that number's assignment, is one no version
defines, and a receiver can prove it without trusting anything the sender said.

That is what makes rejecting it safe. Skip-unknown is the fail-open evolution
mode and it earns its place only where the tag might be legitimate. Where it
cannot be, skipping ignores a peer the receiver has just proved broken, in an
authentication path.

The ninth round found §7 and §4 disagreeing here, §7 making the skip a MUST and
§4 making it one of two conforming answers, so a conforming server failed the
conformance suite. Deciding on the number settled the branch at or below the
ceiling, and made the case testable in a way neither reading had been: a
receiver now fails it by reading the value into the defined field, and fails it
by skipping the tag and parsing on.

It did not settle the branch above the ceiling, and the tenth round found the
two sections still disagreeing there. §4's MAY reaches a message declaring a
minor version the receiver implements, which is exactly what a driver declaring
its own version against a peer of that version constructs, so §7's unqualified
MUST failed a receiver taking the choice §4 gave it. §7 now drives the skip with
a declared minor version the receiver does not implement, which is the branch §4
leaves no choice in.

The eighth round's latitude survives where it means something. A field number
above the ceiling may belong to a later version, so a receiver skips an optional
tag carrying it. A required tag never qualifies, since a later version's
addition is optional. Where the message also declares a minor version the
receiver implements, the sender has broken §3.2 by carrying the tag at all, and
a deployment that wants strict input in an authentication path may reject on
that ground.

**Why "a minor version the receiver implements" needed defining.** The
skip-or-reject latitude turns on it, and §7's skip case turns on its complement,
so a driver and a receiver reading it differently disagree about a conforming
message. Read as "is at", a 1.2 receiver must skip a 1.1 message's unknown tag;
read as "at or below its own", it may reject. Neither reading is available from
the words. §4.1 makes each minor version's schema contain the ones before it, so
at or below is the reading the freeze already supports, and it is now stated.

The eleventh round found a fourth copy in §5. It told a receiver to skip any
unrecognised optional tag in an `ERROR` message, with no mention of the ceiling,
so an `ERROR` carrying an optional field numbered 1 was skipped under §5 and
rejected under §3.3. §5 now points at §3.3 and states nothing of its own.

**Why the pivot is a named quantity.** It was stated six times in the normative
text, in two phrasings that are not interchangeable: "above every number the
receiver's version assigns", and "at or below the highest number it assigns".
The first is vacuously true where a message type assigns no field and the second
has no value there, so the two answer a version 1.1 tag in a `PING_REQUEST`
oppositely, one skipping and one rejecting. Two of the six also switched
quantifier inside a single sentence, naming a set and then referring back to it
as "that number". Naming the quantity once and referring to it settles the empty
case in the definition rather than six times over, and the tenth round reached
it after three consecutive rulings on copies of this same rule drifting apart.
The definition says a type its version assigns no field in has a ceiling below
every field number, rather than giving that ceiling a value, because field
number 0 is reserved and a property carries the meaning a number would only
imply.

**Why no negotiation.** The two ends are not deployed together. Each declares
its own version, and §3.3 makes a minor difference safe without either side
agreeing to anything. Rejecting a major mismatch outright makes an incompatible
pair fail immediately and identically rather than misparse into a wrong
authentication decision.

**Why the version MUST names its exceptions.** §4 required `VERSION_UNSUPPORTED`
for an unimplemented major version without qualification, and §2 has a server at
its concurrency bound answer `RATE_LIMITED` to the same input. Two MUSTs on one
message is not something an implementer can resolve, and the step-order
paragraph beside it named only §5's peer refusal as displacing step 1. Both now
name the property: a refusal reached at accept was settled before the version
could be read, and nothing the version draws changes it.

**Why validation runs outermost first, with the earliest failure winning.**
Several checks can fail on one message, and without an order two conforming
servers answer differently, which leaves a hand-authored vector with two correct
expectations and no way to state one. The order the message is laid out is also
the order a decoder is written in, so the rule costs nothing to follow.

Framing before `msg_type` is the part worth stating. It means a server fully
parses a request it is about to refuse, which is cheap on a bounded message.

Waiting for the declared octets is the one step that holds up every other, and
the fifth round said so about framing before the sixth generalised it. A
receiver cannot know whether the octets will all arrive until they do, so
ordering arrival strictly first would make it defer every verdict to the end of
the message. §3 is built the other way: its drain protocol exists for nothing
but rejecting before the message has arrived, it requires the response to be
written before the drain begins, and a client's fixed receive buffer depends on
the same freedom. So a message both truncated and faulty in the octets that
arrived takes either status, and §7 excludes it as a case rather than choosing
one, on the same footing as a request that stalls before its version word.

**The seventh round chose one and the eighth put it back.** The case for
choosing was that the two statuses reach different PAM codes, so one stack falls
through where another stops on identical octets. What that left out is that a
rule in either direction conscripts a way of reading a socket. Making the fault
win obliges a receiver that reads a message whole to parse a prefix it has
already given up on. Making the truncation win obliges a receiver that validates
as it reads to wait for the rest of a message it has already judged. The
protocol has no stake in which of those a receiver is, and every reason not to
choose for it.

The stakes are also smaller than they looked. The input needs a peer broken
twice over, since one that merely died mid-write leaves a valid prefix and no
fault at all, and a peer that died is not reading the response whichever status
it gets. A hostile peer gains nothing either, since it can already pick between
the two by sending a clean short message or a complete bad one.

The round-seven entry claimed as well that the status varied between runs of one
implementation, on where a socket read split. It does not. A receiver that
validates as it reads reaches a fault before the end of file every time, and one
that reads first parses nothing either time. Each is deterministic on its own;
the divergence is between the two strategies, which is exactly what makes
conscripting one of them the whole cost of the rule.

The fifth round paired arrival with framing alone, which left the message type
as the one place a receiver holds a settled verdict and may not report it. A
server that has read a `msg_type` it does not implement had to go on waiting for
a declared seventeen kilobytes before it could say so, because a truncation
would have taken the status instead. Nothing about an unimplemented type becomes
truer by waiting, and an unimplemented major version is already answered off the
envelope alone.

The generalisation needs its own bound, or it would license an early verdict on
a field that has not arrived yet. Required-field presence bounds itself, since a
field is missing only once the message ends. The selectors, the limits and the
decision needed the bound stated as a rule. The twelfth round's wording gave a
reason that does not hold of them. A field number does not order the steps: a
`STATUS_REQUEST` carries `max_age` ahead of `service`, so the limit is legible
before the selector the order puts first. What keeps the permission off them is
that it covers a fault. §6.2.1 makes `SERVICE_UNSUPPORTED` an answer a server
gives, and `RATE_LIMITED` is a refusal. Those still wait.

What goes in that pass is only what holds without a schema, and the boundary
moved in the fifth round. It used to include the at-most-once rule, on the
reasoning that a newer client sending a later message type expects
`REQUEST_UNSUPPORTED` and has fallback logic for it, so answering
`REQUEST_UNSUPPORTED` to a message that is *also* corrupt lets a genuine encoder
bug be swallowed by that fallback.

That reasoning was sound and its premise was too narrow. Whether a tag may
repeat is a property of the message type's schema, so a receiver rejecting a
repeat in a type it does not implement is asserting something it holds no schema
to establish. §4.1 freezes the loosening of an assigned tag rather than
repeatability itself, so a later minor version may still assign a repeatable
field, and the question stays one only a schema answers.

The cost of the change is the swallowed encoder bug, which needs a client to be
using an unimplemented message type *and* to have a broken encoder. The cost of
keeping it was refusing a message on an assertion the receiver could not
make.

Only the framing a receiver can check without a schema goes there. Tag
recognition and required-field presence go after `msg_type`, because §3.3
matches a tag against the schema that `msg_type` selects, and an unimplemented
type has no schema to match against. Placing them before it made every tag
unrecognised, so a message type carrying any required field drew
`REQUEST_MALFORMED`, and §4.1's promise that a new message type is a minor bump
held only for a type with no required fields. Nothing worth adding has none.
`REQUEST_MALFORMED` is also the wrong signal for the client to act on: it cannot
tell an old server from its own encoder bug, which is the distinction the
promise depends on.

### 4.1 What a later version may change

**Why the freeze is one line about tags.** A tag's number, type, width and
required status all live in its value, so forbidding each separately would be
four ways of saying the same thing. Widening a field or making an optional field
required produces a different tag, which adds a field beside one that stays and
forbids nothing.

**Why field number 0 is reserved.** The ordering check compares each tag against
the one before it, with zero as the value before the first. §3.3's shape checks
reject a tag whose field number is zero before any of that runs, and reserving
the number is what makes zero safe as that initial value. A tag of `0x0000` is
otherwise perfectly legal, being an optional fixed-width one-octet `integer`
numbered 0.

The ordering check alone would not stop it. §4 compares for no tag lower than
the tag before it rather than for a strictly higher one, because §4.1 lets a
later minor version assign a repeatable field whose occurrences carry equal
tags. `0x0000` in first position is not lower than zero, so it passes. The shape
check is what rejects it, and the reservation is what lets the sentinel be zero
at all.

The failure would not appear for years. No version 1.0 tag is `0x0000`, so every
vector passes; the break arrives with the first minor version to use the number,
against every peer already deployed, and reads as a fault in the new
implementation rather than in the old ones. Spending one number of 256 removes
the possibility.

**Why a field number is never reused.** Reusing a number for a differently-typed
successor makes every log line and packet dump ambiguous about which field it
meant. Numbers are not scarce at 255 per message type, and a lifetime budget
that large will not bind.

The sixth round made the gaps safe rather than merely documented. A later
version must take a number higher than every number already assigned in its
message type (§4.1), so a skipped number is closed by the same rule that keeps a
new field sorting after every existing one, and the next author cannot see a gap
and take it.

The eleventh round found that retirement could not survive the ceiling. §4.1 let
a later minor version remove a field and retire its number, and the ceiling rule
treats an unassigned number below the ceiling as proof the sender is broken, so
a peer at the version that assigned a field, sending it to a peer at the version
that retired it, was rejected for conforming. The fix removes the operation: a
number once assigned stays assigned in every later minor version, a later
version may deprecate a field but keeps accepting it, and there is nothing to
retire. Carving retirement out of the ceiling would have needed a retired-number
list, a second concept doing what an assigned field the receiver ignores already
does, carried by every receiver forever.

**Why a required field forces a major bump.** Every older peer rejects a message
carrying a required tag it does not recognise. Such a field is a new major
version in everything but name, and giving it that number costs nothing.

Note this means the required and optional rule is what makes the model safe, not
the version number. A required field shipped in a minor bump by mistake is still
rejected, so the mistake can't fail open.

**Why a later version may not spend the headroom past `MAX_LENGTH`.** §4.1
freezes the constant and every field maximum, and permits optional additions
without saying what bounds their total. Under 1.0 the largest AUTHENTICATE
request is 17520 octets against 32768, and that headroom is what the additions
spend. Nothing said they had to stop.

A version that overspent would define a message legal under its own schema and
rejected by every conforming receiver. That includes one implementing the same
version, so it is a version that cannot talk to itself. Both spec reviewers
reached this in the fourteenth round. It is cheap now and impossible after the
freeze.

## 5. The ERROR message

**Why a status is a message type rather than a header word.** §3 covers why a
request and a response have the same shape. What follows from it here is that
the schema of a failed response no longer needs stating separately: `ERROR` has
one like every other message type, and a receiver selects it the same way.

**Why there is one `ERROR` type rather than one per request type.** A failure
carries the same two things whatever was asked, and a server that never read the
request's type still has to report one. Deriving the type from a request it did
not parse is impossible for `VERSION_UNSUPPORTED`, where §3 forbids reading
further, and for `REQUEST_TOO_LARGE`, where the type lies in a body it refused
to read. A single type is answerable in every case, and needs no reserved value
standing for a type that was never read.

**Why the status has no `OK`.** It would be a value the only message carrying it
may never carry, which takes two rules to police: one forbidding a sender from
emitting it and one telling a receiver what to do with it anyway. Leaving 0
unassigned deletes both. It also makes a zeroed `ERROR` message fail closed
through §3.4, which is the rule the other two enumerations were later brought
into line with.

**Why the status leaves 2 unassigned as well.** §3.4 holds the affirmative
outcome at 2 in every enumeration that has one. The fifth round moved `OK` and
`SERVING` there without checking the enumerations against each other, which put
`REQUEST_MALFORMED` at the same number as both, so a client that read a status
out of a message it had misdispatched turned the commonest rejection into an
approval. Before that round it had read as a denial and failed closed.

Leaving 2 empty in the one enumeration with no affirmative case restores it and
states it as an invariant rather than an accident: no value that names a
non-affirmative outcome in one enumeration is affirmative in another. The cost
is one number of 255 in a namespace this sparsely used. §6.4 already spends
numbering on the same hazard, so a receiver that misdispatches meets two
defences rather than one, and neither is the `msg_type` check that should have
caught it first.

**The invariant was stated too widely for two rounds.** It read "no value
assigned in one enumeration is affirmative in another", which is false: 2 is
assigned in `result` and affirmative in `serving`, and the reverse. The
consequence drawn from it survives, since no refusal anywhere sits at 2 and 2 is
the only affirmative number, so a misdispatched refusal still cannot read as an
approval. What does not survive is the claim's reach. Two affirmatives share the
number, so a `serving` of 2 read as a `result` is `OK`, and what stops that is
`serving` and `result` holding different required tags rather than anything in
§3.4. The eighth round found it; `IMPLEMENTATION.md` had been stating the narrow
version correctly all along, which is why nothing was built on the wide one.

**Why two enumerations rather than one.** A status says how the server handled
the message. A result says what it decided. One enumeration would make every
code valid in both positions, and about half of them are meaningless in each: a
verdict of `REQUEST_MALFORMED` says nothing, and a handling outcome of
`AUTHN_DENIED_POLICY` says nothing. Separating them also lets a client read the
status first and learn whether a result is present at all.

**Why the statuses carry a direction and the response codes are separate.**
Every code in §5 is a server's statement about a request. A client never sends
an `ERROR`, so nothing else can ever appear there. §7 used to conscript the same
names for what a client found in a *response*, which gave one name two opposite
meanings and left a caller unable to act on either.

The `REQUEST_` prefix says what the enumeration always meant. It goes on the
codes that name a fault in the request, and not on those describing the server's
own state or the pair's compatibility, because those carry no direction to
state. `STATUS_UNKNOWN` belongs to neither family and no server emits it. A
naming scheme only some of the codes follow looks arbitrary until you notice the
rest are not about the request at all.

The response codes are deliberately not members of that enumeration. A server
can never send one, so giving them numbers would spend wire vocabulary on values
that are illegal in both directions. They are a client's contract with its
caller, they are a SHOULD, and a client with better names for its own host is
still conforming.

The names cover what a client can find. A response never started, or started and
stopped, or arrived and was wrong. The tenth round added a fourth shape, arrived
whole and never ended, and the eleventh removed it again when §2 stopped making
the end part of the message. `RESPONSE_ABSENT` and `RESPONSE_INCOMPLETE` stay
separate because their causes differ: nothing arriving is usually a refused peer
or a wrong socket path, and a truncated response is usually a crash. Both map to
the same PAM code, so the distinction earns its keep in a log rather than at the
stack. What separates either of them from `RESPONSE_TIMEOUT` is what ended the
wait, which is the entry above.

**Why `REQUEST_MALFORMED` is narrow.** It maps to a PAM code that tells an
operator to file a bug, so anything reported through it had better be a bug.
Truncation is not: a peer crashing mid-write, being killed, or losing its
connection all produce a short message, and only a sender that declared a length
it did not fill is at fault. Those causes are far more common than the bug, so
`REQUEST_INCOMPLETE` exists to keep the common case pointed at the deployment.

A field above its §3.1 maximum is `REQUEST_MALFORMED` all the same, and it did
not always read that way. The entry here used to argue the opposite, that the
realistic instance is a large `credential` a client passed through exactly as
its host handed it, so calling it an encoder defect sends an operator hunting a
codec fault that is not there. Round five removed the premise: §3.1 now has a
client asked to send an over-long field report `INVALID_REQUEST` to its host
without opening a connection. A server that receives one therefore has a peer
that sent what §3.1 forbids, which is a sender defect like any other, and §5
already said as much. The passage was one round five did not sweep.

**Why the type half got its own case.** §3.3 forbids a client to open a
connection for a value its field's type rejects, in the same breath §3.1 uses
for a value above its maximum, and §7 drove only the second. In version 1.0 the
constructible instance is a NUL in a `string`, and `user` is where it lands: a
client that truncates at the NUL and sends the prefix asks the server about a
different name than its host asked about. The case shares F8 and asserts what
the maximum case asserts, that the driver's socket saw no connection, which is
what a truncating client fails.

**Why `REQUEST_TOO_LARGE` is the envelope only.** Its value is that a receiver
reaches it from the declared length alone and answers without parsing further.
A field maximum cannot be checked that way: the receiver has already accepted
the envelope and read the body, so nothing is saved and nothing distinguishes
the failure from any other the body carries. Keeping the status to the envelope
makes it mean one thing, and the field case keeps its diagnosis in
`error_detail`, which is more use to an operator than a second status would be.

A separate `REQUEST_FIELD_TOO_LARGE` was weighed and not added. Both would map
to the same PAM code and neither would change what an operator does, so it would
buy a name and no behaviour. The limits are far above any expected value, so a
field over one is more likely a client bug than a legitimately large token, and
a client that sent it ignored a rule it could have checked itself.

**Why an unrecognised `service` has a name of its own.** It used to be
`REQUEST_UNSUPPORTED`, on the line that §5 says why no decision was reached and
§6.2.1 says what the decision was, so a disagreement about what exists belongs
to the first. That line broke on inspection in the seventh round. §5's statuses
describe how a server handled a message, and a server that recognises no such
`service` handled the message perfectly: it parsed it, accepted it, and found
that the name selects nothing it holds. What disagrees is two configurations,
neither of them faulty.

Folding it into the denials was the other candidate and is worse.
`AUTHN_DENIED_POLICY` asserts the credential was valid, which the server never
determined, so §6.2.1 forbids it outright. `AUTHN_DENIED` fits but says nothing,
and a bare denial for a misconfigured `service=` reports the wrong party at
fault. The user is denied for something no user chose, and every login
afterwards fails identically with nothing naming the line responsible.

So it is a status of the response rather than of the protocol, in whichever
enumeration the response type carries: `result` at 7 for AUTHENTICATE, `serving`
at 4 for STATUS. Appendix B maps it to `PAM_SYSTEM_ERR`, which is what keeps the
operator signal the old `REQUEST_UNSUPPORTED` gave.

**Why it is safe to name.** Every disclosure rule in §6.2 is about what a denial
tells the remote party. This tells it nothing: `service` is the one field §6.2
marks trusted, the remote party does not influence it, and the answer is about
the deployment rather than about the credential or the identity. There is
nothing to be discreet about, which is why the code is a MUST where the
reason-naming denials are a MAY.

**Why `SERVICE_` and not `AUTHN_`.** Every prefix answers the same question for
a caller, which is where to look, and `AUTHN_*` answers nowhere, because it is
a verdict. This one sends a caller to the `service` its own configuration told
it to send, which no existing family covers. Subject first keeps it clear of the
denials, the same reasoning that put `PEERCRED_DENIED` in that order.

**Why `REQUEST_UNSUPPORTED` is better for losing it.** It covered a message type
a server does not implement, which is a code fault, and a selector it does not
recognise, which is a configuration fault. That straddle was raised as an
objection four times and declined each time on the ground that both need a
person and reach the same PAM code. The fifth time, the remedy went the other
way: rather than splitting the status, take the second condition off it. The
status now means one thing.

**Why falling back to a default policy is forbidden.** A server with one
configured backend has no reason to look at `service` at all, so the natural
implementation ignores it and authenticates against the only policy it has. That
is the failure `service` exists to prevent: it is the one field §6.2 marks
trusted, precisely because it selects policy, and applying the wrong policy
means checking a credential against the wrong audience.

**Why `UNAVAILABLE` is a status and not a result.** Every other result is a
decision. `UNAVAILABLE` is the absence of one, which puts it with the outcomes
that report how the request went rather than what it decided. Keeping it among
the results would also let one response state both that the server answered and
that it had no answer.

**Why `REQUEST_TIMEOUT` is not what a slow dependency reports.** Both reach a
PAM module as `PAM_AUTHINFO_UNAVAIL`, so choosing wrongly changes nothing an
operator sees in the outcome and everything about where they look. The name now
carries what this paragraph used to have to explain: `REQUEST_TIMEOUT` says the
request did not arrive, which points at the client and the network between them.
`UNAVAILABLE` says a dependency did not answer, which points upstream. The two
are sequential phases of one exchange, and a single deadline spanning both
reports the second as the first.

**Why the line between them is whether waiting helps.** "A dependency" against
"the server itself" sounds exhaustive and is not. A backing file that is present
and readable but holds an unparseable line is neither: it is the server's own
data, not its code and not something it failed to reach. So is a trust bundle
that expired. Those are the commonest real failures in both reference servers,
and the pair of statuses had nowhere to put them.

Whether a retry could succeed sorts them, and it is also what the operator acts
on. It puts loaded data with `INTERNAL_ERROR`, which reaches the PAM code that
fetches a person, rather than with `UNAVAILABLE`, which tells a host to wait for
something that will still be broken.

**Why a peer refusal gets `PEERCRED_DENIED`.** A server that checks the
connecting peer had nowhere to put the outcome. §2 obliges it to answer once it
holds `ludsa_version_major`, and none of the other statuses fits: the request
was not at fault, the server's own state was not at fault, and `RATE_LIMITED`
says a later attempt may fare better, which is never true of a refusal on
identity. This entry used to separate the two by saying `RATE_LIMITED` was about
declined work, which stopped being the line when the seventh round widened it to
cover a declined connection. Retry value is the durable distinction and was
always the operative one. So the check had to happen at accept, where
closing without replying is what §2 prescribes anyway, and the diagnosis was
lost.

The disclosure is smaller than the one §6.2 guards against. A uid is not freely
chosen. An attacker must already be running as one to try it, where a user name
can be guessed without limit, so the enumeration that makes
`AUTHN_DENIED_USER_UNKNOWN` optional has little purchase here.

**Why the section says when a server answers rather than when it checks.** It
used to say the check completes before a message arrives, so a server that
answers has no version from the client. That was an assertion about
implementations rather than a rule, and it was not true of the deployment
`IMPLEMENTATION.md` recommends: a server that checks after reading
`ludsa_version_major` is the only one whose refusal reaches the client's log at
all. Beside it sat an unconditional permission to close without replying, which
the §2 of the time forbade from the moment the version word was read. The tenth
round removed that prohibition, and the entry above says why.

The seventh round replaced that with silence: three positions, all permitted,
nothing said about choosing between them. The eighth round found what the
silence cost. A peer that is both unauthorised and sending a faulty request has
two true things a server could report, and which one it reports came down to
where the implementer happened to put the check.

So the rule is about when the check happens, and the answer is at accept, before
the server has read anything. The kernel's credentials are readable there and
can be kept, so waiting gains nothing and a server that waits has read octets
from a peer it could already have refused.

What it does next is its own choice, and each answer buys something. Closing
without replying is the only one that says nothing, so a server unwilling to
disclose the refusal takes it. Answering at once in the server's own version
costs nothing and tells a client configured for that version why it was refused.
Waiting a bounded interval for `ludsa_version_major` buys an answer the client
can parse whatever version it speaks, at the cost of holding a connection open
for a peer already refused. Reading the version word and then closing anyway is
the same permission §2 gives any refusal reached at accept, and it buys the
discard rather than the diagnosis.

**Why the peer refusal's answers stopped being a closed list.** §5 gave three,
and §2 then let any refusal reached at accept close without replying whether or
not it read the version word, which is a fourth shape and the one a peercred
check acquires by accident. A closed list is a claim about every case, and this
one had been outgrown by a section pointing at it.

**Why the specification ranks none of them and the guidance does.** Which one is
right depends on how loaded the server is, and load is not something a wire
specification can observe. In the ordinary case the third is the one to take.
The client learns why it was refused, in a version it can parse, and a client
still writing a credential is not cut off mid-write. A bounded interval and a
discard loop are cheap against that.

What the ordinary case does not cover is a server under pressure from the
refusals themselves. An unauthorised peer opening many connections is the shape
that turns each of those accommodations into a per-connection cost: an interval
held, and a discard loop run, for peers the server has already decided not to
serve. Closing at once is how a server sheds that, and it has to be available
without qualification, or a server would have no conforming way out of
exhaustion.

So the first answer is neither the recommendation nor an oversight. It is the
fallback, and a deployment reaching for it under load is doing the right thing.
The same reasoning is why the drain here is a MAY where §3's is a MUST: §3's
binds a server that rejected a message, which is a peer it was willing to serve,
and this one does not.

**Why reading a version is safe once the refusal is settled.** The ninth round
first forbade it. Both reviewers had proposed making §5's accept bullet defer to
the bounded interval §2 then carried, and the author reversed that: a server
which has refused a peer must not parse octets from it to decide what the
refusal looks like. The prohibition went further than its reason. What the
reason forbids is letting a refused peer change the answer, and a version word
read to choose an encoding changes nothing. The status is `PEERCRED_DENIED`
whatever arrives, whatever version it names, and whether or not anything arrives
at all.

That displacement is why nothing else surfaces here. `VERSION_UNSUPPORTED` and
`REQUEST_TIMEOUT` describe something the peer did after the refusal was settled,
and reporting either sends an operator to the wrong configuration file. A peer
that is refused and also speaks an unimplemented version is misconfigured twice;
the refusal is the one worth reporting, because it is the one that would still
refuse if the version were fixed.

The tenth round added the `RATE_LIMITED` a server gives at its concurrency
bound, which is reached alongside this check and which the entry had not
considered. It falls the same way and for a sharper reason: `RATE_LIMITED` tells
a peer a later attempt may fare better, which is never true of a refusal on
identity, and it sends an operator to capacity when the fault is permissions. §5
now leads with the property, that nothing displaces the refusal, and names the
answers it does only as instances of it, which is the same move made on §2's
list of when a server may say nothing.

A drain is a separate question and the answer falls out. §3's drain binds a
server that rejected a message, and this refusal precedes any message, so none
is owed. A server may still read and discard, which is what keeps a client from
being cut off mid-write, and it parses nothing beyond `ludsa_version_major`. It
therefore holds no length it may rely on, and §3 already says its permission to
stop early does not apply where a server has none.

**Why a per-`service` peer rule still has somewhere to sit.** A deployment whose
rules govern which peers may reach which `service` values cannot settle that at
accept, because the `service` arrives in the request. Its credentials check
still happens at accept, which is what the rule above requires. What waits is a
second gate at §4's step 11, and `PEERCRED_DENIED` answers that one too, since
it is still the connecting peer being refused. A request too malformed to yield
a `service` never lets it decide, so the framing status is the only answer it
has.

The eleventh round ordered the two answers that meet at step 11. A `service` the
server does not hold is `SERVICE_UNSUPPORTED` whoever asks, because the gate
guards what stands behind a name and an unheld name has nothing behind it. The
other order surfaced a client's misconfigured `service` as a peer refusal, which
sends the operator to the wrong file, and the two map to different PAM codes.
What a refused peer learns from it, that a name is unheld, is small beside what
`PEERCRED_DENIED` already tells it. No case drives the order. A case reached
`PEERCRED_DENIED` itself in the thirteenth round, and this passage had recorded
a reason against one: that a fixture for the check would describe a server's
policy and nothing in the protocol. That reason does not separate such a fixture
from F1, F2 and F3, each of which describes a deployment's policy too. What
carried the reversal is a fact the passage could not have weighed, because the
thirteenth round found it: §7 claimed every case driven against a server is
answerable, which is false of a server that refuses the uid the driver connects
from, since it refuses every case alike and §2 leaves a driver unable to tell
that from a fault. F12 is a precondition of reading the suite at all before it
is an input to any one case. The order the check runs in still has no case.

An earlier draft justified a later check by saying it lets a server compare the
process id the kernel attests against the one the request claims. That is wrong
as a reason to delay. The kernel's credentials are readable at accept and can be
kept, so a server checks the user id immediately and compares process ids later.
The comparison needs the request and the allowlist check does not.

The name puts the subject first, where the result codes in §6.2.1 put the verb
first. That is deliberate. `PEERCRED_DENIED` is a status in §5 and refuses a
connection; `AUTHN_DENIED_CREDENTIAL` and the rest are results in §6.2.1 and
refuse an authentication. Nothing is ambiguous in use, since they travel in
different messages, and the reversed shape keeps the two families apart on the
page.

**Why the timing note is guidance and not a rule.** A server answering
`AUTHN_DENIED` uniformly and then returning in a microsecond for an unknown user
and thirty milliseconds for a policy lookup has disclosed what the code
withheld. That much is worth stating. What to do about it is not: a fixed floor
costs latency on every denial, doing the work either way costs load, and which
is tolerable depends on traffic this document knows nothing about. A MUST here
would bind deployments to a number nobody can pick for them.

**Why `UNAVAILABLE` and `INTERNAL_ERROR` are separate.** Both mean no verdict
was reached, and an operator does different things about them: one waits or
falls through, the other fetches a person. Reporting a condition that needs a
person as `UNAVAILABLE` sends whoever is on call to inspect infrastructure that
is working correctly, and the real cause then survives every restart.

An earlier draft of this entry sorted them by whose component failed, which the
entry above supersedes and which the sixth round found still standing in §5's
table. Fault ownership is a heuristic that agrees with the real test most of the
time and not always: a dependency refusing the server's own credentials is
someone else's component and still needs a person here.

**Why an `ERROR` message still carries `error_detail`.** A rule that a failed
response carries no fields at all reads well until §3.1 asks a server to name
the bound it rejected a field for. Diagnostics on a rejection are worth more
than the tidiness, and one optional field is the whole cost. Assigning it in the
`ERROR` group rather than within AUTHENTICATE is what makes a failed PING able
to say why: assigned to one request type, it is an unrecognised optional tag
everywhere else, and §3.3 has a conforming client skip it without a word.

**Why the non-disclosure rule binds the client.** The server has no channel to
the remote party: §2.1 gives it nothing but the socket, and §1 places the remote
party on the far side of the host application. A rule naming the server
therefore cannot be broken by one. The client holds both ends, and the route
that matters is a host application surfacing the value in an error it returns,
which is what a PostgreSQL validator placing it in a `PGresult` would do.

**Why `error_detail` may not carry the credential.** The field is written into
two logs by design, and a log leaves the host in any shipping deployment. §3.1
and §6.2 both ask a server to say something about a value it rejected, and the
natural way to write that quotes the value. For every other field that is useful
and harmless. For the one field that is a bearer token it is a disclosure with
no bound on where it travels, and nothing in the conformance suite compares
`error_detail` content, so no test would ever show it.

**Why tolerance has a bound but logging does not.** §5 leaves what a client does
beyond reporting to the client, which is right: a defective `error_detail` is a
diagnostic, and sanitising one for a log is a reasonable thing to do with it. A
defective `authn_id` is not a diagnostic. It is the value §6.2 has a client
authorise on when it sent no `user`.

So the line is between rendering and acting. A client may transform a defective
value to describe it, because a fault nobody can describe is a fault nobody
fixes, and the server's own logger is told to do exactly that with bytes it
cannot encode. What it may not do is treat the rendered form as the value. The
hazard is the one this document already describes for a server's JSON handler,
one layer further on: a replacement character collapses two identities that
differ only in their defective octets into one, and a client that authorises on
the result grants a token the wrong identity.

Truncation needed no new rule. §3.1 already says no party may truncate a value
to fit, which reaches a receiver shortening an over-long `authn_id` as readily
as a sender.

**Why the safe-to-log guarantee says what it is not.** The guarantee is about
content: no control character, valid UTF-8, inside the maximum. An earlier
wording stopped at "so that a receiver may write it to a line-oriented log
without escaping", which is true and is read as more than it says. The receiver
that matters is a PAM module, and the line-oriented log call it has is
`pam_syslog`, which is `printf`-shaped. Nothing in the banned set excludes `%`,
because `%` is not a control character.

The seventh round raised this as a format-string bug and answered it twice: the
guarantee stopped over-promising, and a MUST was added forbidding a receiver to
use the value as a format string, a template, or anything else that interprets
its content.

**The eighth round took the MUST back out.** A rule about what a receiver hands
a value to is not observable on the wire, no conformance case can reach it, and
it is a property of untrusted input in general rather than of these fields.
Naming three fields carries the worse implication that the rest are safe, and
the rest are the ones a remote party chooses outright: `user` is entirely under
its control, and a server logging it faces the same class of hazard with no
rule beside it. HTTP does not tell an implementer that a response header is not
a format string, and nor should this.

What survives is the half that caused the trouble. The guarantee says what it
covers, the octets, and says it is not a statement about what a receiver may
hand them to. The misreading came from the promise, so the promise is where the
correction belongs. Everything else went to `IMPLEMENTATION.md`, which already
carried the wrong and right `pam_syslog` calls, under a general statement that
every value off the socket is attacker-influenced at the point of use whatever
guarantee its field carries.

The consequence is still not symmetric with the server's logging hazard, which
is why the guidance is emphatic where the specification is silent. A forged log
line misleads whoever reads it. A format string in a module loaded into an
`sshd` running as root, where the server may be an unprivileged sidecar, turns a
compromised server into code execution. Of the three, `authn_id` reaches
furthest: `error_detail` arrives only in an `ERROR`, and the identity is logged
when everything went right.

**Why the checks are bound to the field's declared extent.** A C receiver stores
the value before it inspects it, and the store it reaches for is a
NUL-terminated buffer. The checks then run over a shorter value than the one
that arrived, and pass honestly. §5's tolerance bound already names the failure
that produces, two identities differing only in the defective octets arriving as
one, and a NUL reaches it with no substitution at all. Binding the checks to the
extent the field declared is what closes it, and it is the same sentence §6.2
spends on `credential` for the same reason.

**What was declined with it.** The same round proposed banning U+2028, U+2029
and the bidirectional overrides. Neither earns a place in the banned set. U+2028
and U+2029 do not terminate a line for syslog or for a text log, and the parsers
that once choked on them do not any more. The overrides mislead a human reading
a line rather than forging one, which is a different class, and banning them
would put a filtering obligation on every server and cut against §3.3's rule
that a receiver imposes no constraint a field's own definition does not carry.

**Why the receiver's check is at the point of use.** The producer's obligation
alone leaves nothing that must ever look, so a client may take the guarantee at
face value and write a forged log line on the strength of it. Checking on
receipt closes that and charges every client for a value it may discard: a PAM
module ignores `error_detail` on every successful authentication.

At the point of use the cost falls where the benefit is. A receiver that ignores
the field does nothing. One that logs it makes a single forward pass, allocating
nothing, over bytes it is about to write anyway, and the control-character test
comes out of the same pass that validates the encoding.

That last part is why the range must be read as code points. U+0080 to U+009F
encode as `C2 80` to `C2 9F`, and those second octets are ordinary continuation
bytes: `U+0100` is `C4 80`. A receiver scanning octets for `0x80` to `0x9F`
discards any value with a Latin Extended, Greek, Cyrillic or CJK character in
it. On the path where no `user` was sent, that silently removes the identity the
client depends on.

**Why "safe to log verbatim" is a control-character rule.** The `utf8` type
already supplies valid encoding and no NUL, so the guarantee has to mean
something further or mean nothing. What a line-oriented sink is vulnerable to is
a newline forging a second log line and an escape sequence reaching a terminal,
which is the rest of the control range. The obligation sits on the server
because the client is typically a C module writing to `pam_syslog`, and a
receiver is forbidden from rejecting on it so that a diagnostic never becomes a
failure. `server_desc` carries the same rule because §6.1 gives it the same
destination.

**Why a rate-limited server refuses rather than answers.** `max_age` is a
requirement, not a request. A server that answers from data older than the
client asked for has broken it silently, and the client goes on to believe a
reading that may predate whatever it is investigating. `RATE_LIMITED` tells the
client its requirement was not met, which is something it can act on.

## 6. Messages

**Why an unimplemented `msg_type` gets `REQUEST_UNSUPPORTED` rather than
`REQUEST_MALFORMED`.** The framing was valid and was understood. Telling a peer
its bytes were broken when they were not sends whoever is debugging after a
framing fault that doesn't exist. It is also what makes a new message type
additive: the two sides ship separately, so a newer client sending a later
message type to an older server is expected, and it gets an accurate answer it
can act on.

**Why there is no authorize-only request type.** A client invoked only in PAM's
`account` phase has never seen a credential, so it could answer on `user`,
`service` and `rhost` alone. That question is a real one, and `pam_access`,
`pam_time` and a directory's account lookup all answer it. But it needs no new
message. The AUTHENTICATE response already carries a `result` that distinguishes
a credential outcome from an authorisation outcome, so a server whose
authorisation decision is independent of credential validation has a code for
what it decided. Whether a client acts on it is a separate matter: PAM aborts
the stack when the credential step fails, so a PAM client would never reach the
authorisation step to see it.

So version 1.0 defines three request types and nothing more, and no code is held
in reserve against a future one. An unimplemented request type already has a
defined answer, so if demand appears a later version assigns a code. That is a
minor bump.

The host-side situation such a request type would serve does already exist. sshd
running publickey, GSSAPI or hostbased authentication reaches `pam_acct_mgmt()`
without having run the auth stack, and cron never runs the auth stack at all.
What does not exist is a LUDSA server that can answer without a credential.
`pam_ludsa` meets those hosts by failing closed, because Appendix B has
`pam_sm_acct_mgmt` fail when it holds no verdict for the transaction. That is
the correct answer, and it means the module can't serve an sshd `account` stack
alongside publickey login.

**And such a request would disclose what §6.2 lets a deployment withhold.**
Answering on `user`, `service` and `rhost` alone tells a caller which names the
server holds, for any name it cares to try, with no credential to produce and no
denial to withhold. §6.2 makes naming a denial reason a deployment's choice
because `AUTHN_DENIED_USER_UNKNOWN` discloses that much, and there the
disclosure is bounded: a caller reaches it by sending a credential under a
`service` the server recognises. A credential-free query removes the bound and
the choice together, and leaves an enumeration oracle gated by nothing but the
socket's permissions. STATUS and PING each carry no credential, and neither
answers which principals a server holds: STATUS reports the server's own health,
and PING reports that the server is there. So a later version weighing this is
trading user enumeration for the ability to decline in stacks this protocol was
not built to serve, and the second is worth having only where someone wants a
LUDSA principal to log in by SSH key.

### 6.1 PING

Nothing in the response is required, because the header is the substance of the
exchange. The response header carries the server's major and minor version,
which is the whole compatibility answer, so PING needs no field to do its job
and the body is purely diagnostic.

`server_desc` is a separate thing from the version deliberately. One string
carrying both a name and a release is the obvious shape, and it gives a client
no correct way to test compatibility while putting an incorrect way in easy
reach. Keeping them apart leaves the string with nothing to be misused for,
which is why a client must not parse it.

### 6.2 AUTHENTICATE

**Why `user` must be bound to the credential when it is sent.** In the
PostgreSQL case it comes straight from the client's startup packet and is
entirely under the remote party's control. A valid credential alone establishes
who the caller *is*, not that they may act as the role they asked for.

**Why `user` is optional.** Requiring it makes the server the policy enforcer in
every deployment, and that is a choice about how a system is arranged rather
than a property of authentication. Some credentials identify their subject, and
the driving case is one: a client holding a signed token may need nothing from
the server but the signature check, because it can read the subject itself and
already knows its own policy.

Sending `user` is therefore the question "may this identity act as this user",
and omitting it is the question "is this credential valid, and do you object to
it". Both are real, and a protocol that only asks the first forces every
deployment into an arrangement it may not want.

Omitting `user` does not forbid the server from having a policy. It may extract
the identity, apply whatever rules it holds, and deny. What it cannot do is tell
the client that it did, because there is no field for that and no reason to add
one: a client that could rely on the answer would be asking the first question
after all. So server-side policy in this mode is a second gate rather than the
gate, and the client's own check is what makes the mode safe.

What the two share is that a valid credential alone is never an authorisation.
Omitting `user` moves that obligation to the client rather than dissolving it,
which is why §6.2 states it there. A client that omits `user` and then treats
`OK` as permission has built the failure the server-side rule exists to prevent,
and put it somewhere no server can see.

`authn_id` stays optional in both cases. A client that can read the credential
does not need the server to tell it what it says. One that cannot, or that would
rather not depend on its own reading agreeing with the server's, asks for a
server that supplies it.

**Why consistency within a `service` carries a SHOULD.** Both reviewers in the
fifth round found the same hole: presence is a property of a response, not of a
server, so the arrangement above cannot be checked. A deployment tries one
credential, sees `authn_id`, and generalises, and a server that fills the field
from its upstream and omits it on a cache hit is conforming. The failure is
fail-closed but intermittent, and it presents as a server fault.

**The rule named a mechanism for two rounds, then was generalised too far.** It
asked the cached and derived paths to match the live one. The seventh round
found the justification for that scoping did not distinguish it from the
per-`service` proposal it had rejected, and replaced it with a property: within
one `service`, the same optional fields on every response, whatever path and
whatever declared minor version.

**The eighth round broke that property twice, in ways neither reviewer saw the
other find.** It could not be obeyed: an AUTHENTICATE response carries one
optional field, the identity the server resolved, and a server that denied for a
bad credential resolved none, so every server populating it on `OK` broke the
rule on its first denial. And it contradicted §3, which lets a server carry
information in a field an older client understands, keyed on the declared minor
version, since that is exactly a change to which optional fields are carried.

**The minor-version clause was the author of both, and it was protecting
nothing.** It went in because the seventh round counted three routes by which a
response's fields could vary and closed all three. That count was wrong. A
deployment's client sits at one minor version and meets the same substitution on
every exchange, so its check is stable whatever a server does across versions.
The clause closed a route that was never open, at the price of forbidding a
permission §3 grants.

So the comparison is between answers alike: within one `service`, the same
optional fields on every AUTHENTICATE response carrying the same `result`. That
is satisfiable, because a denial and an `OK` are no longer being compared, and
it is what the arrangement actually needs, since a client that omits `user`
depends on the identity when the answer is `OK` and grants nothing when it is
not. Scoping it to the AUTHENTICATE response also keeps out the message types
that were never at stake: an `ERROR` is a response, and no `service` is known
when a message is rejected early.

The narrower form is close to what a reviewer proposed a round earlier, to
populate the identity on every `OK` or on none. Generalising it to every
optional field on every response is what broke it, and the lesson is the
generalising rather than the rule.

**Why `service` is safe to route on.** It derives from the `pamservice=` option
on the matched `pg_hba.conf` line, which the remote party cannot influence. This
is the recurring *checked, never consulted* principle: a value the remote party
can influence may be tested against configuration, never used to select which
configuration applies.

**Why the credential field is not called `authtok`.** That name is PAM's, and
the protocol is backend-agnostic. `credential` is the word this document already
uses everywhere else for the concept, including in `AUTHN_DENIED_CREDENTIAL`.

**Why it is not re-encoded.** A JWT is already printable ASCII, the framing is
length-prefixed binary rather than line-oriented, and base64 would add a third
to the size plus a decode step in the very component this design keeps minimal.
It is `bytes` rather than a string because the protocol is generic: another
backend may carry a password, or something that is not text at all.

**Why a sender is bound to its own schema, and not only to its own tags.** The
thirteenth round bound a sender to the enumerated values its version assigns. It
said §6 did the same for `msg_type`. §6 said nothing of the kind. It assigns the
values, it tells a receiver what to do with one it does not accept, and it
leaves a client free to put a response type in a request.

The type rules had the same gap. §3.3 bound a receiver to reject a value its
type forbids, and bound a client not to connect with one. It never bound a
sender, and §5 and §7 both rested on a server being bound. `MAX_LENGTH` was the
third. §3 addressed every sentence about it to a receiver, and §5 told a client
reading `REQUEST_TOO_LARGE` that its peer had sent something §3 forbade.

The three now sit in §3.2, beside the tag rules, which is where a sender's
obligations already were. Stating them in one list is what stops the next
version of this gap. A rule written for a receiver reads as complete until
somebody looks for its other half.

**Why the rule is general, and why `credential` still names it.** §3.2 binds
every sender to carry a value as the octets it was given, so the obligation
reaches `user`, `service`, `rhost`, `ruser` and `tty` as well. Stating it once
is deliberate: the ninth round removed a MUST from §5 that named three fields
and so implied the rest were safe, and an enumeration here would repeat that.
The ninth round also found only `credential` bound, which left a client free to
trim or case-fold the role name a decision turns on.

`credential` still carries its own sentence because the rule binds the server
there as well. A signature covers exact bytes, so a value altered on either side
is a different value, and there is an objectively right answer about what the
octets are. `user` has no such answer: what a server's policy does with a name
it received is that deployment's decision, and §6.2 already says the policy maps
the credential's identity to the requested user. A deployment that folds case is
choosing a policy, not misparsing, so the general rule binds the sender and
stops there.

The reason `credential` is `bytes` and takes no text validation had only a
hypothetical case behind it until the thirteenth round. `sshd` manufactures one.
For a user that does not resolve it substitutes `fake_password()` before the
module ever sees it, a value of the same length filled with a repeating pattern
of control characters, so that a caller cannot tell an unknown name from a wrong
password by timing. A codec that checked `credential` for printable text would
pass every PostgreSQL test and reject the ordinary failure path of the other
host the library is meant to sit in. Measured 2026-09-04.

**Why no party may normalise the credential.** A signature covers exact bytes,
so any change breaks verification. The worse failure is asymmetric. A client
that trims and a server that does not is a parser differential in an
authentication path, and a single implementation that trims on *both* sides is
self-consistent and passes all of its own tests. Only the conformance matrix
catches it.

The whitespace-detection SHOULD exists because rejecting is correct, but
reporting "signature verification failed" when the cause was a stray newline is
not. The report goes to the server's own log, because there is nowhere else for
it to go. `error_detail` lives only in `ERROR`, and a credential that fails
validation is a decision, so what the client receives is an
`AUTHENTICATE_RESPONSE` carrying `result` and `authn_id`. Rerouting to `ERROR`
is not open either: `credential` is `bytes`, whose row in §3.3 carries no
constraints, and §3.3 forbids imposing any beyond a field's own.

**Why `error_detail` was not given a tag in `AUTHENTICATE_RESPONSE`.** It would
make the diagnosis wire-visible, and so testable, which is the one thing a
server's private log cannot offer. Against that it takes the largest response
from 534 octets to 1045, doubling the fixed receive buffer a client needs for
every version it will ever meet. It puts free-form text on the denial path,
where §6.2 asks a server to be able to answer discreetly and the disclosure
rules are written against an enumeration rather than against prose. And it adds
a second optional field to that message whose presence varies with the
deployment, which is the property that already leaves `authn_id` unverifiable
from one exchange.

**Why `pid` is logging-only.** It lets a server's log join to a PostgreSQL log
carrying `%p`. Forging it requires access that already permits forging the whole
request, so it is not a security boundary.

The rule says "any decision" rather than "any authentication decision", and the
ninth round found the guidance relying on the gap it does not leave: it offered
an opt-in mode that refused a connection whose attested pid disagreed with the
claimed one, called a correctness check rather than a defence. The name does not
change which rule it breaks, and the trade is bad in both directions. Against an
attacker it buys nothing, since one who can forge the field can forge the
request. Against a deployment it risks refusing valid logins on a self-reported
value the moment a namespace assumption stops holding. Logging the disagreement
keeps the diagnostic and drops the hazard.

**Why there is no machine-readable companion code.** The result codes are the
machine-readable layer. A finer classification, separating *expired* from *bad
signature* from *audience mismatch*, is backend vocabulary, and §6.3 already
declines to give backend-specific values their own tags for that reason. A code
coarse enough to be backend-agnostic would duplicate the result. A server
wanting the distinction already has it in its own metrics without the protocol's
help.

What such a code would add is letting the *client* classify, and so alert from
the host application's logs rather than the server's. What would settle it is
someone wanting that. Should it ever be added, §4.1 makes it an optional field
in the `ERROR` message: diagnostic and never authoritative, since `result`
remains the verdict, so an older client that skips it loses a log detail rather
than a decision.

#### 6.2.1 Auth result

**Why one result rather than two verdicts.** Two fields, one per decision, is
the obvious shape when the client has to answer twice. They cannot usefully
disagree, though. Either the credential is bad, and there is nothing to
authorise, or it is good and the authorisation answer stands on its own. The
pair would only ever carry one piece of information, and the second field would
be derivable from the first.

**Why the second precondition needed splitting.** §6.2.1 gave
`AUTHN_DENIED_USER_UNKNOWN` two conditions and then said, of the pair
`AUTHN_DENIED_POLICY` excludes, that those are what it names. The table assigns
the second only where no `user` was sent, so a request carrying a `user` that
resolves, with a credential resolving no identity, had a reason-code by the
paragraph and none by the table. Both readings deny, but two servers naming
reasons named different ones. The paragraph now says which condition belongs to
which request, and leaves that case to `AUTHN_DENIED`, which is always
available.

The code carries the distinction instead. A client that must answer at two
points reads which decision the code belongs to.

**Why a denial says as little as the server chooses.** The denials are one
required code and a set of optional refinements. `AUTHN_DENIED` names no reason
and is always available; the others are for a server willing to disclose that
much.

The reason is enumeration. `AUTHN_DENIED_USER_UNKNOWN` tells a client which user
names exist, and that does not stop at the client: a host application behaves
observably differently for an account it does not know, which is the leak
OpenSSH spends effort closing. A server whose users are not public knowledge has
a real interest in saying nothing.

The thirteenth round gave `AUTHN_DENIED_USER_UNKNOWN` the prohibition
`AUTHN_DENIED_POLICY` already carried. §6.2.1 stated it in the indicative while
§7 had a case turning on it, so a server naming the code for a user it holds
failed a case with no rule behind it. Appendix B is what makes that more than
tidiness. It maps the code to `PAM_USER_UNKNOWN`, which is the entry an operator
keys `user_unknown=ignore` on. A server free to emit it for a user it does serve
can then make a module stand aside for a principal it owns.

Until the fifth round the codes did not allow that. The two denials were
`AUTHN_DENIED_POLICY` and `USER_UNKNOWN`, and a server that wanted to leak
nothing had to pick one for both cases. The only candidate was
`AUTHN_DENIED_POLICY`, whose meaning is that the credential was valid, so the
private choice was the dishonest one: a server that had never checked a
credential asserted it was good, and a client reading §6.2.1's two-step split
concluded the credential step had passed. `AUTHN_DENIED` makes the private
choice truthful.

**Why naming a reason is left to a deployment.** A code reserved for the cases a
server is willing to explain names the remaining case by omission, where one
case is all that remains. Answering `AUTHN_DENIED_CREDENTIAL` for a bad
credential, `AUTHN_DENIED_POLICY` for a policy denial and `AUTHN_DENIED` for
everything else tells a client that `AUTHN_DENIED` means the user is unknown.
Leaving two cases under it discloses nothing.

An earlier draft made this a MUST binding every denial, on the ground that a
rule of that shape cannot drift as a policy set changes. It was overruled. Which
code answers a given denial is a policy and configuration question rather than a
protocol one, and it trades diagnosis and traceability against disclosure. The
reason often reaches a log and goes no further. A server is asked to offer both
modes so the trade stays with whoever runs it, and one that offers only the
discreet mode is conforming.

**What the earlier framing got wrong.** It presented the choice as a question
about the server's own user model, authoritative or rules-only, and sent the
reader to §6.2.1 for what each implies. §6.2.1 puts the two on opposite sides of
the credential-versus-authorisation split, so a choice the reader had been told
was about house style decides what a client concludes about the credential. The
cross-reference was accurate and still misled, because it did not say the choice
was semantic.

**Why `AUTHN_DENIED_POLICY` now carries a MUST NOT.** Its meaning is that the
credential was valid, so a server that has not established validity may not
answer it. An earlier round declined a rule keying the choice to validation,
because that rule was two-way and forced an authoritative-user-set server off
`AUTHN_DENIED_USER_UNKNOWN`. This one is one-way. It leaves every other code
free and closes the case where a server parses an identity out of an unverified
credential, finds no rule, and tells a client the credential passed.

**Why `AUTHN_DENIED` is a credential outcome.** It is not, strictly: it is
nothing at all. But a client has to place it somewhere, and the credential step
is the safe side. Treating it as an authorisation outcome would have a client
conclude the credential was good on a code that states nothing, and return
`PAM_SUCCESS` from `pam_sm_authenticate` on the strength of it.

**Why `AUTHN_DENIED_POLICY` is the only authorisation outcome.** Its meaning is
exactly that the credential was valid and policy denied it anyway, so it is the
one code that passes the credential step. `AUTHN_DENIED_USER_UNKNOWN` looks like
it belongs alongside it, since the name suggests a server resolved an identity
and then found no mapping. Nothing in the protocol requires that order. A server
may know the user is unknown before it examines a credential, and a flat-file
server would. So `AUTHN_DENIED_USER_UNKNOWN` implies nothing about the
credential, and it fails the credential step, which is the earlier and safer of
the two.

**Why a `user` that does not resolve may not be answered
`AUTHN_DENIED_POLICY`.** For six rounds it could be. A server that validated the
credential and then found no mapping had established what `AUTHN_DENIED_POLICY`
requires, so both codes described the case and the choice fell to how the server
was built.

The seventh round joined that to the paragraph two sentences away in Appendix B.
The codes sit on opposite sides of the credential and authorisation split, so
the same request fails `pam_sm_authenticate` against one conforming server and
returns `PAM_SUCCESS` from it against another. A stack without an `account`
line, which Appendix B already names as the fail-open, therefore admits the
caller against the second server and denies against the first. Whether a
deployment's omission is exploitable came down to which server it met, and no
deployment can test for that.

So the condition is assigned by whether the name resolves rather than by the
order a server worked in, which is what both glosses said in the first place.
Nothing is disclosed that was not disclosed before: every reason-naming code is
optional and `AUTHN_DENIED` is always available, so the rule reaches only a
server that had already chosen to name a reason.

The eleventh round found the precondition itself saying "no mapping exists for
the requested `user`". In §6.2's vocabulary a mapping is the policy relation, so
the sentence named the case it existed to exclude, and a flat-file server
reading it that way answered `AUTHN_DENIED_USER_UNKNOWN` for a name it held on
another credential's line. The precondition now uses the table's own word and
says what resolving means: the server holds the name, whatever its policy says
about who may act as it.

### 6.3 STATUS

**Why STATUS carries no credential.** Its contents are low-sensitivity: health,
counters, and configuration deliberately exposed for diagnostics. In the
intended deployments no consumer needs one without the other. Probes run inside
the server's own container at its own uid, and metrics scraping uses a separate
HTTP endpoint. So anything permitted to query health is also permitted to
authenticate, which is an accepted consequence rather than an oversight.

**Why `service` selects a policy.** A server may serve several services with
different configuration, meaning different expected audiences and different
mappings, so there is generally no single answer. An unrecognised value must be
refused rather than silently answered from a different policy's configuration.

**Why `max_age` rather than a flag.** A boolean asking the server to recheck is
the simpler-looking option, and it makes STATUS two operations under one name,
report and refresh. It also has to name what gets rechecked, which presumes
every server has an upstream. A freshness bound is a constraint on the query
instead. It covers both cases with one field, since zero asks for a check, and
it says nothing about what the server is made of.

It also removes any need for the response to report an age. The client sets the
bound, and the server either meets it or refuses.

**Why a failed check still satisfies `max_age`.** A server whose dependency did
not answer has just learned that its dependency is not answering. That is a
fresh reading, and it is the reading the caller wanted.

**Why `upstream_*` and `cache_*` are not fields.** They are the obvious things a
health endpoint reports, and they presume a server with an upstream and a cache.
A server backed by a flat file has neither, and would leave six tags empty for
ever. §6.3 has a server with more to report than `serving` and `server_desc`
carry expose it by its own means, to a client written for that server.

### 6.4 Tag assignments

The table is checkable in a way the tags alone are not: each row states the
required flag, the length, the type and the number separately, and the tag
beside them either decodes to those four or the row is wrong. That is the
property to verify when the table changes, and it is mechanical.

**Why tags are shared across groups.** A tag is unique within its group and not
across the table, and §3.3 matches a tag against the schema its `msg_type`
selects, so a shared tag is never offered to the wrong group. Sharing one is
harmless to a conforming receiver.

Version 1.0 first numbered to avoid it anyway, and it was not free. `status`,
`result` and `serving` share a metadata byte, and so do `error_detail`,
`server_desc` and `authn_id`, so avoiding a shared tag meant starting
`AUTHENTICATE_RESPONSE` at 2, `STATUS_RESPONSE` at 3 and `PING_RESPONSE` at 4,
with unassigned numbers left behind in each. The ninth round removed them. Every
group numbers from 1, `status`, `result` and `serving` are all `0x0181`, and
`error_detail`, `authn_id` and `STATUS_RESPONSE`'s `server_desc` are all
`0x0243`.

**What the gaps bought.** One reader: a decoder that fills a shared struct
before it consults `msg_type`, which §6 already forbids. For that reader the
numbering turned a misread into a rejection. §6.4 stated the result as a flat
invariant over the whole table, §4.1 never held later versions to it, and
successive review rounds were spent on the difference between the two.

**And it could not reach the case that matters.** `result`'s `OK` and
`serving`'s `SERVING` are both 2, so the same reader takes a `STATUS_RESPONSE`
for an `AUTHENTICATE_RESPONSE` and reads a health check as a grant whatever the
tags are. §3.4 says so and hands the case to §6. Distinct tags would have
covered that reader in the directions where a misread was already fail-closed
and left him exposed in the one where it was not.

**What stands instead.** §6's rule that a client establishes `msg_type` before
it reads a field, §4's order settling the response type before a tag is
compared, and §3.4 leaving 2 unassigned in `status`, so a `status` read as a
`result` is never `OK`. The last is the only one that survives a receiver
ignoring the other two, and the sixth round put it there for that.

Field numbers are unassigned rather than reserved, because an unassigned
required tag is already safe by construction. A peer meeting one rejects the
message, which is the fail-closed default the rule exists to give.

**Why `AUTHN_DENIED_CREDENTIAL` carries a precondition too.** §6.2.1 bound the
policy code and the unknown-user code and left the credential code free. A
server that had established the credential was valid could still name it. The
same policy denial then failed the `auth` phase against one conforming server
and passed it against another, which is the defect §6.2.1 exists to close, in
the safe direction. What it costs an operator is a reissued credential that was
never the fault. Every code naming a reason stays optional and `AUTHN_DENIED` is
always available, so the rule binds only a server that chose to name one.

**Why a server matches `service` by octets.** §3.2 binds a sender to carry the
value as it was given, and the receiver's half was never stated. §6.2 states it
for `user`, where the name goes to a policy and what that policy does with it is
the deployment's decision. `service` is different because it selects which
policy runs. A server comparing with `strings.EqualFold` routes a name to a
policy that a server comparing octets answers `SERVICE_UNSUPPORTED` for. Both
are self-consistent and both pass their own tests. F1 supplies the recognised
and unrecognised names as opaque strings, so the conformance matrix cannot see
the difference either.

## 7. Conformance

**Why vectors are hand-authored.** Generating `testdata/` from an
implementation's output makes that implementation, and its bugs, the
specification. The resulting suite then passes while proving nothing. This is
the shortcut most likely to be taken, because dumping is much faster than
hand-encoding.

**Why one case carries no expected outcome.** Excess octets past the declared
length may not have arrived, so a conforming receiver may accept or reject a
message carrying them, and a vector would be asserting one implementation's
choice. Listing it separately says so, rather than leaving a suite to discover
it as flakiness.

**Why the table names the side each case is driven against.** The obligation is
a receiver's, and most cases can be constructed for either side, but every
version 1 `string` field is a request field and a request carries no `msg_type`
a response permits. Some cases therefore have no constructible input in one
direction, and a table written as a server's obligation asks a driver to send
what it cannot build.

**Why a driver checks content and not only encoding.** §7 is built around cases,
an input driven at a peer with an outcome expected of it, and a sender's
obligations do not fit that shape: no request makes a server emit a reserved
enumeration value or a control character in a field it promised was clean. It
does that because it has a bug.

The eighth round pointed out that not being able to provoke a breach is no
reason not to notice one. The three checks cost a comparison each, over octets a
driver already holds, on responses the suite already collects. Without them a
server that renders an attacker-controlled `user` into `error_detail` unescaped
ships a forged log line to every client it serves and the suite stays green,
which is the exact hazard `IMPLEMENTATION.md` spends a section on from the
server's own side.

Recording is not rejecting, the same split the undefined-tag check draws. As a
receiver the driver does what §5 asks of any client. As a harness it reports the
peer as non-conforming, and those are different acts on the same octets.

**Why the over-long value gets a fixture of a new kind.** F1 to F6 are facts
about a peer's configuration and F7 is a channel a driver reads. F8 is the first
that hands something *to* the client under test, through the interface its host
uses rather than through the socket, and that is what the rule needs: §3.1
forbids a client to open a connection for a value it may not send, so the only
way to reach it is to give the client such a value.

It is affordable because the client-side harness already does it. The PAM
measurement that settled how a token reaches the module is a conversation with a
chosen token, so an oversized one is the same mechanism with a different value.

**Why the verbatim rule needed a case on the server side too.** The rule binds
both parties in one sentence, and §7 tested one of them. The driver hands a
client a `user` with whitespace around it and reads the wire, and nothing did
the same for a server and a credential. The tenth round found the gap, and it is
the gap that matters most: this is the rule `IMPLEMENTATION.md` calls the worst
failure mode in the protocol, because a server that trims on receipt is
self-consistent. Its reference backend reads stored credentials a line at a time
and has already lost the terminator, so both sides of its comparison are
trimmed; its wire vectors pass because they test the codec against fixed bytes
and never reach the handler. Everything it owns goes green, and it disagrees
with every other implementation about which tokens are valid.

The case is a credential the backend accepts with a newline appended, expecting
a denial. A trimming server answers `OK`. The expectation names the
`AUTHN_DENIED` codes rather than "not `OK`" for the reason the decision cases
do: a server holding no such service answers `SERVICE_UNSUPPORTED`, which is not
`OK` and reaches no decision, so the looser wording would pass a peer that never
examined a credential. That formulation was ruled out in the seventh round and
the reviewer's proposal reintroduced it.

It sits beside the client-side case rather than with the decision cases, whose
lead-in says they are the only test of a different rule in the same section. A
third row there would have made that sentence false.

F3 carries the fact the case turns on, which is whether appending a newline
actually invalidates the credential for that backend, and it names the newline
so that a backend refusing other octets and ignoring whitespace cannot be marked
as trimming. For a signed token it does. A backend that ignores trailing
whitespace on its own account cannot run the case, and the concession says so,
on the terms every fixture uses for a peer that cannot run a case.

**Why a driver has to be told what the peer implements and reads.** §7 spent
several cases on facts about the peer that nothing on the wire carries. The
SHOULD-report cases drive a defective `utf8` field at a client, and §5 relieves
a client of checking a value it does not use, so a driver picking a field this
client ignores records a SHOULD it never owed. The rows name no field, which
leaves the choice to a driver that has no basis for it.

`server_desc` is the sharp end. It appears only in a `PING_RESPONSE` and a
`STATUS_RESPONSE`, so a client that sends neither request can never be given
one. F8 hands a value to a request field through the host interface, which is a
different thing from provoking a message type, and no fixture did the second.

The acceptance rule had the same shape from the other side. It names a
`PING_REQUEST` as the minimum-length message, and §6 obliges a server to
implement no particular request type. A server that answers
`REQUEST_UNSUPPORTED` has accepted the length and failed at §4's step 5, so it
conforms and the rule as written would have recorded a failure. What was missing
was the observable: the assertion is that the length is not what the receiver
rejected.

A server implementing only AUTHENTICATE cannot be driven for the rule at all.
The minimum declared length is `msg_type` and `TAG_END`, which a `PING_REQUEST`
or a `STATUS_REQUEST` can carry because their fields are optional or absent, and
which an `AUTHENTICATE_REQUEST` cannot because it has required fields.
Neither reviewer reached that, and the concession says so rather than leaving a
driver to discover it.

**Why the mid-write case asks a client to write badly on purpose.** §2.1's rule
is the one whose violation kills an `sshd`, and the tenth round found its only
case unable to exercise it. The case had the driver read part of the request and
close while the client was still writing, and no request this protocol permits
is large enough for that. Appendix A's largest sits an order of magnitude inside
a default socket send buffer, so the client returns from its write before the
driver has read anything. It then reports `RESPONSE_ABSENT` exactly as it would
for an ordinary silent server. Both reviewers reached that independently, and
both proposed conceding it.

F10 is the author's answer instead, and it inverts where the difficulty sits. A
driver cannot make a conforming client write in pieces, but the client can offer
to. Bounding the octets written before a flush, and pausing between them, is a
test affordance a client that wants to be tested here can supply, and it makes
the window the driver needs. Nothing about it belongs in the protocol: one write
is the better behaviour at runtime, and a client without the switch is not
non-conforming, which is the concession every such fixture carries.

F8 stays on the case with its reason corrected. It never made the client "still
writing", which is what the old text claimed. A larger request makes it likelier
that a client issues more than one write, which is the only way the fallback
construction catches anything at all.

**Why the relative-path case needs a fixture of its own.** F9 is the working
directory the driver gives the client's host, and the case does not work without
it. A relative path resolves somewhere, and if the driver does not choose where,
it resolves to a name that holds nothing: the client gets `ENOENT`, reports
`CONNECTION_FAILED`, and passes whether it refused the path or resolved it. Both
assertions are satisfied by the behaviour the case exists to catch.

The driver therefore binds its socket in a directory, puts the client's host in
that directory, and configures the client with the basename. A client that
resolves now lands on the driver's socket, so the assertion that no connection
arrived is the one that fails it.

**The same shape, and the same defect, in the case next to it.** A case against
a client's path handling has to put the driver's socket where a non-conforming
client would go, because a wrong path that names nothing is indistinguishable
from a path never used. The tenth round found the too-long case failing that
test as written: it had the driver configure an over-long path and assert no
connection, which a truncating client satisfies by landing on a name that holds
nothing. It also claimed to catch both a client that truncated and connected and
one that connected to whatever the truncated path named, and caught neither.

A truncating client has more than one place to land, which is why the driver
binds more than one socket. The path travels in a fixed array and the two
spellings that get it wrong differ by the terminator: one fills the array and
one stops an octet short. A driver binding at only one of them misses the other
silently, in the same way the original case missed everything. So the driver
binds at the array's size and at one less, and configures the client with a
longer path sharing that prefix. Stated as a property because the size varies
four ways across platforms, on the same ground as §2's own rule.

A platform may refuse to bind at the array's full size at all, which the
guidance measured on macOS. A truncating client cannot connect at that length
there either, so the assertion holds with no socket bound, and the case says so,
since a driver cannot be asked to do what its platform forbids.

The report ends the case rather than a deadline. A driver waiting to prove a
negative would have to decide how long a connection that is never coming takes
not to arrive; waiting for the client to report instead makes the assertion
exact, since a client that connected before reporting has already failed.

**Why some cases name a fixture.** An unrecognised `service` needs the driver to
know a name the peer under test recognises and one it does not, and neither is
derivable from this document. Rate limiting is worse: §6.3 makes it a MAY, so a
peer with no limiter cannot exhibit `RATE_LIMITED` and is not thereby
non-conforming. The decision case needs a credential the peer accepts and
a `user` the server holds that its policy refuses, which is a property of a
deployment rather than of this protocol. Naming what the driver must be told
turns each from a silent assumption into an input.

F12 is the same shape from the other side, and it went in during the thirteenth
round for a reason about the suite rather than about its own case. A server
making §5's check refuses before the version word, so it refuses every case in
the section alike, and a driver that has not established whether its uid is
accepted cannot read a single result. The case it also earns asserts what every
shape of §5's answer shares, that nothing the request asked for arrives, since
§5 leaves a refusing server its choice among them.

F5 is the odd one. §2's identity check binds only a client that was configured
with an expected identity, so the fixture supplies both halves of that
configuration, and the case turns on which uid is at the far end of the socket
rather than on the octets crossing it. It went in during the fifth round, when a
reviewer pointed out that the conditional MUST had been chosen over a bare
SHOULD precisely because it is observable, and that nothing in §7 observed it.
The rule was sound and its justification was not delivered.

§2 names no mechanism, and the two a client can use read different things: peer
credentials report the listening process, and an ownership check reads the path.
The eleventh round found the fixture supplying only the owner, which passes a
peer-credential client against a driver listening as itself, so the fixture now
supplies both.

**Why a client's names are not normative, and why that costs a fixture.** The
names a client reports never cross the socket, so no peer can depend on them and
a MUST would bind something no other implementation can observe. A host wants
its own vocabulary as well: a PAM client returns a PAM code, and one wanting a
distinct code per condition subdivides. So the names are a SHOULD and the
conditions are what bind.

The price is that a driver cannot read the outcome of a client-direction case
from anything the wire carries. §7 asserted those outcomes for several rounds
with nothing supplying either the client's own names or the channel they arrive
on, which made the client half of the suite rest on a premise the document never
stated. F7 is that premise written down. It is a precondition of a kind of case
rather than of any one case, which is why the *needs* column does not carry it:
doing so would mean marking every row that reads an outcome from a client, and a
later case added of that kind would silently be marked wrong.

The ninth round narrowed the kind. F7 used to claim every case driven against a
client, and a case whose assertions are all on the socket does not need it: one
asserts that the driver reads end of file and that its response is accepted for
writing, another that a field on the wire carries its octets unchanged. Neither
reads the client's own names or the channel they arrive on.

That ruling reached F7's own entry and this passage and not the sentence in §7
that generalises it, which went on claiming every case driven against a client
for a round. The tenth round found the two halves of one ruling disagreeing,
which is the shape this document keeps producing when a change lands in more
places than the person making it visited.

**Why the client's line sits at one octet and the server's at two.** The two
thresholds answer different questions. A server needs `ludsa_version_major`
before it knows which version to answer in, so a truncated word leaves it
nothing to say and it says nothing. A client is not answering. It only has to
tell a server that never spoke from one that spoke and stopped, and the first
octet settles that.

§7 used to call these the same line. Reading them as one puts a client's
one-octet case under `RESPONSE_ABSENT`, which reports a server that failed
mid-response as one that was never reached. Those send an operator to different
places: the first to a crashing server, the second to a socket path, a peer
allowlist, or a server that is not running.

Case 24 exists because case 15 cannot reach the one-octet case: its condition is
that `ludsa_version_major` has been read, which is a server's threshold applied
to both sides. The pair is complete now, with §7's rule that a request stalling
below two octets draws no response covering the server's half.

**Why a client separates unsupported from malformed.** Both are defects in the
server, and §4.1 freezes the response types a request permits, so neither can
arise from a version gap within major version 1. The split is not about who is
at fault. It is about where the reader looks next: a `msg_type` the client has
never heard of sends someone to compare the two builds, and one the client knows
but should not have received sends them to the server's response path.

It also rescues a case. Before the split, an unimplemented `msg_type` and a
repeated tag both came back `RESPONSE_MALFORMED`, so §7 case 22 could not tell a
client that honoured §4's ordering from one that did not, and the case was
marked `either` while testing nothing on that side. Now the two orderings give
two different names, and the case earns its marking. The expected answer flipped
afterwards, when the repeat check moved behind `msg_type`, but the case still
turns on which of the two a receiver reaches first.

**Why a third selector case pins the order.** §4 says the order between steps 8
and 9 is not observable, because both yield the same status, and a receiver
taking them the other way round conforms. Steps 11 and 12 are not like that. A
`service` the server does not recognise answers in a `STATUS_RESPONSE`, and a
`max_age` its limit forbids answers `RATE_LIMITED` in an `ERROR`, so the two
outcomes differ in message type as well as in code. An order §4 states and no
case drives is an order two implementations can take differently while both
believe they conform. The tenth round found no case driving this one. The case
supplies both conditions in one request, which is the shape the numbered table
already uses to pin steps against each other.

**Why the order binds what a receiver holds.** §4 said a message failing more
than one check takes the earliest outcome, and the sixth round then let a
receiver report a fault it had already detected rather than wait for octets that
would not change it. Both spec reviewers found the two in conflict: a server
reading as it goes answers `REQUEST_UNSUPPORTED` at the `msg_type`, and a
framing fault in the next octet would have answered `REQUEST_MALFORMED` under
the order. Both proposed making the dispatch statuses wait for the framing pass,
which reverses the sixth round's ruling and puts a server that already holds a
settled verdict back to waiting for a declared seventeen kilobytes to report it.
That is the cost the sixth round declined, and nothing about an unimplemented
type becomes truer by waiting. It is also the shape the eighth round settled for
a message both truncated and faulty: a rule either way conscripts one way of
reading a socket, and the protocol has no stake in which a receiver is. Here the
two answers reach the same PAM code, so the case for choosing is weaker than the
one the eighth round already declined. So the order is stated as binding the
outcomes a receiver holds, and case 26 is retired, since it required the answer
only one reading gives.

**Why the decision cases name a recognised `service`.** They expected a `result`
that was not `OK`, which was exact until `SERVICE_UNSUPPORTED` became a `result`
in the seventh round. An `AUTHENTICATE_REQUEST` requires a `service`, so a
driver must choose one, and nothing on those rows said which. Choosing one the
peer does not hold obliges it to answer `SERVICE_UNSUPPORTED`, which is not `OK`
and is not a decision, so both rows passed while nothing was tested. A
`ludsa-mockd` configured with no services at all certified against the only test
of §6.2's binding rule without ever examining a credential.

Naming the `AUTHN_DENIED` codes rather than "not `OK`" is what closes it, and
naming them by their prefix rather than by their numbers is what keeps it closed
if a later version adds one.

**Why only the first decision case excludes a code.** §6.2.1's line is whether
the requested name resolves, and the tenth round found the code table's own
gloss contradicting it. Once the gloss was corrected, the first case became able
to separate the two denials, but only because F3 supplies a `user` the server
holds. Without that the name might not resolve, and
`AUTHN_DENIED_USER_UNKNOWN` would be the true answer. The case still cannot
require `AUTHN_DENIED_POLICY`, since naming a reason at all is a server's
choice. The second case cannot separate them, because whether a name of zero
octets resolves is a deployment's own fact and F3 settles only that the policy
refuses it.

**Why the over-long identity case sends no `user`.** The case was written
without saying, and the only `user` F3 supplies is one the policy refuses, so a
server would deny, and §6.2's comparison runs within one `result`, which lets a
conforming server carry `authn_id` on an `OK` and not on a denial. It then has
no value to carry and no `INTERNAL_ERROR` arises. The case would have failed a
conforming server. With no `user`, and F3 supplying a credential the server
would answer `OK` to without one, the identity's length is the only thing
between the server and an `OK` carrying the field. F3 also had to say whether
the server is configured to supply `authn_id`, since one that is not answers the
same `OK` with no field and is conforming.

**Why the empty `user` case needs the fixture to say so.** An absent `user` asks
the server not to decide. It does not bind the exchange to an empty or null
name: the client may not care who the identity is, or may be applying its own
policy and delegating only validation. A `user` of zero octets is a different
request, one about the user named by the empty string, and §6.2 forbids treating
the two alike.

The failure is invisible. A server that collapses them skips its binding check
and answers `OK` on any valid credential, and nothing on the wire separates that
from a correct answer, which is why no rejection case reaches it.

The case therefore needs a deployment where absent gives `OK` and empty does
not, and that is a property of policy rather than of this protocol. A policy of
the shape "any service account in this namespace may act as any user" maps the
identity to every name, the empty one included, so a correct server answers `OK`
and the case cannot see the defect. Naming the requirement in F3 turns that from
a silent false pass into a peer the case does not apply to, on the same footing
as a server without rate limiting and case 18.

**Why a client applies the same conditions but not the same names.** Most of §3
binds both sides, so a client detects the same conditions in a response. It has
nothing to reply to, having already sent its request and half-closed, so §7's
table gives it the conditions and §7's response codes give it the names. That
makes a library's contract fall out rather than be invented.

It used to give it the names too, by having a client report the status the
server would have sent. That was wrong in one direction: a caller reading
`REQUEST_TOO_LARGE` could not tell whether to shorten its own credential or to
go and look at the server.

**Why `RESPONSE_TOO_LARGE` exists and the other conditions fold into
`RESPONSE_MALFORMED`.** The client-local names shadow §5's statuses, and the
shadow had one hole: nothing paired with a response declaring a length above
`MAX_LENGTH`, so §7 asserted a pairing it did not supply, twice. Folding it into
`RESPONSE_MALFORMED` was the free repair and was declined, because the two
conditions differ in what a client had to do to find them. A client reports
`RESPONSE_TOO_LARGE` off the envelope, having parsed nothing, which is the same
line §5 draws on the server's side. Everything a client finds inside the body,
an over-long field included, is `RESPONSE_MALFORMED`.

**Why a client relays every status it read.** §7 named four to relay, which left
the `REQUEST_` statuses and `STATUS_UNKNOWN` with nowhere to go, while Appendix
B mapped all of them on the stated ground that a status reaches a client only in
an `ERROR` message. A client meeting `REQUEST_TOO_LARGE` had a status §5 forbade
it to report for a fault it found itself, and no client-local name either. §5's
prohibition was always about a fault the client found, and a status a server
emitted is that server's statement about a request the client did send. The list
became the property, which is what the closed list should have been from the
start. The property carries two routes, since a status a client does not
recognise reaches it under no name the server sent: it relays what it read, or
records `STATUS_UNKNOWN` for a value §3.4 gives it no name for.

## Appendix A: derived message maxima

**Why derived figures are written down at all.** They are recomputable from §3.1
and §6.4, so in principle nobody needs them. In practice `IMPLEMENTATION.md`
claimed `STATUS_RESPONSE` was 33301 octets for as long as it took someone to
notice, which was several review rounds after the field that made it that large
had been deleted. A figure nobody states is a figure nobody checks.

**Why the table disclaims its own authority.** Two sources of truth that can
disagree are worse than one that is inconvenient. The table says §3.1 and §6.4
win, and the tag verifier recomputes it, so drift is caught rather than trusted.

**Why it says what a peer emits rather than what a receiver accepts.** The
figures are the obvious thing to validate against, and validating against them
is the one use that breaks the protocol: §4.1 lets a minor version add optional
fields, so a receiver that rejects anything above 534 rejects a conforming peer
one release later. Stating the distinction where the numbers are is cheaper than
correcting it afterwards.

## Appendix B: mapping to PAM

**Why a host binding sits in a normative document at all.** Nothing in the
appendix is a wire rule, and §7 cannot test any of it. It is there because two
PAM clients that map the same outcome differently give an operator a stack that
behaves differently against each, with an identical wire trace both times. That
is a real interoperability failure with no other home: `IMPLEMENTATION.md` is
non-normative and read by nobody by design, so a module author working from the
specification alone would never meet it.

**Why the `account` line carries a requirement.** `AUTHN_DENIED_POLICY` returns
`PAM_SUCCESS` from `pam_sm_authenticate`, so a stack without an `account` line
admits a caller the server denied. That is a fail-open produced by a convention
rather than by a wire rule, and no conformance case can see it. Binding the
client to document it puts the obligation on someone the specification can bind,
which a rule aimed at a deployment would not.

**Why the `OK` row is qualified.** The table exists so two PAM clients behave
alike, and its `OK` row said `PAM_SUCCESS` without condition. §6.2 forbids a
client that sent no `user` from treating `OK` as an authorisation decision, so a
client following the table is the one that breaks the rule, and it breaks it
open. The row now says what it presumes, and points at the obligation a client
on that path still owes.

**Why `AUTHN_UNKNOWN` stops the stack.** It mapped to `PAM_AUTHINFO_UNAVAIL`,
the code the appendix itself names as the one an operator keys a fall-through
on. §4.1 lets a minor version add a result code and §3.4 has a predating client
read it as `AUTHN_UNKNOWN`, so a server upgrade turned every denial carrying the
new code into a fall-through on every deployed client, with nothing on the
client changed and nothing to detect it. The appendix already answers a code
that says nothing, `AUTHN_DENIED`, with the outcome that assumes least, and
§6.2.1 puts `AUTHN_UNKNOWN` on the credential side on the same ground: a client
that cannot read the answer has not been told the credential was good.
`PAM_SYSTEM_ERR` denies as well, and the appendix keeps it for something that
needs a person rather than a different credential. This client read the
envelope, the message type and the response it expected, and only the outcome
value was unassigned. What a later version adds to `result` is an outcome of an
authentication attempt, so an authentication failure is the likelier report. The
thirteenth round asked that question, which this entry had not answered. The
cost is that a client meeting a broken server stops its stack where it used to
fall through, which is the direction a value it cannot read should fail in.
`STATUS_UNKNOWN` keeps the fall-through, since a status reports how a server
handled a message rather than what it decided.

**Why the `account` phase fails closed with no verdict.** The mapping table
presupposes a `result`, and for ten rounds nothing said what `pam_sm_acct_mgmt`
returns when it holds none for this transaction, or holds one obtained for a
different `PAM_USER`. A stack whose `auth` line belongs to another module
reaches this module's `account` phase with nothing cached whenever that module
succeeds, and the appendix's whole case for `PAM_SUCCESS` at
`pam_sm_authenticate` on `AUTHN_DENIED_POLICY` is that the `account` phase
denies it. The eleventh round made it a rule: the verdict must come from the
same transaction and be for the user now asked about, and with none the module
fails closed. `PAM_SYSTEM_ERR` is the code to prefer because a stack in that
shape is a configuration needing a person, which is the split the appendix
already draws. `PAM_IGNORE` is forbidden alongside `PAM_SUCCESS` because it lets
the rest of the stack decide as though the module were not there. The project
had treated this as a rule from the start, and only the working notes carried
it.

The thirteenth round added the path with no `PAM_USER` to key on. The rule
presumed a `user` for the same reason the `OK` row did, and the twelfth round
stated that presumption in the row and left it unstated here. A client on §6.2's
optional path holds a verdict keyed to an identity it derived, so it holds
nothing for the user it is now asked about and could never pass its own account
phase. `pam_ludsa` always sends `user`, which is why nothing reached it, and the
appendix binds any PAM client that adopts the mapping.

**Why the three peer-identity outcomes share a code.** A socket the client
cannot open, a peer that is not the one the client expects, and a server that
refuses the peer it got are one condition seen from three positions: these two
processes are not permitted to talk, because a configuration says so. They all
map to `PAM_AUTHINFO_UNAVAIL`. An earlier draft sent `PEERCRED_DENIED` to
`PAM_SYSTEM_ERR`, which made the server's half of the check stop a stack that
the client's half and the filesystem's half let fall through.

The objection to falling through is that an attacker who can provoke a denial
forces the stack on to whatever is configured next. That is true, and it is
equally true of a wrong socket mode and of an unreachable server, both of which
already fall through, so it argues against the whole `PAM_AUTHINFO_UNAVAIL`
design rather than against this row. The narrower version is better: a peercred
denial reproduces reliably where an unreachable directory is transient, so an
operator who sees only a fall-through never learns the allowlist is wrong. That
is an argument for logging it, not for a different code.

**Why the table is a default rather than a fixed mapping.** The appendix exists
because two clients mapping one outcome differently give an operator a stack
that behaves differently against each, on an identical wire trace. That argues
for a stated default, not for a fixed one. A deployment may reasonably want a
peer refusal to stop its stack where another wants it to fall through, and the
choice belongs to whoever runs it.

Naming defaults and permitting an override serves both: two clients nobody has
configured behave alike, and a deployment that needs something else says so
rather than patching a module. The one thing an override may not do is turn a
failure into a success, which is the only direction in which a configuration
mistake here is unrecoverable.

**Why the override bound names the entry point.** The prohibition was stated as
a test of what a whole stack does, which is what the emphasised cell needs:
`PAM_SUCCESS` at `pam_sm_authenticate` for an authorisation outcome admits
nobody, because the `account` phase denies. Read as the bound on overrides, the
same sentence let a deployment defer any denial that way, since the stack still
denies wherever an `account` line exists. That multiplies the appendix's one
fail-open by every code, on the omission the appendix already says a module
cannot detect. The bound now names the entry point: no override returns
`PAM_SUCCESS` where the table does not, and the deferral stays a cell the
appendix chose rather than a pattern it licensed. What a deployment loses is a
rearrangement nobody has asked for.

The peer-identity outcomes are asked to move together for the same reason they
share a default. An administrator deciding that a peer refusal should stop the
stack means every view of it a client can name, not the half the server happened
to detect. The tie reaches a `CONNECTION_FAILED` the client got by refusing the
server's identity or by being refused the socket, and stops there. Both are
permanent until somebody edits a configuration, which is the property the tie
turns on, and the fourteenth round widened it to the second after finding this
entry still counting three positions where Appendix B named two. It cannot reach
a server that refuses in silence. §5 lets one close without replying, and the
client is then left with `RESPONSE_ABSENT`, which §2 makes indistinguishable
from a server that failed, so the default is the only thing that governs it. The
tenth round widened that name to any connection a client was unwilling to use,
so it now also covers conditions that clear on a retry, and a socket that is not
there yet is the ordinary startup-order case. One setting across the whole name
would make a boot race stop a stack that an administrator had configured against
a misconfiguration. The thirteenth round found that, and the reviewer who found
it was reading the objection record rather than this file.

**Why the rest is non-normative.** No mapping is right for every host.
`pg_ludsa` has no PAM, and a client with a host that distinguishes more finely
than PAM does should use its own names. Making the table a MUST would bind
clients the protocol has no business binding, and would freeze under §4.1 a
choice worth revisiting.

## 8. Non-goals

**No confidentiality or integrity on the wire.** The socket's permissions are
the security boundary. A peer able to write to it can already assert any
verdict, so encrypting the channel would only protect against an attacker who by
construction no longer needs to attack.

**No PAM conversation.** LUDSA cannot drive prompts, so it cannot support
interactive flows. This is deliberate: the target is non-interactive credential
presentation.
