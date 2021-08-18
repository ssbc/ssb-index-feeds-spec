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

```
{ type: 'metafeed/index', indexed: '%msgkey' }
```

# Fusion identity

Implementation of the [fusion identity spec]

# Replication

Current replication is taken care of by [ssb-friends] and 
[ssb-replication-scheduler] based on follow or block relations as 
described in the latter repo.

With meta feeds, contact messages on the main feed still form the
basis for feed replication. From this basis the parts of the meta feed
needed for the application should also be replicated. 

Assuming one wants to do partial replication of a subset of a feed,
the following steps should apply.

**1. Discover and replicate the meta feed.** Given the main feed ID 
(but not the main feed messages themselves) of a friend, we first need 
to the root meta feed ID of that friend. To achieve that, we use the 
RPC `getSubset` from [subset replication] on any connected peer, 
asking for a message of type `'metafeed/announce'` on our friend's 
main feed. If connected peers do not support this RPC, then we can 
fallback to `createHistoryStream` to fetch the friend's full feed 
in memory (i.e. without persisting it to the log) and then filter for
the first message of type `'metafeed/announce'`. Once we find that
message, we will know the feed ID for our friend's root meta feed, and
we can replicate it in full, like we do with any other feed.

**2. Scan through the meta feed, looking for relevant sub feeds.** The
meta feed, once replicated, will reveal a tree of sub feeds owned by
the friendly peer. Our local peer should have some application-specific
configurations that determine which sub feeds are relevant for replication
and which ones are to be ignored. These configurations should be matched
against the replicated meta feed, to identify which sub feeds are we about
to replicate.

**3. Replicate a sub feed in full or in slices.** Once we know we want to
replicate a sub feed, the application-specific configuration should also
determine whether to replicate it in full (as traditionally), or in
slices, e.g. only the N latest messages in the sub feed. The former is
easy to achieve with classic SSB libraries, but the latter requires 
adaptations in [ssb-ebt] so that we signal to remote EBT peers that we
already have all older messages before the N latest, while in reality
we don't have them. That is, we need new events in EBT that merely 
signal what is the clock state of feeds, without immediately requesting
updates for those feeds. Finally, once the sub feed is replicated, if 
it was a meta feed, then we may need to go back to step 2 to comb over 
the sub sub feeds.

**4. Special treatment for replicating index feeds.** Most sub feeds are
trivial to replicate, but index feeds are different, because their 
messages point to messages in another feed (let's call it the target feed),
and we are ultimately interested in the contents of the messages on the 
target feed. To handle this, we first replicate index feeds in full, and
persist the messages to the log. Then, we run a variant of EBT where 
`getAt` is implemented in terms of the index feed messages, and `append`
will do a database `addOOO` for messages on the target feed. This EBT
variant will require its own `replicate` duplex RPC in a separate muxrpc 
namespace not shared with the conventional `ebt.replicate`, because the
data passed through the duplex will be different.
 
## Open questions

- Should pubs also use meta feeds?
- How do we handle other feed types?
- Do we need a rotational feed for the latest messages?

[ssb-meta-feed]: https://github.com/ssb-ngi-pointer/ssb-meta-feed
[bendy-butt]: https://github.com/ssb-ngi-pointer/bendy-butt-spec
[ssb-friends]: https://github.com/ssbc/ssb-friends
[ssb-ebt]: https://github.com/ssbc/ssb-ebt
[ssb-replication-scheduler]: https://github.com/ssb-ngi-pointer/ssb-replication-scheduler
[subset replication]: https://github.com/ssb-ngi-pointer/ssb-subset-replication
[fusion identity spec]: https://github.com/ssb-ngi-pointer/fusion-identity-spec
[ssb-ql-0]: https://github.com/ssb-ngi-pointer/ssb-subset-replication-spec#ssb-ql-0
