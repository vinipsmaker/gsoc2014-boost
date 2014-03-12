# Boost HTTP server

## Introduction

This document aggregate notes about performance considerations within the
proposed design in the main document.

### The `boost::http::server::backend` interface

<!-- TODO: update paragraph below according to decision that will exist related
to that <request,response> vs <socket-like> thing -->

An unified interface for _request_s pools and _response_s pools was chosen to
decrease the overhead of indirect function calls. Also, it's possible to compose
different pools into one, if the user wants, but the opposite choice would imply
overhead for the common case.

In fact, _request_ and _response_ are always used together and one could think
about integrate them within one object to decrease the memory overhead (only one
pointer stored instead two). But this decision is unwanted because they provide
a nice separation of responsibilities. An alternative approach would be to use
`pair<request,response>` instead separate objects and even has the interesting
advantage of a stronger semantic about them (they're used together and related
to each other, they're deleted together and alike). Now there would be a single
pointer to a pair instead two different pointers and the objects will be
allocated closer to each other, increasing data locality (good for cache) and
provide similar behaviour for an aggregated object, but keeping the clear and
separate responsibilities.

My idea to solve the problem of the other cases of even more constrained is to
expose a templated zero-overhead interface to the internal parser. Such
interface is outside of the scope of this proposal and is less useful for most
of the users, but it is possible to add it in the future.

### The `boost::http::headers` container

If the decision upon a hash-based container is made, some initial hints about a
good hash algorithm would be:

* http://murilo.wordpress.com/2013/10/16/deeper-look-at-phps-array-worst-case/
* https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function
* A crazy idea to analyze the [most used HTTP headers](
  https://stackoverflow.com/questions/114085/fast-string-hashing-algorithm-with-low-collision-rates-with-32-bit-integer)
  and possibly develop a new algorithm. Such idea is outside of the scope of
  this proposal, but this data could at least help benchmarking considered
  choices.

### `boost::http::server::request::uri()`

To correctly answer the HTTP request, the uri always must be parsed. It should
be reasonable, then, to directly store the parsing result instead the naked uri.
This change would avoid the need to parse the uri multiple times within the
chain of handlers. The drawback of this approach (always parse the uri) is the
impossibility to use other parsing methods (which isn't necessary anyway). To
overcome this drawback, one solution would be to embed both objects (raw uri and
`std::experimental::uri` into the request object), but this change doesn't offer
much value and still introduce new memory overhead to every request object (a
bad feature for embedded devices). A second solution would be to pass the
parsing result along the chain of handlers. This concern needs some benchmark
and a more carefully done analysis that only will be done after part of the
implementation is complete. Thus, it's unlikely that more thought will be put
here during this design/interface effort/phase.
