Release notes
=============

This document describes all the changes made to the *Authenticating Clients
with HTTP Signature* document, starting from its first released version.


1.0.1
-----

* Added a notice for the server implementers to *ignore* unsigned request
  headers. (This hasn't been previously stressed enough, and it could lead to
  security vulnerabilities.)

* Added a notice for client implementers to take care not to allow their
  frameworks and proxies modify the request after it has been signed (as this
  could break the signature).


1.0.0
-----

Upgraded to a stable version.


0.4.0
-----

* Loosened requirements on HTTP response code when invalid `Authorization`
  header found. HTTP 400 is also a valid response in this case.

* Loosened requirements on the contents of the `WWW-Authenticate` header.

* Clarified that it's not necessary to verify if `keyId` matches a proper
  format. It's sufficient to check if it has been registered in the Registry
  (and respond with HTTP 403 if it hasn't).

* Added a new server verification step: Verify `X-Request-Id`.


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
