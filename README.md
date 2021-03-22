# SSB secure partial replication

Status: Design phase

This document will outline the steps needed to convert an existing SSB
identity to make it ready for partial replication. First a root meta
feed will be generated from the existing SSB feed as described in
[ssb-meta-feed]. The main feed is linked with the meta feed which
means an application can start using the feeds linked from the meta
feed for partial replication.

The new meta feed should contain the following entries:

```
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'main', id: '@main' },
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'indexes', id: '@indexes' }
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'claims', id: '@claims' }
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'claimaudits', id: '@claimaudits' }
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'trust', id: '@trust' }
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'linkedidentity', id: '@linked' }
```

## Indexes

Indexes is a meta feed of feeds linking to a subset of messages in
another feed. These can be used for partial replication of the main
feed. The feeds inside this meta feed should only contain hashes of
the original messages as their content. These linked messages can be
replicated efficiently as auxiliary data during replication as
described in [subset replication].

Applications should create at least two index feeds:

```
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', id: '@index1', query: 'and(type(contact),author(@main))' }
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', id: '@index2', query: 'and(type(about),author(@main))' }
```

## Claims and audits

Clients supporting meta feeds should create indexes as described above
for their own main feed. In order to still do partial replicating of
older feeds it might make sense for others to create claims and audit
that these claims are indeed valid indexes.

Claims are written in much the same way as indexes:

```
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', id: '@claim1', query: 'and(type(contact),author(@other))' }
```

An auditor verifies claims of other feeds and writes messages of the
following form to the the claimaudits feed:

```
{ type: 'claims/verification', latestseq: x, id: @claim1, metafeed: @mf, status: 'verified' }
{ type: 'claims/verification', latestseq: x, id: @claim2, metafeed: @mf, status: 'invalid' }
```

Because feeds are immutable, once you have verified a feed up until
sequence x the past can never change. In order not to create too many
verification messages, a new message should only be posted if claim is
no longer valid. How often claims should be verified is at the
discretion of the auditor.

In the case where a claim is no longer valid, the claim feed and all
messages referenced from this feed should be removed from the local
database. After this they need to be downloaded again as described
later in the document. It is worth noting that it is limited what a
malicious peer could do. The messages in a claim still needs to be
signed by the author, so at worst messages can be left out.

## Trust

Trust is also a meta feed that contains one feed for each trust area
with ratings within that areas as defined in [trustnet]. One area
where this will be used is for delegating trust related to
verification of claims:

```
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', area: 'claimaudits', id: '@claimaudits' }
```

A trust assignment from you to another feeds claimaudits feed would be:

```
{ type: 'trustnet/assignment', src: '@main', dest: '@otherclaimaudits', metafeed: '@othermf', weight: 1.0 }
```

FIXME: should we use @mf instead of @main for src?

# Linked identity

Linked is a meta feed that is used to maintain virtual identities this
feed is a member of. It works by consensus, meaning as long as all the
members of a virtual identity are mutually linked, the identity is
valid. Any member of the identity can revoke the identity by creating
a tombstone message.

Anyone can create a new identity by first creating a keypair and then
announcing the identity:

```
{ type: 'linked/create', identity: '@id', name: 'arj' }
```

Then the identity can be linked between metafeeds:

```
{ type: 'linked/link', from: '@mf', to: '@othermf' }
```

Once @othermf posts a similar message, the identity is linked and the
creator of the identity should send the private key of the identity to
the new member.

The identity can be extended with members by having all current
members linking the new feed and the new feed linking back.

Any member can revoke the identity by posting the following message:

```
{ type: 'linked/tombstone', identity: '@id' }
```

Once another member sees this message they should also post a
tombstone message, this is to make it harder for an adversary try and
keep the identity alive after one of the feeds has been compromised.

Lastly the name can be changed in a consensus fashion as well:

```
{ type: 'linked/name', identity: '@id', name: 'arj' }
```

A new feed added to the group can merge these messages by including
the name in the link message.

These linked identities thus act as public groups that can also be
used for same-as between multiple physical devices belonging to the
same person.

The goal of linked identities is for small groups where the members
should be publicly known, for larger groups [private-groups] should be
considered instead.

It might also be possible to operate with groups where instead of full
censensus only a quorum is needed. Imagine you have groups of groups,
where instead of having each member of a linked identity ack a link
between the identity and another linked identity, you would only have
say 2/3 of the members do that.

For a good starting point for existing discussions on SSB going back 5
years (linked in the thread):
%YaWEWHDWAY6p/g9zIwCJovsd1SUyHpwuGGz3Ug/jtW8=.sha256

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
first one tries to find the meta feed (FIXME: describe in detail) and
from that the index feed. 

If that is not found, one uses trusted claimaudits feeds combined with
the meta feeds they link to find one that can be used. Trusted is
defined as:

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

If no verified claims are available one should fall back to full
replication of that main feed.

A random new user will at first not trust anyone and thus can't do
partial replication. On the other hand if they are onboarded using
someone they know and trust that would enable partial replication.

Lets look at how onboarding could work for Alice that got invited by
Bob. First alice downloads Bobs main feed. She then trusts Bob and
downloads Bobs meta feed, all trust feeds, claim audits and the linked
feed. By using the metafeed field on trust assignments, Alice is able
to recursively download trust assignments, their claim audits and from
that decide what claims can be used.

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
