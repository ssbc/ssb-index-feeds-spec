# SSB secure partial replication

Status: Design phase

This document outlines the steps needed to convert an existing SSB
identity to make it ready for partial replication. First a root meta
feed will be generated from the existing SSB feed as described in
[ssb-meta-feed]. The main feed is linked with the meta feed which
means an application can start using the feeds linked from the meta
feed for partial replication.

The new meta feed can contain the following messages (described
as [bendy-butt] bencode dictionaries):

```
{ "type" => "metafeed/add", "feedpurpose" => "main", "subfeed" => (BFE feed ID) }
{ "type" => "metafeed/add", "feedpurpose" => "indexes", "subfeed" => (BFE Bendy Butt feed ID) }
{ "type" => "metafeed/add", "feedpurpose" => "fusionidentities", "subfeed" => (BFE feed ID) }
```

## Indexes

Indexes is a meta feed of feeds linking to a subset of messages in the
main feed. These can be used for partial replication of a subset of
the main feed. They should only be used when synchonizing a feed for
the first time. After this, EBT should be used to keep the feed in
sync.

The feeds inside this meta feed should only contain hashes of the
original messages as their content. These index feeds can be
replicated efficiently as described in [subset replication].

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

```json
{
  type: 'metafeed/index',
  indexed: {
    key: '%msgkey',
    sequence: 42
  }
}
```

The `indexed.sequence` might seem redundant but it might be much cheaper
for an implementation to resolve `author@sequence` and checking the hash of the message then keeping a total `hash => message` index.

# Fusion identity

Implementation of the [fusion identity spec]

# Replication

Current replication is taking care of by [ssb-friends] based on follow
or block relations as described in that repo.

With meta feeds, contact messages on the main feed still form the
basis for feed replication. From this basis the parts of the meta feed
needed for the application should also be replicated. 

Assuming one wants to do partial replication of a subset of a feed,
first one tries to find the meta feed using the rpc getSubset from
[subset replication] on the main feed author and type
'metafeed/announce'. Then downloading the meta feed to try and find
index feeds.

## Open questions

- Should pubs also use meta feeds?
- How do we handle other feed types?
- Do we need a rotational feed for the latest messages?


[ssb-meta-feed]: https://github.com/ssb-ngi-pointer/ssb-meta-feed
[bendy-butt]: https://github.com/ssb-ngi-pointer/bendy-butt-spec
[ssb-friends]: https://github.com/ssbc/ssb-friends
[subset replication]: https://github.com/ssb-ngi-pointer/ssb-subset-replication
[fusion identity spec]: https://github.com/ssb-ngi-pointer/fusion-identity-spec
[ssb-ql-0]: https://github.com/ssb-ngi-pointer/ssb-subset-replication-spec#ssb-ql-0
