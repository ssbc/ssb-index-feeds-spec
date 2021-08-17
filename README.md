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
