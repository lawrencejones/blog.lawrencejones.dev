---
layout: post
title:  "pgreplay-go: Postgres load testing"
date:   "2019-02-24 11:00:00 +0000"
tags:
  - postgres
excerpt: |
  TODO

---

While working on a GoCardless infrastructure migration, we found ourselves
searching for a way to de-risk the part involving a Postgres cluster. The
GoCardless' API is critically dependent on this database and any performance
degredation would impact the entire service - it was important to be confident
that the move was safe, and would be transparent to GC users, despite moving our
cluster away from predictable bare-metal machines to Cloud VMs.

Finding an answer to this question involved writing a small tool called
[pgreplay-go](https://github.com/gocardless/pgreplay-go), a remake of an
existing tool [pgreplay](https://github.com/laurenz/pgreplay) with a specific
focus on our performance testing needs. While just a week of work the project
became an opportunity for me to develop technically, from integrating
observability tooling in development to improving my operational knowledge of
Postgres.

This post will break-down the project, explaining why the original pgreplay tool
didn't meet our needs, then digging into the process of building and using
pgreplay-go.

# Our pgreplay strategy

We had previously used [pgreplay](https://github.com/laurenz/pgreplay) to verify
configuration and hardware changes. pgreplay consumes a log of Postgres queries
taken from your production service and replays them against a replica of that
database. By replaying production traffic against our new cluster, we could use
[pgBadger](https://github.com/darold/pgbadger) to qualify how performance had
changed and highlight query regressions.

This process had been successful in the past, but broke almost immediately when
applying it to this migration. Watching Postgres activity during a replay, there
was a sheer drop after ten minutes of execution:

![pgreplay waiting on long-running query](pgreplay-blocking.png)

When replaying queries, pgreplay will ensure they execute in the order from the
input log - any queries that take longer to execute in the replay will block the
execution of subsequent queries. The graph shows how a single query that has
significantly degraded in our new cluster has blocked pgreplay from continuing
its execution.

No matter how much we might want them to, users don't respond to a system under
load by forming a queue and politely waiting for their first query to finish -
more likely they click again, packing yet more stress on an overloaded system.
This replay strategy would allow problematic queries to execute in an
artificially peaceful moment (all other queries have completed execution) and
we'd miss the effect the on-going query might have on those that follow, which
all compete for the same system resource.

Our previous changes had often been straight-up improvements, like provisioning
more memory or upgrading CPU architecture. Clearly this migration wasn't going
to behave like the others, and we could expect more varied changes in query
performance. After trying several work-arounds to improve the blocking, we
concluded a new strategy was required to predict the impact of this migration
before sending real users into the system.

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
