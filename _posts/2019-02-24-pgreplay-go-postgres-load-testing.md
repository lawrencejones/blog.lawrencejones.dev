---
layout: post
title:  "pgreplay-go: Postgres load testing"
date:   "2019-02-24 11:00:00 +0000"
tags:
  - postgres
excerpt: |
  TODO

---

While working on an infrastructure migration, my team found ourselves searching
for a way to de-risk the part involving a Postgres cluster. The API we were
moving was critically dependent on this database and any performance degredation
would impact the entire service - it was important to be confident that the move
was safe, and would be transparent to our users, despite making significant
changes to how we were running the cluster.

Finding an answer to this question involved writing a small tool called
[pgreplay-go](https://github.com/gocardless/pgreplay-go), a remake of an
existing project [pgreplay](https://github.com/laurenz/pgreplay) with a specific
focus on our performance testing needs. While just a week of work the project
was a great opportunity to experiment with new tooling, such as incorporating
observability into local development and improving operational knowledge of
Postgres.

This post will break-down the project, explaining why the original pgreplay tool
didn't meet our needs, then digging into the process of building and using
pgreplay-go.

## Our pgreplay strategy

We had previously used [pgreplay](https://github.com/laurenz/pgreplay) to verify
configuration and hardware changes. pgreplay consumes a log of Postgres queries
taken from your production service and replays them against a replica of that
database. After replaying production traffic against our new cluster, we could
use [pgBadger](https://github.com/darold/pgbadger) to quantify how performance
had changed and detect query regressions.

This process had worked well in the past but broke almost immediately when
applying it to this migration. Watching Postgres activity during a replay, there
was a sheer drop after ten minutes of execution:

![pgreplay waiting on long-running query](pgreplay-blocking.png)

When replaying queries, pgreplay will ensure they execute in the order from the
input log - any queries that take longer to execute in the replay will block the
execution of subsequent queries. The graph shows how a single query that has
significantly degraded in our new cluster has blocked pgreplay from continuing
its execution.

These benchmarks take several hours to run, so having the replay back-up against
a problematic query increases the time required for each config iteration. It
also impacts the realism of your tests: users don't respond to a system under
load by forming a queue and politely waiting for their first query to finish.

Going back to the purpose of our benchmarking, we wanted to know what would
happen when our production demands were served by this new hardware, especially
if the system is going to fail under ever increasing load. After a few attempts
to workaround this constraint with pgreplay, it became clear a new strategy was
required to properly predict the impact of handling real user traffic in the new
cluster.

## An ideal tool

So if this isn't going to work, what might? What we really wanted was a replay
strategy that modelled each connection separately, isolating the effect of the
bad query to a single connection instead of halting the entire replay.

This tool would have a few components: a Postgres log parser which constructs
the work to replay, a streamer that can pace execution to match the original
logs and a database that can model on-going connections which consume the
incoming replay items. Whatever we implement, it needed to be fast (perhaps a
compiled language?) and use a runtime with cheap concurrency, in order to model
each connection.

We needed the benchmark results within a week which meant there were three
(leaving two days for running the benchmarks) days to achieve a working tool
that met our objectives. It would be tight, but three days felt doable to build
out this Goldilocks replay tool, especially if I chose a language I was already
familiar with.

In a burst of optimistic naivety, I decided it was worth giving it a shot.

## Log parsing: how hard can it be?

Before we ever start talking with Postgres we'll need to parse the logs to
figure out what to replay. My initial take on this was that parsing would be
easy- this was a silly assumption, and I soon found myself questioning my
language of choice (Go).

### Multi-line tokenising

An example Postgres log line looks like so:

```
2019-02-25 15:08:27.239 GMT|alice|pgreplay_test|5c7404eb.d6bd|LOG:  statement: insert into logs (author, message)
	values ('alice', 'says hello');
```

Ideally you'd just split your log file by newlines to find each entry, but the
Postgres errlog format doesn't work like this. As many log entries contain
newlines, the errlog format allows for newline characters by prefixing
continuation lines with a leading tab character (`\t`).

Parsing files with a known separation character (think `\n`) is much easier than
this sort of problem. Instead of scanning the file looking for a thing, you're
scanning until you see "not a thing", in this case until you see a line that
doesn't begin with a leading tab. This requires you to look ahead from where
your current token will terminate, and this complicates the logic around when to
split.

Golang has an interface called a [`Scanner`](https://golang.org/pkg/bufio/#Scanner)
that can be used to tokenise a stream of characters using a custom splitter
function. After defining a [splitter function](https://github.com/gocardless/pgreplay-go/blob/v0.1.2/pkg/pgreplay/parse.go#L346-L406)
full of error-prone mutation (wondering why I wasn't using a more modern
interface like `strtok`) it was now possible to parse the Postgres logs into
individual entries.

Feeling like parsing was almost done, I began categorising log lines into
different replay events and promptly hit another roadblock.

### Simple vs Extended Query Protocol

Postgres clients can opt to use one of two query protocols to issue commands
against the server. The first is the [simple](https://www.postgresql.org/docs/11/protocol-flow.html#id-1.10.5.7.4)
protocol, where the client sends a text string to the server containing the SQL
with all parameters already interpolated into the query string. Simple queries
will be logged like this:

```
[1] LOG:  statement: select now();
```

Simple queries are easy: we parse the query from this line and execute it
against the server. The extended query protocol is much harder to parse, as
query execution happens in multiple stages.

The extended query protocol is intended to provide a safe method of
interpolating parameters into SQL queries. The client first provides the SQL
query with parameter placeholders, then follows with values for those paramters.
Executing `select $1, $2` with parameters `alice` and `bob` would yield the
following logs:

```
[1] LOG:  duration: 0.042 ms  parse <unnamed>: select $1, $2;
[2] LOG:  duration: 0.045 ms  bind <unnamed>: select $1, $2;
[3] DETAIL:  parameters: $1 = 'alice', $2 = 'bob'
[4] LOG:  execute <unnamed>: select $1, $2;
[5] DETAIL:  parameters: $1 = 'alice', $2 = 'bob'
[6] LOG:  duration: 0.042 ms
```

---


This is significantly more cumbersome than log
entries being delimited 

The default Postgres errlog format can be parsed by first tokenising into
individual log entries, then parsing the contents of each entry. Postgres
supports CSV log output but the benchmark log capture we wanted to replay had
been taken in an errlog format that looked like this:

In BNF, you'd define this grammar as:

```
<entry>   ::= <timestamp> "|" <user> "|" <database> "|" <session-id> "|" <content>
<content> ::= <line> "\n" ( "\t" <line> )
<line>    ::= [^\n]
```

The first challenge is how Postgres handles multi-line log entries. As we're
logging SQL queries, and SQL queries often contain newlines, Postgres denotes a
trailing log entry line by prefixing the entry with a leading `\t` character.
We therefore need to scan our logfile not by line, but by line without a leading
`\t`.

---

# Emulating system overload

clear at this point that the
we'd made between our clusters, but this migration wasn't going to be so easy.
We needed a tool that better replicated our disaster scenario, where we
performed the migration and our cluster couldn't keep up with production
traffic, leading to system overload.

# What's changing?

Our Postgres cluster consists of three machines, one primary (accepting writes)
a sync and an async. The migration won't be changing this configuration, nor
will we be changing Postgres or associated software versions.

While we'll provision the same CPU and memory across both clusters, we'll make
at least three significant changes that can impact database performance: we'll
be moving from single-tenant bare-metal machines to multi-tenant
virtual-machines; the new cluster will span multiple availability zones,
increasing network latency from 0.1ms to 0.35ms; our storage will change from
RAID-10 local SSDs to networked GCP persistent storage.

This means 

# What is safe?

The Postgres cluster we were migrating is at the heart of every GoCardless API
request. For us to know this migration was safe, we needed to guarantee no part
of our service would be see significant degradation that might be noticed by our
users.

You might be asking

The Postgres cluster we'll focus on is configured with three nodes, one primary
(accepting writes) a sync and an async. This means any successful write is
guaranteed to have been written to two durable locations before acknowledgement
(n+1 safety). Our migration won't alter this migration in any way.



Our legacy Postgres cluster is was running on a collection of three bare metal
machines living in the same data center. The cluster was configured as a
primary, sync and async, ensuring n+1 safety for all writes. We wanted to move
this cluster into a collection of GCP machines running across three availability
zones, preserving the same primary/sync/async configuration.
