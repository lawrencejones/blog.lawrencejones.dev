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
that the move was safe, and would be transparent to GC users.

Finding an answer to this question involved writing a small tool called
[pgreplay-go](https://github.com/gocardless/pgreplay-go), a remake of an
existing tool [pgreplay](https://github.com/laurenz/pgreplay) with a specific
focus on our performance testing needs. While only a week of work, the project
covered a broad range of learning opportunities for me, whether that be
using observability tooling in my development processes or improving my
operational knowledge of Postgres.

This post will break-down the project, explaining why the original pgreplay tool
didn't meet our needs, then digging into the process of building and using
pgreplay-go.
