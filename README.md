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
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'main', id: @main },
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'claims', id: @claims }
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'linked', id: @linked }
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'trust', id: @trust }
```

Claims is a meta feed of claims or indexes, meaning feeds consisting
of a subset of messages in another feed. These can be used for partial
replication. The feeds inside this meta feed should only contain
hashes of the original messages as their content. These linked
messages can be replicated efficiently as auxiliary data during
replication as described in [subset replication].

Applications should create at least two index feeds:

```
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', id: '@claim1', query: 'and(type(contact),author(@main))' }
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', id: '@claim2', query: 'and(type(about),author(@main))' }
```

Trust is also meta feed that contains one feed for each trust area
with ratings within that areas as defined in [trustnet]. One area
where this will be used is for verifications of claims. 

```
{ type: 'metafeed/operation', operation: 'add', feedtype: 'classic', purpose: 'claimaudits', id: @claimaudits }
```

An auditor verifies claims of other feeds and writes messages of the
following form:

```
{ type: 'trustnet/claims/verification', latestid: x, id: @claim1, metafeed: @claims, status: 'verified' }
```

Because feeds are immutable once you have verified a feed up until
sequence x, the past can never change. In order not to create too many
verification messages, a new message should only be posted if claim is
no longer valid. How often claims should be verified is at the
discretion of the auditor.

In the case where a claim is no longer valid, the claim feed and all
messages referenced from this feed should be removed from the local
database. After which they need to be downloaded again as described
later in the document. It is worth noting that it is limited what a
malicious peer could do. The messages in a claim still needs to be
signed by the author, so at worst messages can be left out which for
contact messages and about messages does not seem like a big deal.

Linked is a meta feed that contains links to other feeds. The use case
for this is same-as where other SSB ids can be linked. This allows
applications to use this information to create a better user
experience, such as showing notifications for linked feeds, showing
the linking feeds on a profiles page etc. If the linked feed links
back, the feeds are considered transitively linked.

```
{ type: 'about/same-as-link', from: '@mf', to: '@otherid' }
```

We here should expand on how feeds are replicated within a system with
meta feeds. Existing replication is taking care of by [ssb-friends]
that replicates feeds based on follow or block relations as described
in that repo.

Existing contact messages on the main feed still form the basis for
feed replication. From this basis the parts of the meta feed needed
for the application should also be replicated. Linked feeds should
still be followed, this is to ensure backwards compatibility with
existing clients.

Assuming one wants to do partial replication of a subset of a feed,
one uses trusted claimaudits feeds combined with the meta feeds they
link to find one that can be used. Trusted is defined as:

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

A new user will at first not trust anyone and thus can't do partial
replication. On the other hand if they are onboarded using someone
they know and trust that would enable partial replication.

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
