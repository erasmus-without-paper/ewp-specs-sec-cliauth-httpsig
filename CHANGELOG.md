Release notes
=============

This document describes all the changes made to the *Authenticating Clients
with HTTP Signature* document, starting from its first released version.


0.3.0
-----

* Use public key digests instead of actual keys in `keyId` parameter of the
  `Authorization` header (see
  [this issue](https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-httpsig/issues/1)).

* Allow `Original-Date` to be used in place of the `Date` header (see
  [this issue](https://github.com/erasmus-without-paper/ewp-specs-sec-srvauth-httpsig/issues/1)).


0.2.0
-----

* Make it usable with the v2 security.

* Add `security-entries.xsd` file (which describes the XML element used to
  identify this method of authentication in the manifest files, and the
  Registry).

* Upgraded `draft-cavage-http-signatures-06` to
  `draft-cavage-http-signatures-07`. Still no RFC though.


0.1.0
-----

Initial revision.
