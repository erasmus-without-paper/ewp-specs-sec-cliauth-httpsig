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
   - `date` or [`original-date`][original-date-header] (it MUST contain at
     least one of those, it also MAY contain both),
   - `digest`,
   - `x-request-id`.

   Note, that you MUST allow `headers` parameter to contain *more* values than
   the required ones listed here.

If one or more of these conditions is not met, then the client doesn't use this
method of authentication, or is using it incorrectly. In this case:

 * If you support other client authentication methods at this endpoint, then
   you SHOULD check if the client doesn't use any of those.

 * If the client doesn't use any of the supported authentication methods, then
   you MUST respond with HTTP 400 or HTTP 401 error response, with a proper
   `<developer-message>` (it SHOULD describe the reason why you consider the
   request to be invalid).

   If HTTP Signature Authentication is the preferred method of authentication
   at this endpoint, then is it RECOMMENDED to respond with HTTP 401, and
   include the following headers:

   - `WWW-Authenticate`, as described [here][httpsig-www-authenticate].  Your
     `realm` value SHOULD be `EWP`.
   - `Want-Digest`, as described [here][want-digest].

   For example:

   ```http
   WWW-Authenticate: Signature realm="EWP"
   Want-Digest: SHA-256
   ```


### Verify the host

You MUST verify that the value of the `Host` header included in the request
matches your own domain (the one on which the endpoint is served).

This verification is sometimes handled by the web server itself, and you simply
won't receive requests which don't match your host. If you're not sure, then
verify it in your code anyway.


### Look up the key

Extract the `keyId` from the request's `Authorization` header. (In case of
problems, respond with HTTP 400 error message.)

The key MUST match at least one of the **public client keys** published in the
[Registry Service][registry-api]. Consult [Registry API][registry-api] for
information on how to find a match. If you cannot find a match, then you MUST
respond with HTTP 403 error response. As usual, including a proper
`<developer-message>` is RECOMMENDED.


### Verify the date(s)

You need to parse and verify the values of the `Date` and
[`Original-Date`][original-date-header] headers, if they are included in the
request (at least one of them MUST be). In particular, you MUST verify at least
the ones which have been listed in the request's `Authorization` header, but it
is RECOMMENDED to verify both (if both are included in the request).

Verification process consists of parsing the date, and matching it against the
date reported by your own server clock. The verification fails if:

 * The date cannot be parsed.

 * The date does not match your server clock **within a certain threshold of
   time**.

   - It is RECOMMENDED to use the **5 minutes** threshold.
   - You MAY choose a greater threshold than 5 minutes, but you MUST NOT choose
     a lower threshold than this.

If the verification fails, then you MUST respond with HTTP 400 error response.
Your error response SHOULD include a proper `<developer-message>`.

Also note, that you MUST make sure that your clock is synchronized (otherwise
your clients won't be able to use your service).


### Verify the nonce (optional)

This step is OPTIONAL if TLS is used for transport (in EWP, most APIs require
TLS to be used). However, if TLS is not used (or cannot be trusted for any
reason), then this step is REQUIRED in order to prevent replay attacks. Also
see *Security Considerations* section below.

The `X-Request-Id` header can be used as a [cryptographic
nonce](https://en.wikipedia.org/wiki/Cryptographic_nonce). If you decide to
implement nonce verification, then you MUST reject requests who's nonce has
already been used. You will need to store the set of nonces which have been
used. Thanks to the `Date` header (or `Original-Date` header), you don't have
to store the nonces which have been used earlier than 5 minutes ago.


### Verify `X-Request-Id`

Even if you don't verify the nonce in the previous step, it is still
RECOMMENDED to verify the format of the `X-Request-Id` header. It SHOULD be an
an UUID formatted in a canonical form, e.g. `dc05b425-4e86-4106-8dde-1257fccf53e5`.
If it's not, then you SHOULD respond with HTTP 400 error response.

If you're wondering why we are recommending this, then it's because we want to
force clients to use proper values in their `X-Request-Id` header. We expect
that most servers won't be verifying nonces at this time, and without this
recommendation, some clients might be tempted to include "dummy" values in
their `X-Request-Id` header, and no servers would notice that. We want the
network to collectively prevent that.


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


### Ignore unsigned headers

Servers MUST ignore all request headers which hadn't been signed by the client.
They might have been added by the attacker in transport. This is important
especially in cases when TLS is not used in some parts of the transport, or
when you don't fully trust the partner's TLS implementation.

The safest way to properly ignore such headers is to modify your `Request`
object *now* (during the authentication and authorization process), by either
removing the suspicious headers, or at least changing their name (e.g.
prepending it with `Unsigned-`). Then, pass the modified `Request` along as the
result of your authentication, so that the actual API you are implementing
believes that the client didn't supply these headers at all. This approach is
much safer than trusting yourself to remember to verify this every time before
you access every header in every single ones of your APIs (and some APIs might
be dependent on request's headers).


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
by registry, and the keys (and/or their fingerprints) are served to all other
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

 * You MUST include either the `Date` or [`Original-Date`][original-date-header]
   header in your request. You MAY include both of them. The format of the
   `Original-Date` header, if included, MUST match the "regular" format of the
   `Date` header, as defined in [RFC 2616][date-header]. You MUST make sure
   that your clock is synchronized (otherwise your request may fail).

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
 - `date` or `original-date` (it MUST contain at least one of those, it also
   MAY contain both),
 - `digest`,
 - `x-request-id`.

If it is important for the server to recognize any other of your headers, then
you MUST sign all these headers too. The headers mentioned above are important
for handling authentication, non-repudiation and security described in this
document, but other headers also MAY be very important in your case.

The `keyId` parameter of the `Authorization` header MUST contain a
HEX-encoded SHA-256 fingerprint of the *public key* part of the key-pair which
you have used to sign your request. It MUST match one of the keys you
previously published in your manifest file.


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
(`Original-Date`) and `X-Request-Id` headers, partners MAY implement additional
security measures to prevent such attacks.


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
[original-date-header]: https://github.com/erasmus-without-paper/ewp-specs-sec-srvauth-httpsig#original-date
