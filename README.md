Authenticating Clients with HTTP Signature
==========================================

This document describes how to accomplish EWP client authentication with the
use of HTTP Signatures.

* [What is the status of this document?][statuses]
* [See the index of all other EWP Specifications][develhub]


Introduction
------------

This client authentication method makes use of:

 * The `Digest` header, specified in [RFC 3230][digest-base].
 * The `SHA-256` Digest Algorithm, specified in [RFC 5843][digest-sha256].
 * The `Authorization` header, specified in
   [draft-cavage-http-signatures-07][httpsig-authorization] (presumably,
   [soon to become RFC](https://github.com/erasmus-without-paper/ewp-specs-architecture/issues/17#issuecomment-286142084)),
   along with its `rsa-sha256` signature algorithm.
 * Optional "Nonce & Timestamp" replay attack prevention.


Implementing a server
---------------------

The order of the following steps seems optimal, but you MAY perform them in
different order, if you wish (as long as you perform all of them).


### Verify authentication method used

Make sure that:

 * The client's request contains the `Authorization` header with the
   `Signature` method, as specified [here][httpsig-authorization].

 * The `algorithm` parameter of the `Authorization` header is equal to
   `rsa-sha256`. You MAY support other algorithms too, but `rsa-sha256` is
   (currently) the only one required.

 * The `headers` parameter of the `Authorization` header contains **at least**
   the following values:

   - `(request-target)`,
   - `host`,
   - `date`,
   - `digest`,
   - `x-request-id`.

   Note, that you MUST allow `headers` parameter to contain *more* values than
   the required ones listed here.

If one or more of these conditions is not met, then the client doesn't use this
method of authentication, or is using it incorrectly. In this case:

 * If you support other client authentication methods at this endpoint, then
   you SHOULD check if the client doesn't use any of those.

 * If the client doesn't use any of the supported authentication methods, then
   you SHOULD respond with HTTP 401 with a proper `<developer-message>` (it
   SHOULD describe the reason why you consider the request to be invalid).

If HTTP Signature Authentication is the preferred method of authentication
at this endpoint, then is it also RECOMMENDED to include the following headers
in your HTTP 401 responses:

 * `WWW-Authenticate`, as described [here][httpsig-www-authenticate]. The
   `headers` parameter of your `WWW-Authenticate` header should contain the
   list of headers which are required to be signed (the ones listed above). You
   MAY use any value for `realm` (e.g. "EWP").

 * `Want-Digest`, as described [here][want-digest]. E.g.
   `Want-Digest: SHA-256`.


### Verify the host

You MUST verify that the value of the `Host` header included in the request
matches your own domain (the one on which the endpoint is served).

This verification is sometimes handled by the web server itself, and you simply
won't receive requests which don't match your host. If you're not sure, then
verify it in your code anyway.


### Look up the key

Extract the `keyId` from the request's `Authorization` header. It MUST contain
a Base64-encoded RSA public key (we use `keyId` parameter to transfer the
*actual key*, not its ID). If it doesn't, then you MUST respond with HTTP 400
error message.

The key MUST match at least one of the **public client keys** published in the
[Registry Service][registry-api]. Consult [Registry API][registry-api] for
information on how to find a match. If you cannot find a match, then you MUST
respond with HTTP 403 error response. As usual, including a proper
`<developer-message>` is RECOMMENDED.


### Verify the date

You MUST parse and verify the value of the `Date` header included in the
request. You MUST make sure that your clock is synchronized (otherwise your
clients won't be able to use your service).

If the date does not match your server clock **within a threshold of 5
minutes**, then you MUST respond with HTTP 400 error response. Your error
response SHOULD include a proper `<developer-message>`.


### Verify the nonce (optional)

This step is OPTIONAL if TLS is used for transport (in EWP, most APIs require
TLS to be used). However, if TLS is not used (or cannot be trusted for any
reason), then this step is REQUIRED in order to prevent replay attacks. Also
see *Security Considerations* section below.

The `X-Request-Id` header can be used as a [cryptographic
nonce](https://en.wikipedia.org/wiki/Cryptographic_nonce). If you decide to
implement nonce verification, then you MUST reject requests who's nonce has
already been used. You will need to store the set of nonces which have been
used. Thanks to the `Date` header, you don't have to store the nonces which
have been used earlier than 5 minutes ago.


### Verify the signature

You MUST verify the request's signature, as explained
[here][verifying-signature].

If the signature is invalid, then you MUST respond with a HTTP 400 error
response. Your error response SHOULD include a proper `<developer-message>`.


### Verify the digest

Calculate the Base64-encoded SHA-256 digest of the HTTP request's body,
according to [RFC 3230][digest-base] and [RFC 5843][digest-sha256]. Compare it
to the `Digest` header which should be present in the request. The values MUST
match.

In case of mismatch, you MUST respond with HTTP 400 error response. Your error
response SHOULD include a proper `<developer-message>`.


### Identify the covered HEIs

In most cases, you will also need to identify which HEIs are covered by the
requester (most APIs will require that). Note, that there can be any number of
them (`0..*`, see discussion
[here](https://github.com/erasmus-without-paper/ewp-specs-api-echo/issues/3)).

In the previous steps you have already found a *list* (!) of `<host>` elements
bound to the client's public key. Now, you will need to build on that
information, and retrieve *the list of HEIs these hosts cover*. Consult
[Registry API specification][registry-api] for useful hints (i.e. examples of
XPath expressions).


Implementing a client
---------------------

### Generate a key-pair

You need to generate an RSA key-pair for your client. You MAY use the same
key-pair you use for your TLS communication if you want to.


### Publish your public key

Each partner declares (in his [Manifest file][discovery-api]) a list of public
keys it will use for communicating with other hosts. This list is later fetched
by registry, and the keys (or their fingerprints) are served to all other
partners see (see [Registry API][registry-api] for details).

Usually (but not necessarily always) you will bind your public key to all HEIs
you cover. Once the server confirms that the client is in possession of a
proper private key of the certificate, it is then able to identify (with the
help of the Registry again) which HEIs such client covers.

Note, that the Registry will verify if your keys meet certain security
standards (i.e. their length). These standards MAY change in time. Remember to
include `<admin-email>` elements in your manifest file if you want to be
notified about such changes.


<a name="headers"></a>

### Include all required headers

 * You MUST include the `Date` header in your request, as defined in [RFC
   2616][date-header]. You MUST make sure that your clock is synchronized
   (otherwise your request may fail).

 * You MUST include `X-Request-Id` header in your request. It's value MUST be
   an UUID (preferably, version 4), unpredictable and unique for each request,
   formatted in a canonical form, e.g. `dc05b425-4e86-4106-8dde-1257fccf53e5`.
   If it's not unique, or it is in different format, then your request MAY
   fail.

 * You MUST include the `Digest` header in your request, as explained
   [here][digest-base]. You MUST use SHA-256 algorithm for generating the
   digest.


### Sign your request

You MUST include the `Authorization` header in your request, as explained
[here][httpsig-authorization]. You MUST use the `rsa-sha256` signature
algorithm, and you MUST include **at least** the following values in your
`headers` parameter:

 - `(request-target)`,
 - `host`,
 - `date`,
 - `digest`,
 - `x-request-id`.

If it is important for the server to recognize any other of your headers, then
you MUST sign all these headers too. The headers mentioned above are important
for handling authentication, non-repudiation and security described in this
document, but other headers also MAY be very important in your case.

The `keyId` parameter of the `Authorization` header MUST contain a
Base64-encoded RSA public key. If MUST be one of the keys you previously
published in your manifest file.


Security Considerations
-----------------------

### Replay attacks

HTTP Signatures, on their own, do not protect servers against replay attacks.
For this reason, most EWP APIs require all communication to occur over TLS.
TLS, when properly deployed, *does* protect the servers against replay attacks.

Some partners pointed out that in some cases we cannot be 100% sure that TLS
has been securely deployed by all the partners. For example, if the client's
machine is tricked into installing attacker's root certificate,
man-in-the-middle attacks are possible. In this scenario, the use of end-to-end
HTTP signatures prevents the attacker from *modifying* the message, but it
*doesn't* prevent him from executing replay-attacks. With help of the `Date`
and `X-Request-Id` headers, partners MAY implement additional security measures
to prevent such attacks.


### Non-repudiation

It's worth noting, that servers may store the request, along with its headers,
in order to be able to prove in the future that the request actually took
place. In some cases, for example approving important documents by the other
party, this feature might be handy.


### Main security questions

The [Authentication and Security][sec-intro] document
[recommends][sec-method-rules] that each client authentication method
specification explicitly answers the following questions:

> How the client's request must look like? How can the server detect that the
> client is using this particular method for authentication?

See *Implementing a client* chapter above. The server detect this method by
checking for the existence of a proper set of headers (in particular, the
`Authorization: Signature` header).

> How can the server verify which HEIs are covered by the requester?

This is described in the *Identify the covered HEIs* chapter above (in the
*Implementing a server* section).

> How can the server verify that the request has not been tampered with, nor
> replayed by a third party?

Tampering would invalidate the signature. As to the replay attacks - see a
separate chapter above.

> Does it provide non-repudiation? Can a server provide a solid proof later
> on, that a particular request took place, and that it originated from a
> certain client?

Yes. See *Non-repudiation* section above.


[discovery-api]: https://github.com/erasmus-without-paper/ewp-specs-api-discovery
[develhub]: https://developers.erasmuswithoutpaper.eu/
[statuses]: https://github.com/erasmus-without-paper/ewp-specs-management/blob/stable-v1/README.md#statuses
[digest-base]: https://tools.ietf.org/html/rfc3230#section-4.3.2
[digest-sha256]: https://tools.ietf.org/html/rfc5843#section-2.2
[httpsig-authorization]: https://tools.ietf.org/html/draft-cavage-http-signatures-07#section-3.1
[error-handling]: https://github.com/erasmus-without-paper/ewp-specs-architecture#error-handling
[httpsig-www-authenticate]: https://tools.ietf.org/html/draft-cavage-http-signatures-07#section-3.1.1
[want-digest]: https://tools.ietf.org/html/rfc3230#section-4.3.1
[registry-api]: https://github.com/erasmus-without-paper/ewp-specs-api-registry
[verifying-signature]: https://tools.ietf.org/html/draft-cavage-http-signatures-07#section-2.5
[date-header]: https://tools.ietf.org/html/rfc2616#section-14.18
[sec-method-rules]: https://github.com/erasmus-without-paper/ewp-specs-sec-intro#rules
[sec-intro]: https://github.com/erasmus-without-paper/ewp-specs-sec-intro
[cliauth-none]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-none
[cliauth-tlscert]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-tlscert
[cliauth-httpsig]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-httpsig
[srvauth-tlscert]: https://github.com/erasmus-without-paper/ewp-specs-sec-srvauth-tlscert
[srvauth-httpsig]: https://github.com/erasmus-without-paper/ewp-specs-sec-srvauth-httpsig
[reqencr-tls]: https://github.com/erasmus-without-paper/ewp-specs-sec-reqencr-tls
[resencr-tls]: https://github.com/erasmus-without-paper/ewp-specs-sec-resencr-tls
