# HTTP server on boost

## Goals

* Develop a library for Boost submission and inclusion that abstracts HTTP.
* Asynchronous design that provides scalability.
* Modular design that doesn't force all abstractions to be used together.
  * The main goal of the modular design is to fit well in embeded projects
    developed for resource-constrained devices while it still provide the
    possibility and some abstractions for better performance on devices not so
    constrained, such as pools to recycle objects and threaded versions that can
    (mostly) cooperate with the same interfaces.
* Expose the HTTP power.
  * Chunked entities. A stream interface that allow live-stream.
  * 100-continue status. A feature to reduce network consumption.
  * Upgrade support. A feature required for WebSocket.
* Allow multiple backends such as FastCGI and CoAP. It will be useful also for
  HTTP 2.0.

## Non-goals

* Provide lower abstractions to the HTTP parser.
  * This project will use ready HTTP parsers, then I'll be free to focus on
    other important features.
  * It can be replaced later, because the parser interface will not be exposed
    and it won't affect code that makes use of this library.
* Provide even higher-abstractions to write template-driven MVC web applications
  quickly.
  * But it'll be possible to build such abstractions on top of the developed
    library.
  * In fact, there are a lot of higher-level abstractions competing with each
    other, providing mostly incompatible abstractions. By not targeting this
    field, this library can actually become a new base for all these
    higher-level abstractions and allow greater interoperability among them.

## Some target scenarios

I've wrote this section several times, but Bjorn Reese put into better words
than me, then I'll quote it.

> 1. Targets embeddable HTTP servers (e.g. to implement ReST APIs).
>    These servers should be able to co-exist with other embeddable
>    communication facilities (e.g. Bonjour device discovery or
>    BitTorrent.)
> 2. Create a C++ toolbox for embeddable HTTP servers.
>    This requires a modular design and a coherent thread model.
> 3. Flexible design that can encompass HTTP chunking, HTTP pipelining,
>    WebSockets, FastCGI, etc.
> 4. Scalable performance.
>    This requires an asynchronous design.
> 5. Reuse existing std or Boost components, such as Boost.Asio.

-- Bjorn Reese, [on the boost mailing list](
http://lists.boost.org/Archives/boost/2014/03/212072.php)

## Design considerations

The idea in this initial proposal is to implement a core library which is useful
and can work on projects which require low-overhead implementation. What I mean
by low-overhead is that the library won't force you to use abstractions that
your project doesn't need or cannot afford. This is achieved through a modular
design. You don't need a request router if the only purpose of your project is
to stream some video being captured at realtime, for instance.

Another design decision of the design is to go full async, but also allow
threads, then you can mirror the successful design of the [Monkey HTTP Daemon](
http://monkey-project.com/), where you have multiple requests per event loop and
one event loop per thread. Again, the design is not all-or-nothing and if you
don't want to, this library won't force you to use threads.

Any proposal submitted to be part of standard libraries (or general ones like
boost) should, obviously, be general and usable by projects with different
requirements. One example of the difficulties in such challenge, within this
proposal, is a collection of cache and pool objects, which can improve the
performance of big-scale applications, but might increase the memory-overhead in
embedded applications to the point where it becomes a non-solution.

The design should allow the programmer to pick the interesting pieces and glue
them together to fit in his project.

The non-intrusive and modular design choices don't come for free. Now, everytime
you're going to write a web application for your company, you'll probably be
writing the same skel code to setup the http server, the session store objects,
the request handler dispatching/router and the glue among them. Fortunately, (1)
the boilerplate is very small, (2) it's possible to go from non-intrusive design
to the intrusive design (the opposite is not that easy, though) and (3) it's
possible to provide minimal project examples that could be used as templates for
the user. One example by what I mean with #2 is to provide an even higher
abstraction that will assume that the user wants to use a common set of
components and provide a less verbose interface, but this higher-level
abstraction requires a core library first and is outside of the scope of this
proposal.

The only requirement/intrusiveness of the library, ideally, should be an async
programming model, but this only affect programming tastes and doesn't imply
that the library cannot be used in certain devices or clusters.

The only feature that should include an overhead in the proposal is the
decoupling of HTTP messages and HTTP connections (the HTTP parser). HTTP
messages should be feeded by producer backends that could implement different
communications such as a builtin HTTP server, a FastCGI receiver and others.
The cited overhead is only one level of indirection and should be very small. To
provide the same functionality, one pure C API would need to add this small
overhead too. Also, this design brings large benefits that easily overcome the
ultra low overhead added concern, such as:

* Obviously, HTTP messages are not tied to HTTP connections anymore.
  * Now it's easier to apply a finer-grained threaded request dispatch where you
    can detect very busy threads by the amount of messages and not by the amount
    of connections. Of course I want something even clever while things are
    still kept simple, but let's discuss this later.
* CGI/FastCGI, HTTP/2.0 and limitless others without breaking the logic of the
  handlers created by the application programmer.
* HTTP pipelining is easier to implement (for me and for the user of the
  library).
* There are other nice implications, but I want to detail this later.

### Node.js

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

### PHP

I tried to look at how PHP developers solved their problems, but they seem to
have very different priorities/concerns and it wasn't helpful for this proposal
at all.

### Django

Django doesn't has a great concern on async operations and scalability, but
their developers had developed solutions for common problems and they spent time
to improve security. The resulting documentation of the project become a nice
place to look for what are the next problems to solve once we have a core HTTP
server library in Boost and the documentation also taught me some nice things
about security.

### CppCMS

CppCMS has a great list of ideas to improve the performance of the HTTP server,
such as:

* Recycle objects, connections, data and pages.
* Taking caching to the limits (cache invalidation, two levels cache, caching
  the gzip compressed result).
* Monkey's HTTP Daemon design of several requests per event loop and one event
  loop per thread.

These same ideas should be kept in mind while designing the library.

### cpp-netlib

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

### [pion](https://github.com/cloudmeter/pion)

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

<!-- TODO: The below paragraph -- understand why/how schedulers are used within
    pion -->

Another thing worth mentioning is the scheduler. Again, the project isn't very
documented, turning the idea of custom schedulers almost useless. But the
concept of a scheduler is well known and I'll take a closer look on how this
concept is used within pion later.

### POCO

I took a look at some gritty details on POCO http server and the design looks
mostly okay, but it doesn't have a strong focus on asynchronous operations like
ASIO. The interface itself doesn't resemble ASIO at all (eg. by making use of
things like std::istream for Poco::Net::HTTPRequest::read).

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

### Others

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

### SG4

SG4 is the name of the [responsible working group for networking within the
next C++ standard](http://isocpp.org/std/the-committee). Some of their proposals
are related to abstractions that ASIO already provides, such as representation
for IP addresses. There are also proposals that aren't directly interesting for
this proposal, but there is also one interesting proposal that may
affect/inspire the design documented here. This interesting proposal is the
N3625, which documents an URI library for C++.

There is a [standalone version of the URI library a github](
https://github.com/cpp-netlib/uri). Details about how this document/library
impacts this proposal will be found below, where appropriate. Currently there is
no need to make use of this proposal or to mirror it in boost, but this feeling
might change while I finish this design document.

<!-- if proposal becomes too greedy and requires a ready uri implementation:

The general plan is to make use of this library, while it's kept updated and
possibly mirror its interface in boost, while it isn't merged with the C++
standard.

-->

### std::future

std::future approach will make the life of developers easier once the `await`
keyword enter on the standard and was considered in this proposal. The
popularity of node.js showed that nesting can get insane pretty fast. Just
compare the following two artificial examples:

* https://gist.github.com/vinipsmaker/8741321#file-6-cpp
* https://gist.github.com/vinipsmaker/8741336#file-7-cpp

Now you should remember that [this problem happens in real-world](
https://stackoverflow.com/questions/4234619/how-to-avoid-long-nesting-of-asynchronous-functions-in-node-js).

Boost ASIO provide first-class support for `std::future` through
`boost::asio::use_future`, but I don't want to force this model on the user.
There are other approaches to make the code more readable under the callback
model (see n3964 paper for more details).

## Leveraging Boost

The library would be built on top of Boost ASIO to provide an async API. Boost
AFIO would be used in places where file I/O is used, such as the static file
server. Someone on the Boost mailing list pointed me that the AFIO's batch
feature could be used to help with HTTP pipelining, but the separation of
request objects and request connections already solves the interface side of the
problem and AFIO would be, at most, an implementation detail not seen by the
user. After further inspection, I decided to not use AFIO's batch features in
the implementation, but this is, once again, just an implementation detail and I
can change my mind and the code without affecting people. I don't want to give
details about the decision, because it's an implementation detail and I want to
see the review focused on the interface, the main contribution of this proposal.

## Leveraging other projects

The initial implementation will use Ryan Dahl's HTTP parser, which is a parser
used by large projects such as node.js itself, doesn't buffer the data, can be
interrupted at anytime and has a permissive license. Some reasons to replace the
parser later are:

* Ryan Dahl's parser was designed with a C interface in mind (something like C's
  `qsort` vs C++'s `sort`). This isn't really a big problem here.
* Ryan Dahl's parser doesn't support arbitrary HTTP method names. This is a big
  concern on long-term and is a motivator to change the parser later.
* A parser interface exposed to the user will benefit some projects for embedded
  devices, but the C-based Ryan Dahl's HTTP parser interface can be way better
  exploring C++ and Boost features.

The reason to use a ready parser for now is to focus on more important aspects
of the project in the beginning like API, scalability and flexibility.

## The proposal

This proposal defines not only an initial interface, but also an usable (in the
sense of applicability) core set of abstractions. Said that, interface is very
subject to change. The implementation would very well give new insights about
performance and its availability would be a playground for applications to
stress test and improve the flexibility.

Usual tasks would be to test correctness and exception-safety of the
implementation, define and document guarantees, so an useful application can be
created without undefined behaviour and continue to work on future versions.
All tests should be automated to retain usefulness.

Small applications should be created to help benchmark and spot bottleneck
points. Given that, at least a file server will be created for the proposal. You
can find details about the proposal below.

The proposal might be a little incomplete, but I hope it won't be a problem to
understand the main ideas and I'll be available for any questions you might
have. And besides being incomplete, it's a solid start for the project
(including implementation work).

### namespace

All abstractions presented here should live, unless explicitly stated otherwise,
under the `boost::http::server` namespace.

### The usual life of a HTTP request

This section is here to give you an important overview of one of the simplest
handling models possible with the proposed library. This is done to avoid the
[blind men and an elephant](
https://en.wikipedia.org/wiki/Blind_men_and_an_elephant) syndrome.

#### The built in HTTP server way

1. Programmer's code instantiated a `boost::http::basic_message<unspecified>`
   object and instructed `boost::http::basic_socket_acceptor` to fill its object
   and call a callback when a new message is ready.
2. You open Firefox and make a request.
3. The `boost::http::basic_socket_acceptor` reacts and call your callback upon
   some unspecified asio's event.
4. You use `boost::http::basic_socket` API to respond to the request.
5. The `boost::http::basic_socket` object will rely on the
   `boost::http::basic_socket_acceptor` object to rely on ordering guarantees
   (HTTP pipelining) and the reply will be eventually sent.

Notes:

* `boost::http::basic_message<unspecified>` is ready for the user callback when
  enough data (all headers) was gathered.
* The programmer's code can then inspect input data through the `basic_message`
  object and the several parsers that receive it as input (including session
  support) and reply with the `write_start_line()`, `write(boost::asio::buffer)`
  and `end()` methods of the `message` object.

### The `boost::http::basic_socket` abstraction

This abstraction is being proposed to improve the message passing fundamental
design, abstracting differences between request and response messages away. The
only fundamental difference between request and response messages is the _start
line_. The `basic_socket` object should store the start line (excluding
`"\r\n"`) without trying to decode its meaning, leaving this task for a
specialized abstraction of the appropriate HTTP message type (request or
response).

This abstraction should resemble `boost::asio::basic_socket`. Below you'll only
find the declarations for the sync version of the API. This is only done for
simplicity and the async versions are the main focus of this proposal. One thing
missing from this API is server-push behaviour (for HTTP/2.0) and it'll be added
later.

Three levels (`basic_message`, `basic_socket` and `basic_socket_acceptor`) are
used instead two to better support HTTP pipelining (and even multiplexing on
HTTP/2.0). `basic_message` represents a single HTTP message, while
`basic_socket` represents a socket stream that can be used to send multiple HTTP
messages.

Because HTTP imposes a clear separation of client and server, a "peer"
architecture where everyone "is" client and server like WebSocket or JSON-RPC
cannot be adopted. Thus, `basic_socket_acceptor` will "emit" `basic_message`
objects that cannot be used to send new requests over the channel.
`basic_socket_acceptor` should handle pipelining of incoming data and
`basic_socket` should handle ordering/pipelining of outgoing data.

The design might be refined later. You should also notice that `Protocol`
concept isn't defined, because the precise communication (in the sense of
objects communication and not network communication) isn't completely defined,
but this is this less relevant for a proposal submission and it'll be done
during the implementation.

```cpp
namespace boost {
namespace http {
/** Represents a single HTTP message (a request or a response).
 */
template<class Protocol>
class basic_message
{
public:
    string_type start_line;
    headers_type headers;

    void write_start_line(string_type start_line);
    void write(boost::asio::buffer &buffer);

    // body might very well not be fit into memory and user might very well want
    // to save it to disk or immediately stream to somewhere else (proxy?)
    boost::asio::buffer receive_body_part();

    // used to know if message is complete and what parts (body, trailers) were
    // completed.
    int incoming_state();
    int outgoing_state();

    void end();

    // it doesn't make sense to expose an interface that only feed one trailer
    // at a time, because headers and trailers are metadata about the body and
    // the user need the metadata to correctly decode/interpret the body anyway.
    headers_type receive_trailers();

    // this function already requires "polymorphic handling to take HTTP/1.0
    // buffering into account
    void send_body(boost::asio::buffer &buffer);
};

template<class Protocol>
vector<char> receive_body(basic_message<Protocol> &message, headers_type &trailers);

template<class Protocol>
class basic_socket
{
public:
    typedef basic_message<Protocol> message_type;

    void send_message(message_type &message);
};

template<class Protocol>
class basic_socket_acceptor
{
    void accept(basic_socket<Protocol> &socket);
};
}
}
```

This design was inspired by [these](
https://sourceforge.net/p/axiomq/code/ci/master/tree/include/axiomq/basic_message_socket.hpp)
[classes](
https://sourceforge.net/p/axiomq/code/ci/master/tree/include/axiomq/basic_message_acceptor.hpp)
from the axiomq project and by an interesting discussion with Bjorn Reese.

In case you haven't noticed, this abstraction can also be used to create a HTTP
client. Of course this HTTP client is little primitive, because the focus of
this proposal is to create a embeddable HTTP server.

These abstractions would be used for the built in HTTP server and I'm also
thinking about the possibility of explore the template-based nature of this
abstraction to also reuse (as opposed to use it just for the sake of using) it
somewhere else. Maybe it'll resemble the polymorphic allocators, but maybe
you're having difficult to imagine what I want with this lack of information,
but the interface will improve with the time. The thing to pay attention for now
is that this design is nice, but it's incompleting regarding the proposal scope
(for instance, how do we integrate FastCGI in this design).

### The `boost::http::headers` container

This is one of the only two names currently defined outside of the namespace
`boost::http::server` and it would also be used in a future HTTP client library.

Initially it would be a typedef for an `unordered_multimap<string, string>`, but
I intend to discuss this concern further and possibly run a series of
benchmarks. A custom `KeyEqual` functor will take care of case insensitive keys
while discussions about the right container are ongoing.

**NOTE:** The SG4's uri library also face the problem to define a
case-insensitive interface and the chosen solution was to convert them to lower
case upon normalization.

### The `boost::http::status_code` enum

Just an enum class containing the useful status codes defined in RFC2616. A
client library will receive a status code through the response from the remote
server, then this declaration is done outside of the server namespace, because
it's useful for servers and clients.

The client library (outside of the scope of this proposal) can receive integers
not enumarated in this abstraction, then an integer would be chosen instead of
this enum in such client library, but this abstraction is still useful for
comparassions maintaining readable code. Consider the following example:

```cpp
if ( response.status_code() /* returns an integer */ == status_code::OK ) {
    // ...
}
```

Of course I'm aiming C++11 at minimum and enum classes are useful to avoid
namespace polution. Then, to the code above work, the following two declarations
would be required:

```cpp
bool operator==(status lhs, int rhs);
bool operator==(int lhs, status rhs);
```

### Where the proposal needs to be improved?

Surely, the first thing is completeness os several assorted missing parts.

I also want to better study how a performant multi-threaded model can be
integrated in this library.

The last task for me is to reread the n3964 paper (Library Foundations for
Asynchronous Operations) more carefully to propose an extensible model that can
adapt itself to support futures, coroutines and Boost.Fiber, to name a few. I'm
not sure if it's possible to support them all within the GSoC timeframe and I
need to study a bit further to check that. If ASIO can help me enough, then I
think it'll be possible to implement them all, but if not, I want at least to
make sure that more models can be supported in the future without API breakage.

## Extra features

The implementation of a HTTP server isn't very challenging. The biggest problem
is to get the design right, but I already gave my design ideas in this document,
then we can consider the design (the most challenging part) done. Of course I'm
open to suggestions and I won't mind to refactor the interface in the middle of
the implementation work, but to compensate the big difference of effort between
interface and implementation, some of the following features would be
implemented too (I'll decide which ones are optionals and which ones are
required once I write the timeline for the project):

* Session support (inspired on Tufão's session support, which abstracts
  client-side and server-side based session stores under the same interface, but
  with improved usability thanks to Boost and C++11).
* HTTP/2.0 experimental support (I'm not aware of the protocol details, then I'm
  not so sure about the difficult, but people implementing it are stating that
  is easier than HTTP/1.1).
* WebSocket support.
* FastCGI support.
* A full featured (conditional requests and partial download) file server
* Helpers for threads.
* A robust, flexible and async request router with wrappers to make easier to
  write handlers that don't need the async feature/don't block. Although the
  "increased" difficult for async handlers is quite low... but it's still
  boilerplate.
* Forms and file uploader parsers.
* CoAP support.
* Test/research the best hash algorithm for HTTP headers based on the most
  commonly used ones. Okay, this list is getting kind of crazy, then you should
  know that I'll accept any suggestions or feedback constructively.
* Compression support.
* Proxy? Okay, I'm getting out of ideas, it's your turn to suggest some
  improvement.

Keep in mind that the earlier proposed core library is pretty solid and the
previous cited "extra features" can be implemented without breakage of API or
ABI. Also, some of the earlier features are protocols that I already implemented
before and I wouldn't get lost, need help or take too long to implement.

Also, each one of these "optional features" should have its own document in a
separate place, to better allow the discussion and review of different features.

I'd prefer to implement HTTP/2.0 sooner, then the new features that require
extending (read _extending_ as extending, not _changing_) the core API to
provide server-sent responses to non-yet-received-requests would be propagated
sooner.

Good candidates to be implemented after HTTP/2.0 are the threads helpers and the
router with some sample handlers to prove the flexibility and power of the API.
This proof can help to convince boost members to agree that this is a good
design and should be integrated within the rest of the project.

Another thing to keep in mind is that I prefer to implement features on the
order that will benefit my users the most. On my last year's participation in
GSoC, I've had discussions with the users and developers opposed to the "do
first, ask later" approach, I've gathered and implemented features that users
wanted and even collected opinions/votes on diverging features. I'm used to
adopt this approach every time I have contact with the user.
