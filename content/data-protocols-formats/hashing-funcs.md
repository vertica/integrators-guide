---
title: "Hashing functions"
linkTitle: "Hashing functions"
weight: 3
---

## Uses of Hashing

Vertica has several kinds of hashing:

- Hashing for internal local purposes, like GROUP BY HASH in memory
- Hashing for internal local/subcluster redistribution, like GROUP BY
  HASH across processors/nodes
- Hashing for placing projection data on shards/nodes
- Hashing for end-user or specific applications

The first three are NOT hash functions in the cryptographic sense... it is quite easy to cause collisions in them, and typically to reverse them. Some functions in the latter category are more like proper hash functions, with definitions and characteristics that are well known and documented elsewhere.