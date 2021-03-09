# SSB secure partial replication

Status: Design phase

This document will outline the steps needed to convert an existing SSB
identity to make it ready for partial replication. First a root meta
feed will be generated from the existing SSB feed as described in
[ssb-meta-feed]. As described in that document the main feed is linked
with the meta feed which means an application can start using the
feeds linked from the meta feed for partial replication.

The new meta feed should contain the following entries:

```
{ operation: 'add', feedtype: 'classic', purpose: 'main', id: @main },
{ operation: 'add', feedtype: 'classic', purpose: 'claims', id: @claims }
{ operation: 'add', feedtype: 'classic', purpose: 'linked', id: @linked }
{ operation: 'add', feedtype: 'classic', purpose: 'trust', id: @trust }
```

Trust is a feed that contains trust ratings as defined in [trustnet]
about other feeds and will be used to evaluate if claims can be used
or not. It is possible to assign trust ratings to any feed belonging
to a meta feed included the meta feed itself. The trust rating will
apply to the feed and all feeds transitively linked from this. By
assigning a rating on a feed deeper in the tree, it is possible to
overwrite the rating of an inherited value.

Claims is a meta feed of claims or indexes, meaning feeds consisting
of a subset of messages in another feed. These can be used for partial
replication. The feeds inside meta feed would only contain hashes of
the original messages as their content, and linked messages can be
replicated efficiently as auxiliary data during replication as
described in [subset replication].

FIXME: define exactly what claims feed we need: contacts, about, do we
need a feed for the latest messages?

Linked is a meta feed that contains links to other feeds. The use case
for this is same-as where other SSB ids can be linked. This allows
applications to use this information to create a better experience,
such as showing notifications for linked feeds, showing the linking
feeds on a profiles page etc. Automatic trust can be assigned if the
linked feeds links back.

```
{ type: 'about/same-as-link', from: '@mf', to: '@otherid' }
```

Assuming one wants to do partial replication of a subset of a feed,
first one looks in derived feeds in ones network to see if any has a
high enough trust rating (FIXME: define) to be used. If not, if a
connection is establed with another peer that support [subset
replication] and is trusted (FIXME: define) then use that to get the
data. Lastly if none of these options are available fall back to full
replication.

## Open questions

- When should derived feeds should be generated, will this be
  different between normal clients and pubs?
- What does this mean for following? If we detect a meta feed with a
  main feed that we already follow, should the follow automatically be
  upgraded to the main feed. Or should this only be done on feed
  rotation of the main feed?
- How do we handle other feed types?
- What initial trust should be assigned and to what?
- When and how will trust be calculated for derived feeds?

[ssb-meta-feed]: https://github.com/ssb-ngi-pointer/ssb-meta-feed
[trustnet]: https://github.com/cblgh/trustnet
[subset replication]: https://github.com/ssb-ngi-pointer/ssb-subset-replication
