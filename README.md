Installation
---

From NPM:

    npm install private

From GitHub:

    cd path/to/node_modules
    git clone git://github.com/benjamn/private.git
    cd private
    npm install .

Introduction
---

In JavaScript, the only data that are truly private are local variables
whose values do not *leak* from the scope in which they were defined.

This notion of *closure privacy* is powerful, and it readily provides some
of the benefits of traditional data privacy, a la Java or C++:
```js
function MyClass(secret) {
    this.increment = function() {
        return ++secret;
    };
}

var mc = new MyClass(3);
console.log(mc.increment()); // 4
```
You can learn something about `secret` by calling `.increment()`, and you
can increase its value by one as many times as you like, but you can never
decrease its value, because it is completely inaccessible except through
the `.increment` method. And if the `.increment` method were not
available, it would be as if no `secret` variable had ever been declared,
as far as you could tell.

This style breaks down as soon as you want to inherit methods from the
prototype of a class:
```js
function MyClass(secret) {
    this.secret = secret;
}

MyClass.prototype.increment = function() {
    return ++this.secret;
};
```
The only way to communicate between the `MyClass` constructor and the
`.increment` method in this example is to manipulate shared properties of
`this`. Unfortunately `this.secret` is now exposed to unlicensed
modification:
```js
var mc = new MyClass(6);
console.log(mc.increment()); // 7
mc.secret -= Infinity;
console.log(mc.increment()); // -Infinity
mc.secret = "Go home JavaScript, you're drunk.";
mc.increment(); // NaN
```
Another problem with closure privacy is that it only lends itself to
per-instance privacy, whereas the `private` keyword in most
object-oriented languages indicates that the data member in question is
visible to all instances of the same class.

Suppose you have a `Node` class with a notion of parents and children:
```js
function Node() {
    var parent;
    var children = [];

    this.getParent = function() {
        return parent;
    };

    this.appendChild = function(child) {
        children.push(child);
        child.parent = this; // Can this be made to work?
    };
}
```
The desire here is to allow other `Node` objects to manipulate the value
returned by `.getParent()`, but otherwise disallow any modification of the
`parent` variable. You could expose a `.setParent` function, but then
anyone could call it, and you might as well give up on the getter/setter
pattern.

This module solves both of these problems.

Usage
---

Let's revisit the `Node` example from above:
```js
var p = require("private").makeAccessor();

function Node() {
    var privates = p(this);
    var children = [];

    this.getParent = function() {
        return privates.parent;
    };

    this.appendChild = function(child) {
        children.push(child);
        var cp = p(child);
        if (cp.parent)
            cp.parent.removeChild(child);
        cp.parent = this;
        return child;
    };
}
```
Now, in order to access the private data of a `Node` object, you need to
have access to the unique `p` function that is being used here.  This is
already an improvement over the previous example, because it allows
restricted access by other `Node` instances, but can it help with the
`Node.prototype` problem too?

Yes it can!
```js
var p = require("private").makeAccessor();

function Node() {
    p(this).children = [];
}

var Np = Node.prototype;

Np.getParent = function() {
    return p(this).parent;
};

Np.appendChild = function(child) {
    p(this).children.push(child);
    var cp = p(child);
    if (cp.parent)
        cp.parent.removeChild(child);
    cp.parent = this;
    return child;
};
```
Because `p` is in scope not only within the `Node` constructor but also
within `Node` methods, we can finally avoid redefining methods every time
the `Node` constructor is called.

Now, you might be wondering how you can restrict access to `p` so that no
untrusted code is able to call it. The answer is to use your favorite
module pattern, be it CommonJS, AMD `define`, or even the old
Immediately-Invoked Function Expression:
```js
var Node = (function() {
    var p = require("private").makeAccessor();

    function Node() {
        p(this).children = [];
    }

    var Np = Node.prototype;

    Np.getParent = function() {
        return p(this).parent;
    };

    Np.appendChild = function(child) {
        p(this).children.push(child);
        var cp = p(child);
        if (cp.parent)
            cp.parent.removeChild(child);
        cp.parent = this;
        return child;
    };

    return Node;
}());

var parent = new Node;
var child = new Node;
parent.appendChild(child);
assert.strictEqual(child.getParent(), parent);
```
Because this version of `p` never leaks from the enclosing function scope,
only `Node` objects have access to it.

So, you see, the claim I made at the beginning of this README remains
true:

> In JavaScript, the only data that are truly private are local variables
> whose values do not *leak* from the scope in which they were defined.

It just so happens that closure privacy is sufficient to implement a
privacy model similar to that provided by other languages.
