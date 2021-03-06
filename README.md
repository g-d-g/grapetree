`grapetree`
============

A simple, powerful generalized hierarchical path-routing module for client-side and server-side applications.

Using grapetree, applications can be written as a tree of routes where a number of routes are active at any given time.
A URL is broken into a set of routes and each route can perform some work like loading data and rendering views.
Two routes that share a parent route can share some of the same functionality or data.

Like [crossroads.js](http://millermedeiros.github.io/crossroads.js/), [router.js](https://github.com/tildeio/router.js), and [cherrytree](https://github.com/QubitProducts/cherrytree),
`grapetree` embraces the [single-responsibility principle ](http://en.wikipedia.org/wiki/Single_responsibility_principle)
and is entirely stand-alone, free of dependencies on a framework of any kind.

For a frontend application using grapetree, routes are the central part of the application. Its where you manage the
lifecycle of your views - when to create or destroy your views, and often when to change major parts of their state.

Features
=====================
* Intuitive nested routing description
* Route lifecycle hooks so each part of the path can have a setup and cleanup function
* Intuitive error bubbling (errors bubble to the nearest handler up the tree)
* Small footprint (< 16 KB unminified uncompressed)

Motivation
============

The main motivation for this library is to enable powerful and easy-to-understand url-routing for single-page applications.
Very often, you want to build a website that gives users URLs they can save or share, and changes the url depending on their location within the site.
Url routing is the way you describe how to switch between urls.

While GrapeTree doesn't proscribe whether you use traditional [url hash navigation](http://oshyn.com/_blog/General/post/JavaScript_Navigation_using_Hash_Change/),
or the more modern [browser history API (or "pushState")](https://developer.mozilla.org/en-US/docs/Web/Guide/API/DOM/Manipulating_the_browser_history),
it gives you the scaffolding you need to appropriately manage URLs.

Example
========

Lets pretend we're writing a blog site. Here's a basic set of routes for some category pages, article pages, search, and an about section:

```javascript
var Grapetree = require('grapetree')

var router = Grapetree(function() { // root

    this.route('blog', function() { // triggered by a path like '/blog'
        this.enter(function() {  // runs when the page is changed to '/blog/...'
            loadBlog()
            var blogSwitchFuture = blogSwitch('highlights')
            return blogSwitchFuture // sub-route (eg /blog/category/x) will wait until this future is resolved to run
        })
        this.exit(function() {  // runs when the page is change from '/blog/...' to something else (like '/article/...')
            hideBlog()
        })

        // paths with parameters:
        this.route('category/{}', function(category) { // triggered by a path like '/blog/category/all'
            this.enter(function() {
                blogSwitch(category)
            })
            this.exit(function() {
                blogSwitch('highlights') // back to highlights
            })
        })
    })

    this.route('article/{}', function(number) {
        this.enter(function() {
            loadArticle(number)
        })
    })

    this.route('search/{}', function(query) { // triggered by a url like /search/"whatever"
        this.enter(function() {
            renderSearchResults(query)
        })
        this.exit(function() {
            destroySearchResults()
        })
    })

    this.route('about', function() {
        this.redirect('/about/info') // redirects to render the 'info' subroute if the url has no more parts after 'about'

        this.route('info', function() {
            this.enter(function() {
                showInfo()
            })
        })
        this.route('contact', function() {
            this.enter(function() {
                showContactPage()
            })
        })
    })

    this.default(function(path) {   // default routes can be set up if none match
    	this.enter(function() {
        	show404Page("Sorry, "+path+" doesn't exist")
        })
    })
})

// listen for browser back/forward button changes
window.onpopstate = function() {
    router.go(window.location.pathname).done()
}
// trigger the browser to change its url when your application changes the router path
router.on('go', function(path) {
    var cur = window.location
    if(cur.pathname !== path) {  // only push state if the url is different
         history.pushState({}, 'title', cur.protocol+'//'+cur.host+path)
    }
})

// on-load, run the page's location through the router to load the appropriate stuff
if(window.location.pathname === '/') {
    // some default route you want to take the person to on load of the root page
    router.go('/ticket/create', false).done()  // make sure that if you don't pass the future returned by `go` anywhere, that you call done on it (see async-future's docs for more info)
} else {
    router.go(window.location.pathname, false).done()
}

```

See [grapetree-core's unit tests](https://github.com/fresheneesz/grapetree-core/blob/master/test/grapetreeCoreTest.js) for an exhaustive set of examples.

Install
=======

```
npm install grapetree
```

Usage
=====

```javascript
var GrapeTree = require('grapetree')
```

`GrapeTree(routeDefinition)` - Returns a new router instance based on `routeDefinition`, which should be a function that gets a `Route` object as its `this` context.

`GrapeTree.Future` - A reference to [the async-future module](https://github.com/fresheneesz/asyncFuture), which `grapetree` uses internally. This does not have to be the futures/promises implementation you use to return a future from `enter` and `exit` handlers, but a future must have a `then`, `catch`, and `finally` method.

Router objects
--------------

`router.go(newPath[, emitChangeEvent])` - Changes the current path and triggers the router to fire all the appropriate handlers. Returns [a future](https://github.com/fresheneesz/asyncFuture) that is resolved when the route is complete or has an error that isn't handled by a Route's error handler.

* `newPath` - The path to change to.
* `emitChangeEvent` - *(default true)* Whether to emit the `"change"` event.
* `softQueue` - *(default true)* If true, causes the path to only be executed if it's the last one in the queue (and be discarded otherwise). If false, every queued path is handled in order.

`router.cur` - gets the current path (transformed if a transform is being used).

`router.transformPath(trasformFns)` - Sets up path transformation, which modifies the internal path before passing it as an argument to the `"change"` event and `Route.default` handlers and after getting an external path from the `router.go` and `Route.route` functions. This is mostly used for libraries that want to extend grapetree (like what grapetree itself does with grapetree-core).

* trasformFns - an object like {toExternal: function(internalPath){...}, toInternal: function(externalPath){...}}

`router.on` - router inherits from [EventEmitter](http://nodejs.org/api/events.html) and so gets all the methods from it like `router.once` and `router.removeListener`. This can throw an exception if no Route `error` handlers catch an exception.

#### Router events

* 'change' - Emitted after all the handlers for a particular new path have been run. This is the only event. The event data contains the new path.

Route objects
--------------

`this.route(pathSegment, routeDefinition)` - creates a sub-path route. The routes are tested for a match in the order they are called - only one will be used.

* `pathSegment` - the parts of the path to match a route path against (e.g. /a/b or /x). The route only matches if each item in `pathSegment` matches the corresponding parts in the path being changed to. If any of the items in the array are `"{}"`, matching parts of the path being changed to are treated as parameters that will be passed to the `routeDefinition` function.
* `routeDefinition` - a function that gets a `Route` object as its `this` context. It is passed any parameters that exist in `pathSegment` in the same order.

`this.default(routeDefinition)` - creates a default sub-path route that is matched if no other route is. The `routeDefinition` function is passed the full sub-route (ie if the default is within "/a/b" and the path is "/a/b/c/d", default will get "/c/d")

* `routeDefinition` - a function that gets a `Route` object as its `this` context. It is passed the new pathSegment being changed to. If `router.transformPath` has been called, the parameter will have been transformed with the transform.

`this.redirect(newPath[, emitOldPath=false])` - Changes the route to be loaded only if no subroute matches. If a subroute matches, the redirect is ignored.

* newPath - the new route
* emitOldPath - if true, the 'change' event will be triggered with the *original* path

`this.enter(handler)` - sets up a handler that is called when a path newly "enters" the subroute (see **Route Lifecycle Hooks** for details).

* `handler(parentValue)` - a function that will be called when the path is "entered". The handler may return [a future](https://github.com/fresheneesz/asyncFuture), which will be waited on before child enter-handlers are called.
  * `parentValue` is the value of the future returned by its parent's enter handler, or undefined if no future was returned by its parent.

`this.exit(handler)` - sets up a handler that is called when a new path "exits" the subroute (see **Route Lifecycle Hooks** for details).

* `handler(parentValue, divergenceDistance)` - a function that will be called when the path is "exited". The handler may return [a future](https://github.com/fresheneesz/asyncFuture), which will be waited on before parent exit-handlers are called.
  * `divergenceDistance` is the number of routes between it and the recent path-change (e.g. for a change from /a/b/c/d to /a/b/x/y, c's divergence distance is 0, and d's is 1). This is useful, for example, if some work your exit handler does is unnecessary if its parent route's exit handler is called.
  * `parentValue` is the value of the future returned by its parent's **enter** handler (*not* its parent's or child's exit handler), or undefined if no future was returned by its parent.

`this.error(errorHandler)` - Sets up an error handler that is passed errors that happen anywhere in the router. If an error handler is not defined for a particular subroute, the error will be passed to its parent. If an error bubbles to the top, the error is thrown from the `router.go` function itself. The handler may return [a future](https://github.com/fresheneesz/asyncFuture), which will propogate errors from that future to the next error handler up, if that future resolves to an error.

* `errorHandler(error, info)` - A function that handles the `error`. The second parameter is an object with info about where the error happened. It has the following members:
  * `info.stage` - the stage of path-changing the error happened in. `stage` can be either "enter", "exit", or "route"
  * `info.location` - the path segement (relative to the current route) where the error happened ('' indicates the current route)

Route Lifecycle Hooks
-------------

#### Handler order

1. 'change' event handler
2. Exit handlers - from outermost to the divergence route (the route who's parent still matches the new route)
3. Enter handlers - from the convergence route (the route matching the first segement of the new path) to the outermost new route

#### Explanation

The routing hooks in `grapetree` are simple but powerful. Basically exit handlers are called from leaf-node routes inward, and enter handlers are called outward toward the leaf-nodes.

```javascript
var router = Router(function() { // root
    this.route('a', function() {
    	this.enter(function() {
       		// entering a
        })
        this.exit(function() {
        	// exiting a
        })
        this.route('x', function() {
            this.enter(function() {
                // entering x
            })
            this.exit(function() {
                // exiting x
            })
        })
    })
    this.route('b', function() {
    	this.enter(function() {
        	// entering b
        })
        this.exit(function() {
        	// exiting b
        })
    })
})

router.on('change', function(newPath) {
    console.log('went to '+newPath.join(','))
})

router.go('/a/x').then(function() {
    return router.go('/b')
}).done()
```

The order the handlers are called in the above example is:

1. change event: "went to a,x"
2. entering a
3. entering x
4. change event: "went to b"
5. exiting x
6. exiting a
7. entering b

Error Handling
==============

Error handling in grape-tree is an attempt to be as intuitive as possible, but they are a bit different from traditional try-catch, because the success of a parent route should not depend on the success of a child route (as opposed to normal try-catch situations where the calling code's success depends on the called code).
Here are some facts about how errors are handled:

* If a child route has an error and it bubbles up past its parent, its parent is not actually affected - all its enter and exit handlers are called as normal.
* Errors bubble up from the route where they happened, to the error handler of nearest ancestor route that has one.
* If a route has an error, the path will be incomplete (e.g. if you try to go to /a/b/c/d and there was an error at c, the path will end up being /a/b)

See [grapetree-core's unit tests](https://github.com/fresheneesz/grapetree-core/blob/master/test/grapetreeCoreTest.js) for exhaustive examples of how error handling works. For the most part, you should be able to use it well without fully understanding the intricacies of how it works.

Default Handlers
================

Like the error handlers, default handlers for a route also cover the scope of that route's children. In other words, if a child doesn't have a default route, its parent's (or grandparent's etc) default route will be used.
This allows you to have a single default handlers at the top level that will catch any invalid route.

Techniques
==========

One non-obvious technique that you can use is to use an empty route to use a pass-through route to group code that is identical for only some paths in a given level.
For more info, look at the unit test "pass-through route".

Recommendations
===============

If you're using grapetree for URL routing, I recommend that all URL changes that happen in your code go through grapetree using `router.go`, and conversely, any action that brings you somewhere that requires a URL change should be done using `router.go`. This is the right way to ensure that your URL always matches where in your application you are.

Todo
====



Changelog
========
* 1.1.1 - pulling in new version of core to fix a bug
* 1.1.0 - adding `router.cur` property
* 1.0.0 - BREAKING CHANGE - pulling in new version of core, which makes softqueue the default.
* 0.4.3 - upgrading core version
* 0.4.2 - upgrading core version
* 0.4.1 - pulling in bug fix from core
* 0.4.0 - pulling in change from core for redirecting
* 0.3.0 - pulling in change from core for bubbling default handlers
* 0.2.0 - pulling in minor api change from core - exit handlers get two arguments now
* 0.1.0 - pulling in minor fix from core and another minor fix
* 0.0.1 - first version

How to Contribute!
============

Anything helps:

* Creating issues (aka tickets/bugs/etc). Please feel free to use issues to report bugs, request features, and discuss changes
* Updating the documentation: ie this readme file. Be bold! Help create amazing documentation!
* Submitting pull requests.

How to submit pull requests:

1. Please create an issue and get my input before spending too much time creating a feature. Work with me to ensure your feature or addition is optimal and fits with the purpose of the project.
2. Fork the repository
3. clone your forked repo onto your machine and run `npm install` at its root
4. If you're gonna work on multiple separate things, its best to create a separate branch for each of them
5. edit!
6. If it's a code change, please add to the unit tests (at test/grapetreeTest.js) to verify that your change
7. When you're done, run the unit tests and ensure they all pass
8. Commit and push your changes
9. Submit a pull request: https://help.github.com/articles/creating-a-pull-request

License
=======
Released under the MIT license: http://opensource.org/licenses/MIT
