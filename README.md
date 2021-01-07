# SSB secure partial replication

Status: Design phase

This document will document the steps needed to convert an existing
SSB identity to make it ready for partial replication. First a meta
feed will be generated from the existing SSB feed as described in
[ssb-meta-feed]. In this new meta feed a couple of entries will be
added:

```
Main: { type: add, feedtype: classic, id: @main }
Derived: { type: add, feedtype: classic, id:@derived }
Linked: { type: add, feedtype: classic, id: @linked }
Trust: { type: add, feedtype: classic, id: @trust }
```

Trust is a feed that contains trust ratings as defined in [trustnet]
about other feeds and will be used to evaluate if claims can be used
or not. It is possible to assign trust ratings to any part of a meta
feed included the meta feed itself. A more specific feed will
overwrite the rating of a less specific, so a rating of the Derived
feed would overwrite the meta feeds overall rating.

Derived is a meta feed of claims, meaning feeds consisting of subset
of other feeds. These can be used for partial replication.

Linked is also a meta feed that contains links to other feeds. This
could be other SSB ids such as mobile clients or even hypercore feeds,
such as a cabal identifier.

Assuming one wants to do partial replication of a subset of a feed,
first one would look in derived feeds in ones network and see if any
has a high enough trust rating (FIXME: define) to be used. If not, if
a connection is establed with another peer that support [subset
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

[ssb-meta-feed]: https://github.com/ssb-ngi-pointer/ssb-meta-feed
[trustnet]: https://github.com/cblgh/trustnet
[subset]: https://github.com/ssb-ngi-pointer/ssb-subset-replication
