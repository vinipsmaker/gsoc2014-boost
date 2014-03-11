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

### [pion](https://github.com/cloudmeter/pion)

The project shares some design considerations with cpp-netlib and they won't be
repeated here.

The iterator-based interface in some places is quite nice and can help to avoid
a new allocated container. The query string within urls is an example, but
`pion::http::request::get_queries` returns an `ihash_multimap`, not a custom
wrapper that would play a role similar to the `string_view` proposal.

The complete separation of response pieces (set status code and reason phrase
within two different expressions) looks good, but I don't want to make the API
more difficult than already is (for the user writing handlers, the user writing
backends and me writing even more complex documentation). This API works in
pion, but it isn't very asynchronous.

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

### Others

If you have specific concerns, not only about any library, make an objective and
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

It's unlikely that the user will want to implement this interface. The user will
probably want to use one of the provided backends. However, the user shouldn't
be discouraged by this warning. This is just a friendly reminder that the plan
is to have most of the common backends integrated and general, then he still
will be able to use by just adjusting some parameters, if ever needed.

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

The proposal for executors and schedulers (n3731) face a similar problem and
they've chosen the same solution, `std::function`.

It's guaranteed that the arguments passed to the `handler` will exist as long as
`response` is not finished.

```cpp
namespace boost {
namespace http {
namespace server {
class backend {
public:
    virtual ~backend() {}

    // ### Operations that mirror the ones within request object ###

    // TODO

    // ### Operations that mirror the ones within response object ###
    // see the response class for more details

    virtual void write_continue(response &r) = 0;

    virtual void open(response &r, boost::http::status_code) = 0;
    virtual void open(response &r, int status_code, string reason_phrase) = 0;

    virtual void write_head(response &r, boost::http::status_code) = 0;
    virtual void write_head(response &r, int status_code,
                            string reason_phrase) = 0;

    virtual void write(response &r, const boost::asio::buffer &buffer) = 0;

    virtual void write_trailer(response &r, const header_type &buffer) = 0;
    virtual void write_trailers(response &r, const headers_type &buffer) = 0;

    virtual void end(response &r) = 0;

    virtual void async_write_continue(response &r,
                                      std::function<void(boost::system::error_code)>
                                      callback) = 0;

    virtual void async_open(response &r, boost::http::status_code,
                            std::function<void(boost::system::error_code)>
                            callback) = 0;
    virtual void async_open(response &r, int status_code, string reason_phrase,
                            std::function<void(boost::system::error_code)>
                            callback) = 0;

    virtual void async_write_head(response &r, boost::http::status_code,
                                  std::function<void(boost::system::error_code)>
                                  callback) = 0;
    virtual void async_write_head(response &r, int status_code,
                                  string reason_phrase,
                                  std::function<void(boost::system::error_code)>
                                  callback) = 0;

    virtual void async_write(response &r, const boost::asio::buffer &buffer,
                             std::function<void(boost::system::error_code)>
                             callback) = 0;

    virtual void async_write_trailer(response &r, const header_type &buffer,
                                     std::function<void(boost::system::error_code)>
                                     callback) = 0;
    virtual void async_write_trailers(response &r, const headers_type &buffer,
                                      std::function<void(boost::system::error_code)>
                                      callback) = 0;

    virtual void async_end(response &r,
                           std::function<void(boost::system::error_code)>
                           callback) = 0;
};
}
}
}
```

> TODO

<!--
 choose interface for object recycling and consider how it could be done
 optional to fit well in memory-constrained devices

 The user is responsible to maintain the lifetime ot the \p pool object
 while the request wasn't finished. I thought about use a reference
 instead pointer, but I wanted to make explicit the fact that the pool
 may be used other times. I'm not satisfied with this design yet and I
 plan to improve it.

 pool_type *pool
 -->

### The `boost::http::server::request` class

The SG4's uri library makes use of `optional<string_view>` attributes to expose
URI components. `string_view`, being a non-owning reference, requires the
original uri to be accessible. Thus, `request::uri()` should return a string.

<!-- TODO: check Unicode/ASCII characteres within the HTTP and URI protocols -->

I'm still thinking about whether template based on the uri type should be used
or not, but I want to come with a few arguments before document the decision
here.

The object **MUST** not be destroyed, cleaned, reset or recycled by the backend
or any other entity after the `end` event is reached. This action is only
allowed **after** the `end` event of the associated `response` object. All
internal backends give this guarantee and failing to comply with it triggers
undefined behaviour.

The pair of request and response only is fed to the user once all headers are
collected. This behaviour is chosen because (1) the http parser itself won't
know how to properly handle the connection while all headers aren't received
(eg. body might use fixed lenth or streaming interface, host header might be
missing, whether connection should or not close after response, ...) and (2) the
user itself can only take a proper action once all headers are received (eg.
file server may need the correct range, home page might need cookies, ...). This
is also the behaviour adopted on node.js.

Because request objects are always delivered with `headers` ready, only
callbacks for later events exist.

<!-- TODO: implicit buffering -->

I need to think more before decide if implicit buffering will happen. So, the
signature to register the data callback may change.

```cpp
namespace boost {
namespace http {
namespace server {
class request {
public:
    request(boost::http::server::backend &backend);

    /**
     * Check if request include `100-continue` in the _Expect_ header. It's just
     * a convenient method that doesn't imply backend access overhead.
     *
     * The backend access will only happen in the
     * `boost::http::server::response::write_continue`. See the mentioned
     * function for more details.
     *
     * The name _required_ is used instead _supported_, because a 100-continue
     * status require action from the server.
     */
    bool continue_required() const;

    /**
     * TODO: Further define this function/decision. Even if HTTP/1.1 do allow
     * upgrade, not all backends will provide it. How to integrate it? Further
     * research.
     */
    bool upgrade_required() const;

    string uri();

    // ### SYNC VERSIONS ###

    void read();

    void end();

    // ### END OF SYNC VERSIONS ###

    // ### ASYNC VERSIONS ###

    void async_read(std::function<void(boost::system::error_code)> callback);

    void async_end(std::function<void(boost::system::error_code)> callback);

    // ### END OF ASYNC VERSIONS ###
};
}
}
}
```

> TODO

### The `boost::http::server::response` class

One HTTP response message is made of a status line (HTTP version, status code
and human readable arbitrary status message), some headers and a body (possibly
streamable).

The HTTP version on the status line is responsibility of the backend. The user
should explicitly pass an integer status code and a reason phrase or use a valid
`boost::http::status_code` value to let the server guess a reason phrase. The
status line must be written before any other information and is responsibility
of the user to call a function to write this message before any other function.
Given this behaviour, it's appropriate to use the name `open` for this function,
but `write_head` convenient functions are also provided.

> Previously, there was a confusing paragraph here, but after a lot of thought
> about the implications and several tests with ASIO, I decided the path that I
> was choosing wasn't adequate to the ASIO model, which I even consider better
> and the text/decision was removed.

No matter if you use the blocking calls or the async calls, a minimum checking
is done before dispatching the actions to the backend. The only error checking
done at this stage is related to the order of the actions. Wrong order of
actions are only caused by programming error (failing to comply with a
precondition) and not related to exceptional events outside of programmer's
control. A separate error_category is used only for this class of errors and
could be used in automated tests to help in assuring the code quality.

The headers are an acessible property of the `response` object and will be
streamed just before the first chunk of the body is issued.

This API abstracts several of the different HTTP behaviours, while it still
allow an efficient implementation. The only non-efficient (or even impossible)
implementation would be body streaming through HTTP/1.0 and some server to
process communcation schemes where the backend is forced to buffer the whole
data before the first piece of information is actually sent, but this API also
exposes the `native_stream()` property to protect the user against such
environments. Due to the previous mentioned fact and some others, it's possible
that the server only will execute an action after the next actions are issued.
Thanks to this fact, the user **SHOULDN'T** issue the next action inside the
callback for a previous action. To help the user, order guarantee is mandated
for all backends.

Lastly, I renamed `close` to `end`, because (1) the backend won't actually close
the socket (just the session) and (2) an unclean close/end may happen and an
`async_close` may be desired, but there is no matching `async_close` in ASIO
sockets, then a different name may be desired to help differentiate the two
entities.

![](response_state.png)

```cpp
namespace boost {
namespace http {
namespace server {
/**
 * Represents the current state in the HTTP response.
 *
 * The picture response_state.png can help you understand this file.
 */
enum class response_state
{
    /**
     * This is the initial state.
     *
     * It means that the response object wasn't used yet.
     *
     * At this state, you can only issue the status line or issue a continue
     * action, if continue is supported/used in this HTTP session. Even if
     * continue was requested, issue a continue action is optional and only
     * required if you need the request's body.
     */
    EMPTY,
    /**
     * This state is reached from the `EMPTY` state, once you issue a continue
     * action.
     *
     * No more continue actions can be issued from this state.
     */
    CONTINUE_ISSUED,
    /**
     * This state can be reached either from EMPTY or `CONTINUE_ISSUED`.
     *
     * It happens when the status line is reached (through `open` or `write_head`).
     */
    STATUS_LINE_ISSUED,
    /**
     * This state is reached once the first chunk of body is issued.
     *
     * \warning
     * Once this state is reached, it is no longer safe to access the
     * `boost::http::server::response::headers` attribute, which is left in an
     * unspecified state and might or might nor be used again by the backend.
     * You're safe to access and modify this attribute again once the `FINISHED`
     * state is reached, but the attribute will be at an unspecified state and
     * is recommended to _clear_ it.
     */
    HEADERS_ISSUED,
    /**
     * This state is reached once the first trailer is issued to the backend.
     *
     * After this state is reached, it's not allowed to write the body again.
     * You can either proceed to issue more trailers or `end` the response.
     */
    BODY_ISSUED,
    /**
     * The response is considered complete once this state is reached. You
     * should no longer access this response or the associated request objects,
     * because the backend has the freedom to recycle or destroy the objects.
     */
    FINISHED
};

class response {
public:
    response(boost::http::server::backend &backend,
             bool native_stream);

    /**
     * Returns the current state.
     *
     * \warning
     * The current state is computed from user issued actions and do not reflect
     * the real state of the backend.
     *
     * It's a function with a trivial implementation based on int returning that
     * is useful for testing.
     */
    response_state state() const;

    /**
     * Clear all state (headers, trailers) from the object and give it another
     * backend.
     */
    void reset(boost::http::server::backend &backend,
               bool native_stream);

    /**
     * Export info about the backend/connection behaviour/nature.
     *
     * Native stream also implies trailers support.
     */
    bool native_stream() const;

    // ### SYNC VERSIONS ###
    // Functions in this section might throw boost::system::system_error

    /**
     * Write the 100-continue status that must be written before the client
     * proceed to feed body of the request.
     *
     * \warning If `boost::http::server::request::continue_required` returns
     * true, you **MUST** call this function (before open) to allow the remote
     * client to send the body. If your handler can give an appropriate answer
     * without the body, just reply as usual.
     */
    void write_continue();

    void open(boost::http::status_code);
    void open(int status_code, string reason_phrase);

    void write_head(boost::http::status_code);
    void write_head(int status_code, string reason_phrase);

    void write(const boost::asio::buffer &buffer);

    void write_trailer(const header_type &buffer);
    void write_trailers(const headers_type &buffer);

    void end();

    // ### END OF SYNC VERSIONS ###

    // ### ASYNC VERSIONS with a callback ###

    // Despites being async, the backend is responsible for guarantee the
    // delivery order of the actions issued.

    // If any issue occurs, the backend **MUST** ignore the remaining queued
    // callbacks/actions and is free to delete them, but the response object
    // itself (and the associated request object) must stay alive and unchanged
    // until the user specifically calls `end`.

    // I'm still thinking about behaviour on empty callback. Maybe it'd allow
    // further optimization on the backend by not storing the backends at all.
    // I'm still thinking and I won't describe this behaviour for now. (TODO)

    void async_write_continue(std::function<void(boost::system::error_code)>
                              callback);

    void async_open(boost::http::status_code,
                    std::function<void(boost::system::error_code)> callback);
    void async_open(int status_code, string reason_phrase,
                    std::function<void(boost::system::error_code)> callback);

    void async_write_head(boost::http::status_code,
                          std::function<void(boost::system::error_code)>
                          callback);
    void async_write_head(int status_code, string reason_phrase,
                          std::function<void(boost::system::error_code)>
                          callback);

    void async_write(const boost::asio::buffer &buffer,
                     std::function<void(boost::system::error_code)> callback);

    void async_write_trailer(const header_type &buffer,
                             std::function<void(boost::system::error_code)>
                             callback);
    void async_write_trailers(const headers_type &buffer,
                              std::function<void(boost::system::error_code)>
                              callback);

    void async_end(std::function<void(boost::system::error_code)> callback);

    // ### END OF ASYNC VERSIONS ###

    /**
     * Once the first chunk of body is sent, this attribute will enter in an
     * unspecified state and might or might not be used again by the backend.
     *
     * It's recommended to the user to _clear_ this attribute once the object is
     * done.
     */
    boost::http::headers headers;
};
}
}
}
```

> TODO

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

### The `boost::http::server::server_backend` implementation

This is just a class implementing the `boost::http::server::backend` interface.
I don't repeat inherited/implemented methods from its parent class here. But
there aren't new functions also (follow the text to understand why), then no
functions are listed at all.

I thought about adding a `virtual boost::asio::ip::tcp::socket socket()` to
allow the SSL backend share most of the code with the TCP backend, but SSL
exposes different different functions to register the callback and it is more
than just the matter of configuring the socket at startup (without mentioning
the session setup mess that I'd need to create).

I also thought about some kind of `push_socket`/`handle_socket` interface, but
it doesn't work well, because (1) it becomes more difficult to manage object
lifetime (unless you introduce yet another level of `std::shared_ptr`) and (2)
`ssl::stream` "likes" to own the underlying socket.

Then I thought about passing the `boost::asio::ip::tcp::acceptor` as argument to
the constructor and expose helper protected functions to manage the manage the
session objects, but the session object itself would need to be different (store
`ssl::stream` or `ip::tcp::socket`) and I give up on the hierarchical design. In
fact, I think I can make the hierarchical design happen, but it'd become too
complex with no justified benefits.

And all this effort spent just to give up on the inheritance. But this isn't a
problem, because http handlers don't communicate directly with the backend and
thus references to some _base class_ will be avoided. SSL/HTTPS will have its
own class.

Arguing even further, the interface of this class isn't that interesting,
because as long as it implement all the backend requirements, the user won't
care and it's unlikely to notice it. So, I'll define its interface later.

```
namespace boost {
namespace http {
namespace server {
class server_backend: public boost::http::server::backend
{
    // implemented pure virtual methods

    // communication with server_acceptor
};
}
}
}
```

### The `boost::http::server::server_acceptor class

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

```cpp
namespace boost {
namespace http {
namespace server {
class server_acceptor {
public:
    typedef std::function<void(request&, response&)> handler_type;

    server_acceptor(boost::asio::ip::tcp::acceptor &&acceptor);
    server_acceptor(boost::asio::io_service &io_service,
                    boost::asio::ip::tcp::endpoint
                    = boost::asio::ip::tcp::endpoint{boost::asio::ip::tcp::v4{},
                                                     80});

    const boost::asio::ip::tcp::acceptor &acceptor() const;
    boost::asio::ip::tcp::acceptor &acceptor();

    void handle(handler_type handler);

    void async_handle(handler_type handler);
};
}
}
}
```

> TODO

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
