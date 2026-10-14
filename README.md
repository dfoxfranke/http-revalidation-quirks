# HTTP Cache Revalidation Quirks

This document captures pathological examples of HTTP exchanges, all thematically
by confusion over the scope of a validator, in which all parties adhere strictly
to RFC 9110 and 9111 requirements, but either are forced by specified
requirements into an undesirable outcome, or behavior required to achieve a
desirable outcome is underspecified. This document will eventually be revised
into an Internet Draft to address these issues.

## 1. Cache serves an unacceptable representation when an acceptable one is available from the origin

A cache holds a stale entry,

```
  GET /
  Accept-Language: fr

  200 OK
  Vary: Accept-Language
  ETag: W/"v1"
  Content-Language: en

  Hello world
```

This response implies that the origin likely has multiple language
representations available, but not French, so it has selected its default
English representation.

Later, a French representation is added. The cache again receives a request,

```
  GET /
  Accept-Language: fr
```

and goes forward with

```
  GET /
  Accept-Language: fr
  If-None-Match: W/"v1"
```

The origin responds,

```
  304 Not Modified
  Vary: Accept-Language
  ETag: W/"v1"
```

The cache has gained no information that a French representation has become
available, so it freshens its existing English entry and serves it to the
client. The client receives an unacceptable representation despite an acceptable
one being available from the origin.

### Proposed fix

Update RFC 9110 §15.4.5 to state that servers SHOULD include representation
metadata when a 304 response is on the basis of a weak validator match. This fix
is insufficient by itself, however, and must be combined with the one proposed
for issue #2.

## 2. Cache serves a representation labeled with incorrect metadata

This case begins the same way as above, but this time the origin includes
representation metadata in its 304 response:

```
  304 Not Modified
  Vary: Accept-Language
  ETag: W/"v1"
  Content-Language: fr
```

Note that RFC 9110 §15.4.5 states, "Since the goal of a 304 response is to
minimize information transfer when the recipient already has one or more cached
representations, a sender SHOULD NOT generate representation metadata other than
the above listed fields unless said metadata exists for the purpose of guiding
cache updates". Its inclusion in this case is defensible because it advertises
to the cache that a better representation than the one it currently has stored
might be available. Some existing implementations, notably Apache mod_cache, do
take advantage of this information and retry their validation request.

Notwithstanding the exceptions provided in RFC 9111 §3.2, the cache arguably
MUST update its stored header field, producing

```
  200 OK
  Vary: Accept-Language
  ETag: W/"v1"
  Content-Language: fr

  Hello world
```

and labeling an English body with metadata indicating French.

Although §3.2 states, 

> In some cases, caches (especially in user agents) store the results of
> processing the received response, rather than the response itself, and
> updating header fields that affect that processing can result in inconsistent
> behavior and security issues. Caches in this situation MAY omit these header
> fields from updating stored responses on an exceptional basis but SHOULD limit
> such omission to those fields necessary to assure integrity of the stored
> response

…this exception is not available, because the cache is merely storing the
response without performing any applicable processing. Even if the exception
does apply, it is only a MAY for a case that requires a MUST to prevent metadata
corruption.

### Proposed fix

Update RFC 9111 §4.3.3 and 4.3.4 to state: 

1. If a validation response contains Content-Type, Content-Encoding,
   Content-Language, or Content-Location headers, and the corresponding headers
   in a stored response are absent or different, the stored response MUST NOT be
   freshened or have any of its headers updated, unless the validation response
   contains a strong validator matching the stored response.

2. A cache MUST NOT serve a stored response body in conjunction with Content-Type,
   Content-Encoding, Content-Language, or Content-Location headers different
   from those contained in the stored response.

3. A cache unable on account of the above prohibitions to obtain a stored body
   suitable for responding to a client MUST retry its validation request without
   conditioning it on any weak validators.

## 3. Last-Modified headers incorrectly treated as strong validators

An origin's filesystem stores an identity-coded entity with a modification
timestamp of Thu, 1 Oct 2026 00:00:00 GMT. The origin server supports on-the-fly
gzip compression for clients whose `Accept-Encoding` header expresses a
preference for gzip.

A client requests,

```
  GET /
  Accept-Encoding: br;q=1.0, gzip;q=0.9, identity;q=0.8
  Range: bytes=0-1023
```

The cache forwards and the origin responds,

```
  206 Partial Content
  Vary: Accept-Encoding
  Content-Encoding: gzip
  Content-Range: bytes 0-1023/2048
  Last-Modified: Thu, 1 Oct 2026 00:00:00 GMT
  Date: Wed, 7 Oct 2026 00:00:00 GMT

  [gzip-encoded partial representation]
```

RFC 9110 §8.8.2.2 makes a `Last-Modified` time a strong validator if, "That
cache entry includes a `Date` value which is at least one second after the
`Last-Modified` value and the cache has reason to believe that they were
generated by the same clock or that there is enough difference between the
`Last-Modified` and `Date` values to make clock synchronization issues
unlikely". Since here there is six days worth of difference, the validator
qualifies as strong.

Later, the origin gains brotli support.

Another client requests,

```
  GET /
  Accept-Encoding: br;q=1.0, gzip;q=0.9, identity;q=0.8
```

The cache goes forward with,

```
  GET /
  Accept-Encoding: br;q=1.0, gzip;q=0.9, identity;q=0.8
  Range: bytes=1024-
  If-Range: Thu, 1 Oct 2026 00:00:00 GMT
```

the origin responds,

```
  206 Partial Content
  Vary: Accept-Encoding
  Content-Encoding: br
  Content-Range: bytes 1024-2047/2048
  Last-Modified: Thu, 1 Oct 2026 00:00:00 GMT
  Date: Wed, 14 Oct 2026 00:00:00 GMT

  [broti-encoded partial representation]
```

The cache splices the incompatible representations into a single malformed
entity.

### Proposed fix

Update RFC §8.8.2.2 to state that a `Last-Modified` time cannot be a strong
validator for any response which contains a non-empty `Vary` header or which
would not be storable in a shared cache. (The latter prevents this issue from
coming up for responses which implicitly vary on an `Authorization` header).

## 4. Altering `Vary` headers causes cache key confusion

A cache has stored,

```
  GET /

  200 OK
  Cache-Control: max-age=600

  Hello world
```

Later, the origin adds a login feature, so the representation of `/` now depends
on what cookie you present.

Later, a client requests

```
  GET /
  Cookie: user_id=bob; Expires=Thu, 22 Oct 2026 00:00:00 GMT
  If-Modified-Since: Thu, 1 Oct 2026 00:00:00 GMT
```

The cached response is stale, so the cache goes forward, and the server responds

```
  304 Not Modified
  Vary: Cookie
  Set-Cookie: user_id=bob; Expires=Thu, 29 Oct 2026 00:00:00 GMT
```

RFC 9111 §4.3.4 would now select the stored response for update, because it
could have chosen for the request, and neither the new nor stored response
contains a validator. Under the rules of the new `Vary` header, it would cease
to match, but it is unlikely that the cache can determine this because caches do
not typically bother to retain request headers that aren't nominated by `Vary`;
the cache would have no memory of what if any cookie was sent with the request
that originally produced the stored response. In the worst case, a cache
confused by this could produce an update,

```
  GET /

  200 OK
  Cache-Control: max-age=600
  Vary: Cookie
  Set-Cookie: user_id=bob; Expires=Thu, 29 Oct 2026 00:00:00 GMT
```

and then serve Bob's login cookie to the next non-logged-in visitor.

### Proposed fix

Update §4.3.4 to state that caches MUST NOT freshen stored responses or update
their headers on the basis of a 304 response whose `Vary` header(s) differ from
those in the stored response. Caches MAY create a new entry under a distinct
cache key that reflects the new `Vary` header(s), but MUST NOT associate any
stored body with the new entry unless a strong validator match is present. A
cache unable on account of the above prohibitions to obtain a stored body
suitable for responding to a client MUST retry its validation request without
conditioning it on weak validators.

## 5. Updates to `No-Vary-Search` policy can alter the scope of validators

A cache has stored,

```
  GET /?a=1

  200 OK
  ETag: "v1"
  No-Vary-Search: params=("a")

  Hello world
```

A client requests 

```
  GET /?a=2
```

and the cache goes forward with

```
  GET /?a=2
  If-None-Match: "v1"
```

The origin responds,

```
  304 Not Modified
  ETag: "v1"
  No-Vary-Search: params=()
```

The cache cannot safely use this response to satisfy the client, because the
"v1" ETag in the response is no longer scoped to the same resource as the one
that the cache intended to revalidate.

On the other hand, consider the following related hazard. An origin server
starts out ignoring all query parameters, and tracks the contents of `/` with
a version counter. So it might respond to

```
  GET /?a=1
```

with

```
  200 OK
  ETag: "v3"
  No-Vary-Search: except=("")

  Hello world, this is version 3
```

Later, it begins treating the `a` parameter as significant, so it assigns `a=1`
its own version counter, and begins responding,

```
  200 OK
  ETag: "v1"
  No-Vary-Search: except=("a")

  Hello a=1, this is version 1
```

but this could eventually count up to v3, in which case, for the same target
`/?a=1` URI and the same "v3" ETag, it would be serving distinct bytes. This
would violate the requirements of a strong ETag, since caches are not required
to implement No-Vary-Search and therefore may not recognize that these two ETags
represent distinct resources.

### Proposed fix

Address this problem from both ends. Caches MUST retry if a change in
No-Vary-Search policy produces a validation of a different resource (simplified
URI) than was intended. Origins which implement No-Vary-Search MUST ensure that
ETags are uniquely named throughout the base URI, not just throughout the
simplified or target URI.

## 6. Body reuse across non-matching stored responses needs clarification

This is <https://github.com/httpwg/http-core/issues/1127>.

A cache has stored,

```
  GET /
  Accept-Language: en, fr

  200 OK
  Vary: Accept-Language
  Content-Language: en
  ETag: "v1"

  Hello world
```

It receives a request,

```
  GET /
  Accept-Language: en, de
```

The stored response is a non-match under RFC 9111 §4.1, so the cache goes forward with

```
  GET /
  Accept-Language: en, de
  If-None-Match: "v1"
```

and the origin responds

```
  304 Not Modified
  Vary: Accept-Language
  ETag: "v1"
  Foo: bar
```

The cache cannot freshen its stored response, but it should nonetheless be safe for it
to create a new cache entry,

```
  GET /
  Accept-Language: en, de

  200 OK
  Vary: Accept-Language
  Content-Language: en
  ETag: "v1"
  Foo: bar

  Hello world
```

and to respond to the client with that. However, RFC 9111 §4.3.3 lacks clarity
on this.

### Proposed fix

Update §4.3.3 to clarify that caches may reuse, after validation, headers and
bodies from stored responses keyed non-matching Vary-nominated request headers
if and only if the validation response matches them on a strong validator. When
combined with the fix for issue #3, that strong validator would always be an
ETag.
