# The pools

The pool interface was starting to get complicated and distracting me from the
set core of abstractions, then I decided to move it out of the initial
proposal. The pool idea was introduced to support the interesting nature from
cpp-cms of recycle objects to reduce allocation overhead. This idea **MUST** be
generic to the point of being disableable to fit well in resource-constrained
devices. With the given requirement of generity over pooling behaviour, an
appropriate and sane interface should be defined. If done wrong, this interface
could actually decrease the performance in some scenarios and I'd like to use
templates to customize the pooling behaviour, thus customizing the policy at
compile-time.

Said that, I don't think I'll be able to finish the pool interface until the
proposal submission deadline, then I'm leaving it out. You can find the old and
obsolete (might no more integrate with the rest of the proposal) interface
below:

## The `boost::http::server::backend_pool` interface

I think this class needs a better name, but I don't like `request_response_pool`
either. Moving on...

<!-- TODO: below paragraph -->

And actually, maybe this interface (together with the rest of the proposal) may
be unimplementable, **but** only because I didn't choose a solution for
integrating `request` + `response` yet. I'm presenting the interface anyway,
because you'll find the idea/concept useful.

The possibility to control object recycling is the idea behind this abstraction.
You could use a simple pool that allocates and deallocates objects immediately
on embedded devices and a pool that keeps a cache ready to be used (thus
avoiding too many allocations) and only free it a little after some time of
inactivity. In fact, it's even possible to implement a `backend_pool` that
accepts an allocator.

It's recommended that a new `boost::http::server::backend` implementation should
allow a pointer to a pool object to be passed on the constructor. Such `backend`
is free to use any pool (or no at all) it wants, but the use of
`boost::http::server::simple_backend_pool` is recommended.

```
namespace boost {
namespace http {
namespace server {
class backend_pool
{
public:
    virtual ~backend_pool() {}

    virtual void push(pair<request*,response*> object) = 0;
    virtual pair<request*,response*> pop() = 0;
};
}
}
}
```

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

## The `boost::http::server::simple_backend_pool` implementation

Just an implementation for the `boost::http::server::backend_pool` interface
that allocates/deallocates objects immediately/as-soon-as-requested. It's the
recommended implementation when the user doesn't choose one explicitly.
