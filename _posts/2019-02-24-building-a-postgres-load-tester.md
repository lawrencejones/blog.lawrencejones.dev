---
layout: post
title:  "Building a Postgres load tester"
date:   "2019-02-24 11:00:00 +0000"
tags:
  - postgres
excerpt: |
  TODO

---

While working on an infrastructure migration I came across a need to benchmark a
Postgres cluster. In an attempt to create realistic load, I ended up writing a
tool that replays captured Postgres activity against a live server, providing an
opportunity to predict how queries might degrade with configuration changes.

This post will discuss the implementation of pgreplay-go, a tool to
realistically simulate captured Postgres traffic. I'll explain why an existing
tool didn't meet our needs and cover some challenges in the implementation.

## Existing tools

When faced with a technical problem, the first step should be to check if an
existing tool can solve it rather than writing it yourself. It happened that
benchmarking a Postgres cluster was something I'd done before, and had used a
tool called [pgreplay](https://github.com/laurenz/pgreplay) to do it.

The design of pgreplay is very simple. You first capture logs from your
production Postgres cluster, then use pgreplay to play them back against a
different target. The captured logs provide a bottled image of production
traffic and post-processing of how the new cluster performed under this load can
help determine if changes are going to degrade performance.

This process had worked well in the past but broke almost immediately when
applying it to this migration. Watching Postgres activity during a replay, there
was a sheer drop after ten minutes of execution:

![pgreplay waiting on long-running query](pgreplay-blocking.png)

The changes in this migration were significant enough that several queries had
degraded in the new cluster. When replaying queries, pgreplay will ensure they
execute in the order from the input log - any queries that take longer to
execute in the replay will block the execution of subsequent queries. What the
graph shows is how a single badly degraded query has blocked pgreplay from
continuing execution until it completes.

Benchmarks can take several hours to run so having the replay back-up against a
problematic query adds yet more time to a slow running process. It also impacts
the realism of your tests: users don't respond to a system under load by forming
a queue and politely waiting for their first query to finish.

## Our ideal tool

So if this isn't going to work, what might? It seemed to me that a better replay
strategy would be one that modelled each connection separately, where
long-running queries would only impact their own connection. This allows us to
measure the effects of our rampaging query is having on the rest of our
production traffic while getting on with the rest of the replay.

Much like the original pgreplay, a tool that achieves this would have just a few
components: a Postgres log parser which constructs the work to replay, a
streamer that can pace execution to match the original logs and a database that
can model on-going connections. The tool would need to be fast (perhaps a
compiled language?) and use a runtime with cheap concurrency in order to model
each connection.

The benchmark results were needed within a week, meaning there were three
(leaving two days for running the benchmarks) days to build something that met
our objectives. It would be close, but three days felt doable to create this
Goldilocks replay tool, especially with a language I was already familiar with.

In a burst of optimistic naivety, I decided it was worth giving it a shot.

## Log parsing: how hard can it be?

By necessity, we start with log parsing. My initial thought was this would be
easy - I couldn't have been more wrong, and I soon found myself questioning my
choice of Go as the implementation language.

### Multi-line tokenising

Ideally you'd split your log file by newlines to find each entry but the
Postgres errlog format doesn't work like this. As many log entries contain
newlines, the errlog format allows for newline characters by prefixing
continuation lines with a leading tab character (`\t`):

```
2019-02-25 15:08:27.239 GMT|alice|pgreplay_test|5c7404eb.d6bd|LOG:  statement: insert into logs (author, message)
	values ('alice', 'says hello');
```

This type of parsing is more fiddly than splitting on newlines. Instead of
scanning your file looking for a 'thing' (`\n`), you're scanning until you see
'not a thing' (not leading `\t`). This requires the scanner to look ahead of
where your current token will terminate, complicating the logic around when to
split.

Given this was a small, simple project, I was keen to avoid heavy-weight
parser-generators that would require an additional build step. I instaed reached
for the standard library to tokenise my log lines - after all, Go has an
interface called a [`Scanner`](https://golang.org/pkg/bufio/#Scanner) for
exactly this purpose.

Several hours and some considerable frustration later, I had a [splitter function](https://github.com/gocardless/pgreplay-go/blob/v0.1.2/pkg/pgreplay/parse.go#L346-L406)
full of error-prone mutation and a distinct feeling that `strtok` would have
been less painful. This was something I hadn't considered when choosing Go for
the implementation, but the stdlib support for string maniuplation is really
poor compared to what I'm used to. It caught me off-guard that Go would be so
poorly suited to the task.

Regardless, I now had something that could parse individual log entries and felt
confident the worst was over. Next problem please!

### Parsing log items (Simple vs Extended)

Holy crap, this doesn't work like I thought it would.

Postgres clients can opt to use one of two query protocols to issue commands
against the server. The first is the [simple](https://www.postgresql.org/docs/11/protocol-flow.html#id-1.10.5.7.4)
protocol, where the client sends a text string to the server containing the SQL
with all parameters already interpolated into the query string. Simple queries
will be logged like this:

```
[1] LOG:  statement: select now();
```

These are easy - we just parse the query from this line and execute it against
the server. It's the extended protocol that gets hard.

Extended query protocol provides safe handling of query parameters by divorcing
the SQL query from the injection of parameter values. The client first provides
the query with parameter placeholders, then follows with values for those
paramters.  Executing `select $1, $2` with parameters `alice` and `bob` would
yield the following logs:

```
[1] LOG:  duration: 0.042 ms  parse <unnamed>: select $1, $2;
[2] LOG:  duration: 0.045 ms  bind <unnamed>: select $1, $2;
[3] DETAIL:  parameters: $1 = 'alice', $2 = 'bob'
[4] LOG:  execute <unnamed>: select $1, $2;
[5] DETAIL:  parameters: $1 = 'alice', $2 = 'bob'
[6] LOG:  duration: 0.042 ms
```

Log lines 1, 2 and 3 represent the initial preparation of this query, where
Postgres creates an unamed prepared statement and binds our two parameters.
Query planning occurs during the bind stage, but it's at line 4 that we know our
query has begun execution.

So ignoring all but lines 4 and 5, how would we parse these log lines executable
instructions? We can see that logging the execution of a prepared statements is
immediately followed by a `DETAIL` log entry that describes the parameter
values, but we'll need to combine these together before we know what query to
execute.

It's now worth discussing the interfaces we'll be using for parsing within our
project. The primary parsing interface is `ParserFunc`, which takes a Postgres
errlog file and produces a channel of replay `Item`'s.

```go
type ParserFunc func(io.Reader) (
  items chan Item, errs chan error, done chan error
)

var _ Item = &Connect{}
var _ Item = &Disconnect{}
var _ Item = &Statement{}
var _ Item = &BoundExecute{}

type Item interface {
    GetTimestamp() time.Time
    GetSessionID() SessionID
    GetUser() string
    GetDatabase() string
    Handle(*pgx.Conn) error
}
```

We expect to run our parser across very large (>100GB) log files and lazily
emit the parsed `Item`'s as soon as we parse them. Of the four categories of
`Item` we can parse, `BoundExecute` represents the combination of a prepared
statement and it's query parameters.

As prepared statements with query parameters come in two log lines, it's
possible for our parsing process to successfully parse a log line (like the
`execute` entry) but the item is not complete, as it lacks parameters. Modelling
this concept required another type to represent an `Execute` entry:

```go
type Execute struct {
  Query string `json:"query"`
}

func (e Execute) Bind(parameters []interface{}) BoundExecute {
  return BoundExecute{e, parameters}
}
```

Unlike all the other types, `Execute` does not satisfy the `Item` interface as
it lacks a `Handle` method, so we'll never be able to send it down our results
channel (`chan Item`). As we parse our items, we track the last recognised
`Execute` log line against each on-goinging connection's `SessionID`, and we
pass this mapping to our `ParseItem` function so that it can match subsequent
`DETAIL` entries against unbound executes:

```go
func ParseItem(logline string, unbounds map[SessionID]*Execute) (Item, error) {
    ...

    // LOG:  execute <unnamed>: select pg_sleep($1)
    if strings.HasPrefix(msg, LogExtendedProtocolExecute) {
        unbounds[details.SessionID] = &Execute{details, parseQuery(msg)}
        return nil, nil
    }

    // DETAIL:  parameters: $1 = '1', $2 = NULL
    if strings.HasPrefix(msg, LogExtendedProtocolParameters) {
        if unbound, ok := unbounds[details.SessionID]; ok {
            // Remove the unbound from our cache and bind it
            delete(unbounds, details.SessionID)
            return unbound.Bind(ParseBindParameters(msg)), nil
        }
    }
}
```

By maintaining the cache of unbounds in a stateful parsing function, we can
support the two-stage process of combining execute and detail lines into a
single item. By representing our `Execute` entry as a type without a `Handle`
method, we can leverage Go's type system to enforce and communicate the
incompleteness of this log entry, which helped the compiler point out where I
had incorrectly sent these items to the database to be processed.

While tricky and much more complicated than I expected, it was nice to find a
solution to the parsing that can be expressed nicely as types, and framing the
problem like this helped me find a good implementation.
