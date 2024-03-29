<!--
SPDX-FileCopyrightText: 2021 Anders Rune Jensen

SPDX-License-Identifier: CC-BY-4.0
-->

# SSB index feeds

Status: Ready for implementation

This document outlines how index feeds are inserted into a meta feed
along with the structure of index feed messages. To understand how
index feeds are used to migrate from classic feeds to metafeeds see
[migration spec].

## Indexes

There are two versions of spec for "index feeds". Version 0 should be 
considered deprecated and does not need to be supported. Version 1
should be implemented.

### Version 1

An index feed is a feed format that in practice is essentially the classic
feed format, but has different SSB URIs for feeds and messages, to carry
different semantics and allow peers to easily distinguish them.

An index feed MUST be referred by the SSB URI `ssb:feed/indexed-v1/__` and
its messages MUST be referred by the SSB URI `ssb:message/indexed-v1/__`.

Similarly, when a metafeed is pointing to a child index feed, it MUST use
`indexed-v1` BFE identifiers. Example:

```
{ 
  "type" => "metafeed/add/derived",
  "feedpurpose" => "index", 
  "subfeed" => (An indexed-v1 BFE feed ID),
  "querylang" => "ssb-ql-0",
  "query" => '{"author":"@main.ed25519","type":"contact"}'
}
```

The query language employed SHOULD be [ssb-ql-0].

The message `content` published by an indexed-v1 feed MUST have the shape:

```js
{
  type: 'metafeed/index',
  indexed: '%msgkey'
}
```
 
### Version 0

Indexes is a meta feed of feeds linking to a subset of messages in the
main feed. These can be used for partial replication of a subset of
the main feed.

The feeds inside this meta feed should only contain hashes of the
original messages as their content.

Applications using the main feed should create at least two index
feeds:

```
{ 
  "type" => "metafeed/add",
  "feedpurpose" => "index", 
  "subfeed" => (Some BFE feed ID),
  "querylang" => "ssb-ql-0",
  "query" => '{"author":"@main.ed25519","type":"contact"}'
}

{ 
  "type" => "metafeed/add",
  "feedpurpose" => "index", 
  "subfeed" => (Another BFE feed ID),
  "querylang" => "ssb-ql-0",
  "query" => '{"author":"@main.ed25519","type":"about"}'
}
```

For the definition of the query language see [ssb-ql-0].

Index message format in a classic SSB feed:

```js
{
  type: 'metafeed/index',
  indexed: {
    key: '%msgkey',
    sequence: 42
  }
}
```

The `indexed.sequence` might seem redundant but it might be much cheaper
for an implementation to resolve `author@sequence` and checking the hash 
of the message then keeping a total `hash => message` index.

[ssb-ql-0]: https://github.com/ssb-ngi-pointer/ssb-subset-replication-spec#ssb-ql-0
[migration spec]: https://github.com/ssbc/ssb-meta-feeds-migration
