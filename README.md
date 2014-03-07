# HTTP server on boost

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
to improve secutity. The resulting documentation of the project become a nice
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

### Twisted

Some members of the boost mailing list suggested me to take a look on Twisted
and _do what they do_, but there wasn't many design ideas that would fit well
for this library. If you have specific concerns, not only about Twisted, but any
library, make an objective and directioned question and you'll have a more
developed answer.

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
new backends (HTTP messages producers). What I propose is to, in the future,
provide a templated zero-overhead interface to the internal HTTP parser. This
option fits well in the design of embedded projects, but doesn't propagate the
disadvantages of a coupled HTTP parser and message producer to the rest of the
library. I'll use templates only in a very few places that don't affect the code
of backends or HTTP handlers.

One problem with cpp-netlib is the fact that they choose a design that made
difficult to support some features like HTTP pipelining and support for other
backends like FastCGI. The design presented here doesn't suffer this problem.

Also, it seems that the lack of unification among the abstractions has been
caused by a lower priority to include support for modern HTTP features and
asynchronous operations in the earlier design. Even though HTTP is not that
_modern_, but pretty stable actually.

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

The reason to use a ready parser for now is to focus on more important aspects
of the project in the beginning like API, scalability and flexibility.

## The proposal

### namespace

All abstractions presented here should live, unless explicitly stated otherwise,
under the `boost::http::server` namespace.

### The usual life of a HTTP request

This section is here to give you an important overview of one of the simplest
handling models possible with the proposed library. This is done to avoid the
[blind men and an elephant](
https://en.wikipedia.org/wiki/Blind_men_and_an_elephant) syndrome.

#### The built in HTTP server way

1. You open Firefox and make a request.
2. The `boost::http::server` object reacts upon some unspecified asio's event.
3. The `boost::http::server` object acquire a new or reused
   `boost::http::server::request` object.
4. The `boost::http::server::server` object creates or reuse an internal HTTP
   parser object and tie it with the recently acquired
   `boost::http::server::request` object.
5. Once the `boost::http::server::request` object is ready -- it happens when
   enough data (all headers) was parsed -- the `boost::http::server::request`
   object and its response object are announced to the programmer.
6. The programmer's code can then inspect input data through the `request`
   object and the several parsers that receive it as input (including session
   support) and reply with the `writeHead(int,string)`, `write(vector<uint8_t>)`
   and `end()` methods of the `response` object.
7. All write/output operations from the `response` object will delegate the real
   work to the appropriate backend, that was chosen during object construction.
   Similarly, the backend will feed all data received through the `request`
   object.

   The `request` and `response` objects also provide some very few high-level
   information about the backend, such as "native-stream" property if HTTP
   chunking or similar is provided or "upgrade-possible" if it's possible to
   upgrade to another protocol (such as WebSocket). The purpose of such exported
   info is to avoid hanging the application in some backends (such as trying to
   live-stream video to a HTTP/1.0 connection).

#### The FastCGI way

A FastCGI situation would be similar to the previous one, but the
`boost::http::server::server` would be replaced by
`boost::http::server::fastcgi`, whose construction would be different. The rest
of the code (the handlers) would remain untouched and compatible.

### The `boost::http::server::backend` interface

An unified interface for _request_s pools and _response_s pools was chosen to
decrease the overhead of indirect function calls. Also, it's possible to compose
different pools into one, if the user wants, but the opposite choice would imply
overhead for the common case.

I'm thinking about separating pool related functions in a second interface,
inheriting from this one.

I'll also define the pool interface, that should just be a convenient class from
where you can get objects or return unused ones.

`handler_type` is a runtime bound functor for several reasons. One of the
reasons is to give freedom for the user to use any functor-like object as
argument. This requirement could also be achieved through templates, but backend
is meant to be an interface for HTTP message producers and function templates
cannot be virtual. Also, the backend would need type erasure to handle all the
different handlers meeting the required signature anyway.

My proposal to solve the problem of the other cases is to expose a templated
zero-overhead interface to the internal parser. Such interface is outside of the
scope of this proposal and is less useful for most of the users, but it is
possible to add it in the future.

The proposal for executors and schedulers (n3731) face a similar problem and
they've chosen the same solution, `std::function`.

```
namespace boost {
namespace http {
namespace server {
class backend {
public:
    typedef std::function<void(request, response)> handler_type;
    /* unspecified: pool_type */

    virtual ~backend() {}

    virtual void async_handle(handler_type handler) = 0;

    /**
     * The user is responsible to maintain the lifetime ot the \p pool object
     * while the request wasn't finished. I thought about use a reference
     * instead pointer, but I wanted to make explicit the fact that the pool
     * may be used other times. I'm not satisfied with this design yet and I
     * plan to improve it.
     */
    virtual void async_handle(handler_type handler, pool_type *pool) = 0;
};
}
}
}
```

> TODO

### The `boost::http::server::request` class

> TODO

The object **MUST** not be destroyed, cleaned, reseted or recycled by the
backend or any other entity after the `end` event is reached. This action is
only allowed **after** the `end` event of the associated `response` object. All
internal backends give this guarantee and failing to comply with it triggers
undefined behaviour.

### The `boost::http::server::response` class

> TODO

### The `boost::http::headers` container

This is the only name currently defined outside of the namespace
`boost::http::server` and it would also be used in a future HTTP client library.

Initially it would be a typedef for an `unordered_multimap<string, string>`, but
I intend to discuss this concern further and possibly run a series of
benchmarks. A custom `KeyEqual` functor will take care of case insensitive keys
while discussions about the right container are ongoing.

If the decision upon a hash-based contained is made, some initial hints about a
good hash algorithm would be:

* http://murilo.wordpress.com/2013/10/16/deeper-look-at-phps-array-worst-case/
* https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function
* A crazy idea to analyze the [most used HTTP headers](
  https://stackoverflow.com/questions/114085/fast-string-hashing-algorithm-with-low-collision-rates-with-32-bit-integer)
  and possibly develop a new algorithm. Such idea is outside of the scope of
  this proposal, but this data could at least help benchmarking considered
  choices.

### The `boost::http::server::server` implementation

I wanted to give maximum flexibility for the user and I added several overloaded
constructors and used an acceptor internally. Some of the constructors would be
just a convenience to save the user from typing. The internal acceptor object
was required to allow these convenience overloads.

To improve the flexibility (delayed open, for instance), I added an accessor
method for the acceptor object. It's possible to also have control over the
acceptor construction by constructing it yourself and then _moving_ it to the
server. ASIO documentation advises to not schedule any operation on the
_moved-from_ object. Then you'll likely want to use the accessor method to the
internal acceptor.

```
namespace boost {
namespace http {
namespace server {
class server {
public:
    server(boost::asio::ip::tcp::acceptor &&acceptor);
    server(boost::asio::io_service &io_service,
           boost::asio::ip::tcp::endpoint
             = boost::asio::ip::tcp::endpoint{boost::asio::ip::tcp::v4{}, 80});

    const boost::asio::ip::tcp::acceptor &acceptor() const;
    boost::asio::ip::tcp::acceptor &acceptor();

    //void boost::asio::tcp::socket
};
}
}
}
```

> TODO :

> decide interface for object recycling and consider how it could be done
> optional to fit well in memory-constrained devices

### Where the proposal needs to be improved?

There's currently poor definition about how error handling should happen and
I'm investigating which is the best approach studying Boost ASIO and Boost AFIO,
at the same time that I compare it with my experience with other async
frameworks.

I'll probably end up using the ASIO approach, but I need to think a bit further
about it.

I still need to consider the ssl connections and integrate it nicely in the
library. By _nicely_ I mean that I need to do it reducing the duplication of
code and considering that the user may want to have multithreaded code.

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
equired once I write the timeline for the project):

* Session support (based on Tufão's session support, which leverages client-side
  and server-side based session stores under the same interface, but with
  improved usability thanks to Boost and C++11).
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

Another thing to keep in mind is that I prefer to implement features on the
order that will benefit my users the most. On my last year's participation in
GSoC, I've had discussions with the users and developers opposed to the "do
first, ask later" approach, I've gathered and implemented features that users
wanted and even collected opinions/votes on diverging features. I'm used to
adopt this approach every time I have contact with the user.
