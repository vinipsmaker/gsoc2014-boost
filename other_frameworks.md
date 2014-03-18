# Other frameworks

You can find my analysis of other HTTP frameworks below.

## Node.js

A good inspiration for the design of this library is Node.js, a project that
allow programmers to run event loop driven JavaScript code on top of the
Google's V8 JavaScript engine to build scalable applications. The project
developed an API that might be interesting for async operations within HTTP, but
the most valuable contribution of inspiration was the success of the project,
which helped to demonstrate how high-level applications could be built on top of
this core set of abstractions.

Currently there are over 62k modules on the [npm registry](
https://www.npmjs.org/) providing abstractions to file-serving, request routing,
protocol implementations, communication schemes and many entities that I'm not
even aware of.

The most popular technique to glue the several modules together is through the
use of a middleware-style framework (such as [connectjs](
http://www.senchalabs.org/connect/) and [expressjs](http://expressjs.com/)). One
great advantage of this model is that becomes possible to provide "partial
solutions" for problems that cannot be entirely solved without domain knowledge
of the application.

One example of these middleware-based frameworks is the cookies parser + session
support, which will usually attach a session map/dict-like object to the request
object that can be used to access data related to that specific client. One fact
hat this example exposes (and the reason why I cited it) is how the nodejs
developers explore the dynamic nature of JavaScript to ease the integration of
different modules within the middleware-style framework. They tend to inject new
attributes in the request objects, but I'm not aware of how to do this
_transparently_ in C++. The one way to mirror this approach in C++ that I'm
aware of is to attach a `map<string, any>` attribute to each request object, but
this approach is _unclean_ and introduces unnecessary overhead to all request
objects.

One solution that I think that would fit well in C++ when combined with the
middleware-like approach in node.js is to make clear to the user to **NOT** use
the request object as a key in some map to exchange data. The recomendation
should be to prepare interfaces for this model (eg. functions that return the
wanted data based on the request object) and use in-memory local session stores
where the user really needs to share data.

Said all that, Node.js itself is not very useful when it comes to API
inspiration, because several of its abstractions were created to deal with the
lack of JavaScript support for certain operations, like the lack of buffer
management. The main contribution for inspiration was its popularity. Its
popularity was responsible for the development/research of some programming
model that could be used to "glue" the different developed abstractions together
while the scalable async property would be kept.

In fact, their popularity is so large, that several projects were born to port
their API to different languages such as Lua, C++ itself and others.

## PHP

I tried to look at how PHP developers solved their problems, but they seem to
have very different priorities/concerns and it wasn't helpful for this proposal
at all.

## Django

Django doesn't has a great concern on async operations and scalability, but
their developers had developed solutions for common problems and they spent time
to improve security. The resulting documentation of the project become a nice
place to look for what are the next problems to solve once we have a core HTTP
server library in Boost and the documentation also taught me some nice things
about security.

## CppCMS

CppCMS has a great list of ideas to improve the performance of the HTTP server,
such as:

* Recycle objects, connections, data and pages.
* Taking caching to the limits (cache invalidation, two levels cache, caching
  the gzip compressed result).
* Monkey's HTTP Daemon design of several requests per event loop and one event
  loop per thread.

These same ideas should be kept in mind while designing the library.

## cpp-netlib

The project has the nice feature of customize the handler's type based on a
template argument. This can be useful to use `std::function` handlers combined
with plugins to allow the user to change the code at runtime keeping everything
as little decoupled as possible while maintaing a bit of style, but also allow
the user of embedded projects to use a handler with less overhead than
`std::function`.

It's just the nice C++ style...:

> You only pay for what you use

However, the task to implement a protocol like HTTP doesn't really introduce the
need for generality. It'd introduce undesired complexity to the definition of
new backends (HTTP messages producers). In fact, cpp-netlib is so overly
generalized that request and response types, that require no generalization, are
nested within template classes, introducing even more unnecessary difficult to
create general handlers. Now, not only the handler registering is "generalized",
but also the arguments of the handling function, whose types comply with
concepts that benefits from no generalization at all. The `request` class is
nested within the `server` template class, but [the server template takes in a
single template parameter](
http://cpp-netlib.org/0.11.0/reference/http_server.html) that cannot specialize
**ANY** of the generic request concepts, such as [R::string_type or
R::headers_container_type](
http://cpp-netlib.org/0.11.0/reference/http_request.html#pod-server-request-concept).
To specialize such class, you'd need (almost) reimplement your own
`netlib::network::http::server`. It doesn't look too benefical to the user.

What I propose is to, _in the future_ (eg. outside of the scope of this
proposal), provide a templated zero-overhead interface to the internal HTTP
parser. This option fits well in the design of embedded projects, but doesn't
propagate the disadvantages of a coupled HTTP parser and message producer to the
rest of the library. A second alternative, that may work as good as the parser
interface and it **is** within the scope of this proposal, is the use of the
proposed `boost::http::basic_socket` object (more details somewhere below,
within the section after the analysis of other frameworks).

I'll use templates only in a very few places that don't affect the code of
backends or HTTP handlers. A few places where I think templates would have a
better use within this library would be in backend implementations (not
interfaces) to control/tune some behaviour through policy tags. And backends
aren't going to be so intrusive in this proposal, then changes to backend
implementations don't require non-related parts to be "prepared".

One problem with cpp-netlib is the fact that they choose a design that made
difficult to support some features like HTTP pipelining and support for other
backends like FastCGI. The design presented here doesn't suffer this problem.

Also, it seems that the lack of unification (for instance, there is one
"synchronous" server and one "asynchronous" server that are _mostly_ providing
the same responsibilities) among the abstractions has been caused by a lower
priority to include support for modern HTTP features and asynchronous operations
in the earlier design. Even though HTTP is not that _modern_, but pretty stable
actually.

Most of the templated nature of cpp-netlib comes around the `request` concept,
which doesn't really need generality. Within cpp-netlib, this concept is defined
for synchronous servers and asynchronous servers. In the model proposed here, I
intend to support primarily the asynchornous model and provide support for
synchronous-like or purely synchronous model through the extensible ASIO model,
but I need to research the topic a little deeper before giving any such
guarantees (synchronous-suppored model).

The remaining of the introduced generality can be provided through static
polymorphism using non-member functions. This generality can be added later and
is outside of the scope of this proposal. These functions could be useful to
abstract differences between a server-incoming http request and a
client-outgoing http request. It's still a good idea to keep them separate
objects, because they serve different purposes (asynchronous reading from a
multitude of backends and asynchronous writing).

One generality within the `request` object lies in some of its attributes, like
`R::string_type` (used in `r.method` within [cpp-netlib](
http://cpp-netlib.org/0.11.0/reference/http_request.html#pod-server-request-concept))
and `R::headers_container_type` (used in `r.headers` wihin [cpp-netlib](
http://cpp-netlib.org/0.11.0/reference/http_request.html#pod-server-request-concept)),
but they are just containers for usual network resources found in HTTP messages
and should reflect this nature (such as using a string of 8-bit char and a
possibly a new `headers` container). Also, this reminds that iterators were
created to avoid N * M implementations, a situation that would pretty much
happen here if you don't go all the way with templates and thus would create an
overly unnecessary complex template-based design that would pretty much kill the
possibility to have plugin-based handlers implemented by different teams while
different backends (builtin server, FastCGI, ...) are used.

## [pion](https://github.com/cloudmeter/pion)

The project shares some design considerations with cpp-netlib and they won't be
repeated here.

The iterator-based interface in some places is quite nice and can help to avoid
a new allocated container. The query string within urls is an example, but
`pion::http::request::get_queries` returns an `ihash_multimap`, not a custom
wrapper that would play a role similar to the `string_view` proposal.

The complete separation of response pieces (eg. set status code and reason
phrase within two different expressions/function-calls) looks good, but I don't
want to make the API more difficult than already is (for the user writing
handlers, the user writing backends and me writing even more complex
documentation). This API works in pion, but it isn't very asynchronous.

One "higher-level" abstraction of pion to register handlers (the request
router/dispatcher) is the use of a tree-like based resource dispatching. Under
this model, you can have several handlers and each one will handle a resource
under one path. For instance, you could have a handler for "/" and a handler for
"/index.html". If there is no registered handler for requested resource, the
handler for the parent resource is used.

Now, before I proceed, documentation isn't very complete and info can be partly
innacurate, but I'll present it anyway because it exposes an interesting design
characteristic to keep in mind. And one more thing is that even if the comments
aren't very accurate, they aren't entirely untrue, because I tested them.

One problem with the tree-based approach is that you cannot compose a chain of
handlers to help the server. What if you change the url/resource attribute of
the request to allow the same handler be used for "/" and "/index.html"? Well,
this is considered a special for pion (as I understand) and the way to go is to
use `pion::http::server::add_redirect` member function. But then the control is
too "static" (in the sense of opposite to dynamic) and handlers cannot have much
control over the dispatching behaviour. Suppose you want to add a handler to
serve from plugins and, if a plugin for the requested resource cannot be found,
proceed to the next handler, the static file handler. Under this example, the
tree style wouldn't fail completely, because you could still chain the handlers
by adding specific knowledge about the remaining handler within the plugin-based
handler (not orthogonal, not clear separation of responsibilities, difficult to
change and the list goes on). The tree style approach would fail if the
plugin-based is outside of your control and cannot be changed (now you'll add a
new level of workarounds to emulate the middleware style). You just won't get
much cooperation among handlers using the tree style of resources approach.

Using the middleware style the possibilities are quite interesting. You could,
for instance, register handlers to a vhost under the "/john_lennon" path and use
another handler to (1) check if the vhost is being used and (2) conditionally
prepend the "/john_lennon" prefix internally (change visible only to the server
and not to the remote user, eg. avoid extra network traffic). Of course there
are times where explicit redirects are wanted, but they are supported under the
middleware style model too.

Well, previously I mentioned that the tree-based was a "problem", but, in fact,
it isn't. The consequences of the tree-based approach aren't problems, but a
consequence of the adopted "style". This is what the tree-based approach is...
a style. This style has the nice advantage of clearly separating different
handlers, a very useful feature to avoid _spaghetti code_. Of course it's
possible to achieve separation under the oher design (middleware style) by
allowing _nested_ routers/dispatchers. The router itself is a handler, then it's
natural to nest routers together. Also, it's possible to implement the tree
style on top of the middleware style, but the opposite is a tough task. Even
advocating the _middleware_ style, I will propose to **not** deliver these
routers in the core library submitted to boost (yet!), but to use a raw handling
mechanism which both styles (and more to come) can use. A modular design,
remember?

## POCO

I took a look at some gritty details on POCO http server and the design looks
mostly okay, but it doesn't have a strong focus on asynchronous operations like
ASIO. I want to clarify that I'm **not** comparing POCO features to ASIO
features. I'm comparing design principles upon the two were built.

In fact, the POCO interface doesn't resemble ASIO at all (eg. by making use of
things like std::istream for Poco::Net::HTTPRequest::read). It's not weak focus
on async operations. It's almost no consideration to async at all.

Another concern would be support for other backends, but looking at the
documentation, it's possible to see a highly separate hierarchy of abstractions
that plays well with multi-threaded design, [modern HTTP features](
https://github.com/pocoproject/poco/blob/492317224154a21407ba346f6e81471561e45250/Net/src/WebSocket.cpp#L142-162)
and would allow multiple backends (but I didn't find any other ready backends,
then **maybe** it means a not easy-to-implement interface was defined).

The only real problem was the lack of a highly asynchronous interface, but their
work would still be useful to help define abstractions (and their hierarchy). Of
course an ASIO-ification should happen in the process.

[This is a nice page to see POCO's look-and-feel](
http://pocoproject.org/docs/00100-GuidedTour.html#5). Due to the too intrusive
design ([need to implement specificy interfaces, even if only one point of
flexibility is required](http://isocpp.org/blog/2014/03/callback) such as
`TimeRequestHandler::handleRequest`), the code became more verbose than
necessary. Compare the code with this [other one](
https://github.com/d5/node.native#sample-code), also written in C++.

Said all that, Poco's focus seems to be any networking application and is pretty
large. There's even database, xml and zip abstractions.

## Others

If you have specific concerns about any library make an objective and
directioned question and you'll have a more developed answer.

Some examples of good question would be:

> Have you looked at the **Django's request router**?

&nbsp;

> Have you looked at the **organization of servers, requests and handlers in**
> **Twisted**?

Do not ask about how the design of complete projects compare the design of this
whole proposal, because I might not have the time to review the other's whole
project. There's the risk that I'd skip the part of the other project that
you're really interested in too. Please keep the questions directioned.

It would be even better if you objectively mention names, then my research
would be faster and you'd have an answer faster.

And try to check if your suggestion fits well in the proposal for an async
library to save me the time to answer the several questions that I hope to
receive. Anyway, I'll certainly respect any questions and try to give an
insightful answer and possibly modify this proposal given the suggested
improvement.
