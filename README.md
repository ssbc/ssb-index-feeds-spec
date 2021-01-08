# SSB secure partial replication

Status: Design phase

This document will outline the steps needed to convert an existing SSB
identity to make it ready for partial replication. First a root meta
feed will be generated from the existing SSB feed as described in
[ssb-meta-feed]. The identity of a user should now be the meta feed.
For backwards compatibility with existing follow messages, a link to a
main feed in a root meta feed is considered the same as a follow to
the root meta feed itself.

A contact follow message to the meta feed will be posted in the
existing feed for discoverability. The new meta feed contains a couple
of entries:

```
Main: { type: add, feedtype: classic, id: @main }
Derived: { type: add, feedtype: classic, id: @derived }
Linked: { type: add, feedtype: classic, id: @linked }
Trust: { type: add, feedtype: classic, id: @trust }
```

Trust is a feed that contains trust ratings as defined in [trustnet]
about other feeds and will be used to evaluate if claims can be used
or not. It is possible to assign trust ratings to any feed in a meta
feed included the meta feed itself. The trust rating will apply to the
feed and all feeds transitively linked from this. By assigning a
rating on a feed in the tree, it is possible to overwrite the rating
of an inherited value.

Derived is a meta feed of claims, meaning feeds consisting of a subset
of messages in another feed. These can be used for partial
replication.

Linked is a meta feed that contains links to other feeds. The use case
for this is same-as where other SSB ids can be linked. This allows
applications to use this information to create a better experience,
such as showing notifications for linked feeds, linking profiles
etc. Automatic trust can be assigned if the linked feeds links back.

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

[ssb-meta-feed]: https://github.com/ssb-ngi-pointer/ssb-meta-feed
[trustnet]: https://github.com/cblgh/trustnet
[subset replication]: https://github.com/ssb-ngi-pointer/ssb-subset-replication
