# SSB secure partial replication

Status: Design phase

This document outlines the steps needed to convert an existing SSB
identity to make it ready for partial replication. First a root meta
feed will be generated from the existing SSB feed as described in
[ssb-meta-feed]. The main feed is linked with the meta feed which
means an application can start using the feeds linked from the meta
feed for partial replication.

The new meta feed should contain the following entries:

```
{ type: 'metafeed/add', feedpurpose: 'main', subfeed: '@main.ed25519' },
{ type: 'metafeed/add', feedpurpose: 'indexes', subfeed: '@indexes.bbfeed-v1' }
{ type: 'metafeed/add', feedpurpose: 'indexaudits', subfeed: '@audits.ed25519' }
{ type: 'metafeed/add', feedpurpose: 'trust', subfeed: '@trust.bbfeed-v1' }
{ type: 'metafeed/add', feedpurpose: 'fusionidentities', subfeed: '@fusion.ed25519' }
```

## Indexes

Indexes is a meta feed of feeds linking to a subset of messages in
another feed. These can be used for partial replication of the main
feed. This means they are only used when synchonizing a feed for the
first time. After this, EBT should be used to keep the feed in sync.

The feeds inside this meta feed should only contain hashes of the
original messages as their content. These index feeds can be
replicated efficiently as described in [subset replication].

Applications using the main feed should create at least two index
feeds:

```
{ 
  type: 'metafeed/add', 
  feedpurpose: 'index', 
  subfeed: '@index1.ed25519', 
  querylang: 'ssb-ql-1', 
  query: '{ op: 'and', args: [{ op: 'type', string: 'contact' }, { op: 'author', feed: '@main.ed25519' }] }' 
}

{ 
  type: 'metafeed/add', 
  feedpurpose: 'index', 
  subfeed: '@index2.ed25519', 
  querylang: 'ssb-ql-1', 
  query: '{ op: 'and', args: [{ op: 'type', string: 'about' }, { op: 'author', feed: '@main.ed25519' }] }' 
}
```

For the definition of the query language see [ssb-ql-1].

Index message format:

```
{ type: 'metafeed/index', indexed: %hash }
```

If an index feed is created under the same meta feed as the main feed,
then no other nodes needs to create an index feed unless the index
feed contains an error or becomes stale. One the other hand if it is
not the same, then 3 nodes in the network should each create an index
feeds, this is to decrease the chances of an index growing stale.

## Index audits

Indexes can be seen as a *claim*, in that, these are the messages
matching a query. It is important that these are accurate and thus
other nodes in the network should audit these. Thus catching malicious
nodes but also simple programming errors.

An auditor verifies an index by being in possesion of the same mesages
and verifies that no messages are left out and that the index is not
growing stale. Anyone can be an auditor, trust is assigned to auditors
as described in the next section. This should keep bad auditors out of
the system.

After verifying an index feed, a message is posted on the audit feed:

```
{ 
  type: 'index/audit', 
  latestseq: x, 
  subfeed: '@index1.ed25519', 
  metafeed: '@mf.bbfeed-v1', 
  status: 'verified' 
}
```

Because the messages on feeds are immutable, once a feed has been
verified up until sequence x, the past can never change. This holds
true because the network will only accept new messages on a feed that
correctly extends the existing feed.

In order not to create too many verification messages, a new message
should only be posted if an index is no longer valid or it has grown
stale. How often indexes should be verified is at the discretion of
the auditor.

An index feed is considered stale if that has not been updated a week
after a message is posted on the indexed feed. These should be posted
as:

```
{ 
  type: 'index/audit', 
  latestseq: x, 
  subfeeed: '@index2.ed25519', 
  metafeed: '@mf.bbfeed-v1', 
  status: 'stale',
  reason: 'not updated since 2021-06-25'
}
```

In case an index is invalid, the following kind of message should be
posted:

```
{ 
  type: 'index/audit', 
  latestseq: x, 
  subfeeed: '@index2.ed25519', 
  metafeed: '@mf.bbfeed-v1', 
  status: 'invalid',
  reason: 'Missing the message %hash.sha256 with sequence 100'
}
```

In the case where an index is no longer valid, the index feed and all
messages referenced from this feed should be removed from the local
database. After this, another index needs to be found an used instead.

It is worth noting that it is limited what a malicious peer could
do. The messages referenced in an index still needs to be signed by
the author, so at worst messages can be left out.

## Trust

Trust is a meta feed that contains one feed for each trust area with
ratings within that area as defined in [trustnet]. One area where this
will be used is for delegating trust related to verification of
indexes:

```
{ 
  type: 'metafeed/add', 
  feedpurpose: 'trustassignments', 
  subfeed: '@assignments.ed25519',
  area: 'indexaudits'
}
```

A trust assignment:

```
{ 
  type: 'trustnet/assignment', 
  src: '@mf.bbfeed-v1', 
  dest: '@othermf.bbfeed-v1',
  weight: 1.0 
}
```

Notice we use the meta feed as the destination. The index subfeed of
the destination is given by the area and layout of meta feeds.

# SHS Connections

One question that arises once we have multiple identities and multiple 
apps which use different subfeeds, is: which cryptographic keypair do
we use for secret-handshake connections? The main (classic) feed? The
meta feed? The subfeed used by the application?

An initial solution to this problem would be to connect with any ID
you wish, and then via replication of the metafeed, other peers could
discover what other IDs you are equivalent to, and this way they
could infer who you are. 

There is one important situation where that solution would not work: 
room servers. A room, by design, does not replicate any feed, and it 
authenticates peers through an allow-list of specific SSB IDs. So if
the room recognizes you by your main feed, you are still not 
authorized to connect using your meta feed or some subfeed, and the
room cannot replicate your metafeed either.

Another important situation to keep in mind is that some apps may
choose to have a separate database that only contains the subfeed
relevant to that app's use case, and maybe some other minor metadata
such as replicating the metafeed and perhaps the `secret` files for
other identities. But we have to assume that the database will not
contain the entire "main" feed, because it is too large and the app
is interested in partial replication.

To put this discussion into a high-level perspective, let us define 
two different concepts here: **use case** and **identity**. 

**Identity** is essentially your cryptographic "fingerprint" or your
"face", it's always the same everywhere you go, you cannot change it 
and people can see who you are. But **use case** is like the social 
context, for instance work context versus party context versus family 
gathering context. You may want to talk to me about work in a party, 
but I may refuse to do that. When changing social contexts, however, 
my identity remains the same.

We can treat these app situations as merely different use cases, 
while the identity remains the same. Like social contexts aren't 
neatly separated (there may be a work *party*, or a family 
*business*), we can't assume that use cases will be neatly separated 
either. So it's okay to allow some implicitness when Bob wants to 
determine what is Alice's use case. Alice will connect as 
Alice/main, but her use case may be strictly only chess. Or, there 
may be an app that combines social use cases (such as Patchwork 
features), while *also* allowing chess use cases. For instance 
Patchbay. It may allow both use cases.

We shouldn't conflate identity with use case, even though they may 
overlap. For instance, the chess app can prompt you to insert your 
24 words of your main feed (in order to use that for shs), but you 
may choose to skip that and use a completely separate newly-created 
identity (for shs), just for chess. That means you are creating a 
new identity **for that** use case. This would be akin to using a 
mask in a party, being a pseudonym in a social context.

In practical terms in the SSB stack, use the main feed ID (or 
whatever ID most of your peers recognize you by) as the **identity** 
for secret-handshake connections, and infer the **use case** via
other means, such as trial-and-error calling some muxrpc APIs.

# Fusion identity

Implementation of the [fusion identity spec]

# Replication

Here we expand on how feeds are replicated within a system with meta
feeds.

Current replication is taking care of by [ssb-friends] based on follow
or block relations as described in that repo.

With meta feeds, contact messages on the main feed still form the
basis for feed replication. From this basis the parts of the meta feed
needed for the application should also be replicated. Linked feeds
should still be followed, this is to ensure backwards compatibility
with existing clients.

Assuming one wants to do partial replication of a subset of a feed,
first one tries to find the meta feed using the rpc getSubset from
[subset replication] on the main feed author and type
'metafeed/announce'. Then downloading the meta feed to try and find
index feeds.

If that is not found, one uses trusted index audits feeds to find an
index that can be used. Trusted is defined as:

A target feed is trusted if:
 -  One has assigned any positive, non-zero amount of trust to the
    target feed, or
 - the trustnet calculation returns the feed as trusted

A trustnet calculation is performed as:
 - All positive, non-zero first order (direct) trust assignments are always
   returned as trusted.
 - If there are no first-order trust assignments with a trust weight exceeding
   `TrustNet.trustThreshold`, computation is shortcircuited and only the first
   order trust assignments are returned. 
 - If there is at least one first-order trust assignment exceeding the trust
   threshold, then [Appleseed] ranks are calculated for the given trust graph.
 - TrustNet's trusted feeds are then calculated by breaking the Appleseed
   rankings into 3 ckmeans clusters, and discarding the cluster of with the lowest ranks. 
 - If any direct trust assignments are not included in the top 2 clusters, then
   they are added to the concatenation of the top 2 clusters before returning
   the result as the trustnet calculation.

If no verified indexes are available one should fall back to full
replication of that main feed.

A random new user will at first not trust anyone and thus can't do
partial replication. On the other hand if they are onboarded using
someone they know and trust that would enable partial replication.

Lets look at how onboarding could work for Alice that got invited by
Bob. First alice downloads Bobs main feed. She then trusts Bob and
downloads Bobs meta feed, all trust assignments, indexes and
audits. By using the metafeed field on trust assignments, Alice is
able to recursively download trust assignments, their audits and from
that decide what indexes to be used.

## Open questions

- Should pubs also use meta feeds?
- How do we handle other feed types?
- What initial trust should be assigned and to what?
- Do we need a rotational feed for the latest messages?


[ssb-meta-feed]: https://github.com/ssb-ngi-pointer/ssb-meta-feed
[Appleseed]: https://github.com/cblgh/appleseed-metric 
[trustnet]: https://github.com/cblgh/trustnet
[ssb-friends]: https://github.com/ssbc/ssb-friends
[subset replication]: https://github.com/ssb-ngi-pointer/ssb-subset-replication
[private-groups]: https://github.com/mixmix/ssb-tribes
[fusion identity spec]: https://github.com/ssb-ngi-pointer/fusion-identity-spec
[ssb-ql-1]: https://github.com/ssb-ngi-pointer/ssb-subset-replication-spec#query-language
