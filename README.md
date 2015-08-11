morphdom
========

[![Build Status](https://travis-ci.org/patrick-steele-idem/morphdom.svg?branch=master)](https://travis-ci.org/patrick-steele-idem/morphdom)

_NOTE: This module is experimental, but seems to work in the latest browsers (including IE9+)._

Lightweight module for morphing an existing DOM node tree to match a target DOM node tree. It's fast and works with the real DOM—no virtual DOM here!

This module was created to solve the problem of updating the DOM in response to a UI component or page being rerendered. One way to update the DOM is to simply toss away the existing DOM tree and replace it with a new DOM tree (e.g., `myContainer.innerHTML = newHTML`). While replacing an existing DOM tree with an entirely new DOM tree will actually be very fast, it comes with a cost. The cost is that all of the internal state associated with the existing DOM nodes (scroll positions, input caret positions, CSS transition states, etc.) will be lost. Instead of replacing the existing DOM tree with a new DOM tree we want to _transform_ the existing DOM tree to match the new DOM tree while minimizing the number of changes to the existing DOM tree. This is exactly what the `morphdom` module does! Give it an existing DOM node tree and a target DOM node tree and it will efficiently transform the existing DOM node tree to exactly match the target DOM node tree with the minimum amount of changes.

`morphdom` does not rely on any virtual DOM abstractions. Because `morphdom` is using the _real_ DOM, the DOM that the web browser is maintaining will always be the source of truth. Even if you have code that manually manipulates the DOM things will still work as expected. In addition, `morphdom` can be used with any templating language that produces an HTML string.

The transformation is done in a single pass and is designed to minimize changes to the DOM while still ensuring that the morphed DOM exactly matches the target DOM. In addition, the algorithm used by this module will automatically match up elements that have corresponding IDs and that are found in both the original and target DOM tree.

# Usage

First install the module into your project:

```
npm install morphdom --save
```

The code below shows how to morph one `<div>` element to another `<div>` element.

```javascript
var morphdom = require('morphdom');

var el1 = document.createElement('div');
el1.className = 'foo';

var el2 = document.createElement('div');
el2.className = 'bar';

morphdom(el1, '<div class="bar"></div>');

expect(el1.className).to.equal('bar');
```

You can also pass in an HTML string for the second argument:

```javascript
var morphdom = require('morphdom');

var el1 = document.createElement('div');
el1.className = 'foo';
el1.innerHTML = 'Hello John';

var newHTML = myTemplate.render({name: 'Frank'});
morphdom(el1, newHTML);

expect(el1.className).to.equal('bar');
```

NOTE: This module will modify both the original and target DOM node tree during the transformation. It is assumed that the target DOM node tree will be discarded after the original DOM node tree is morphed.

# API

## morphdom(fromNode, toNode, options) : Node

The `morphdom(fromNode, toNode, options)` function supports the following arguments:

- *fromNode* (`Node`)- The node to morph
- *toNode* (`Node`|`String`) - The node that the `fromNode` should be morphed to (or an HTML string)
- *options* (`Object`) - See below for supported options

The returned value will typically be the `fromNode`. However, in situations where the `fromNode` is not compatible with the `toNode` (either different node type or different tag name) then a different DOM node will be returned.

Supported options (all optional):

- *onNodeDiscarded* (`Function(node)`) - A function that will called when a `Node` in the `from` tree has been discarded and will no longer exist in the final DOM tree.
- *onBeforeMorphEl* (`Function(fromEl, toEl)`) - A function that will called when a `HTMLElement` in the `from` tree is about to be morphed. If the listener function returns `false` then the element will be skipped.
- *onBeforeMorphElChildren* (`Function(fromEl, toEl)`) - A function that will called when the children of an `HTMLElement` in the `from` tree are about to be morphed. If the listener function returns `false` then the child nodes will be skipped.

```javascript
var morphdom = require('morphdom');
var morphedNode = morphdom(fromNode, toNode, {
    onNodeDiscarded: function(node) {

    },
    onBeforeMorphEl: function(fromEl, toEl) {
        return true;
    },
    onBeforeMorphElChildren: function(fromEl, toEl) {
        return true;
    }
});
```

# FAQ

## Isn't the DOM slow?

No, the DOM _data structure_ is not slow. The DOM is a key part of any web browser so it must be fast. Walking a DOM tree and reading the attributes on DOM nodes is _not_ slow. However, if you attempt to read a computed property on a DOM node that requires a relayout of the page then _that_ will be slow. However, `morphdom` only cares about the following properties of a DOM node:

- `el.firstChild`
- `el.tagName`
- `el.nextSibling`
- `el.attributes`

## What about the virtual DOM?

Libraries such as a [React](http://facebook.github.io/react/) and [virtual-dom](https://github.com/Matt-Esch/virtual-dom) solve a similar problem using a _Virtual DOM_. That is, at any given time there will be the _real_ DOM (that the browser rendered) and a lightweight and persistent virtual DOM tree that is a mirror of the real DOM tree. Whenever the view needs to update, a new _virtual_ DOM tree is rendered. The new virtual DOM tree is then compared with the old virtual DOM tree using a diffing algorithm. Based on the differences that are found, the _real_ DOM is then "patched" to match the new virtual DOM tree and the new virtual DOM tree is persisted for future diffing.

Both `morphdom` and virtual DOM based solutions update the _real_ DOM with the minimum number of changes. The only difference is in how the differences are determined. `morphdom` compares real DOM nodes while `virtual-dom` and others only compare virtual DOM nodes.

There are some drawbacks to using a virtual DOM-based approach:

- The real DOM is not the source of truth (the persistent virtual DOM tree is the source of truth)
- The real DOM _cannot_ be modified behind the scenes (e.g., no jQuery) because the diff is done against the virtual DOM tree
- The virtual DOM is an abstraction layer that introduces code overhead
- The virtual DOM representations are not standardized (will vary by implementation)
- The virtual DOM can only efficiently be used with code and templating languages that produce a virtual DOM tree

The premise for using a virtual DOM is that the DOM is "slow". While there is slightly more overhead in creating actual DOM nodes instead of lightweight virtual DOM nodes, we are not seeing any noticeable slowness in our benchmarks. In addition, as web browsers get faster the DOM data structure will also likely continue to get faster so there benefits to avoiding the abstraction layer.

See the [Benchmarks](#benchmarks) below for a comparison of `morphdom` with [virtual-dom](https://github.com/Matt-Esch/virtual-dom).

## Which is better: rendering to an HTML string or rendering virtual DOM nodes?

There are many high performance templating engines that stream out HTML strings with no intermediate virtual DOM nodes being produced. On the server, rendering directly to an HTML string will _always_ be faster than rendering virtual DOM nodes (that then get serialized to an HTML string). In a benchmark where we compared server-side rendering for [Marko](https://github.com/marko-js/marko) (with [Marko Widgets](https://github.com/marko-js/marko-widgets)) and React we found that Marko was able to render pages ten times faster than React with much lower CPU usage (see: [Marko vs React: Performance Benchmark](https://github.com/patrick-steele-idem/marko-vs-react))

In theory, templating languages such as Marko could support two compiled outputs: one that produces HTML strings (for use on the server) and another that produces DOM nodes (for use in the browser). However, based on our benchmarks we see no reason to switch over to rendering DOM nodes. Rendering to an HTML string performs very well on both the server and in the browser and it simplifies template compilers.

## Is this module used by any library/framework?

This module is used by [Marko Widgets](https://github.com/marko-js/marko-widgets). Marko Widgets is a high performance and lightweight UI components framework for the [Marko templating engine](https://github.com/marko-js/marko).

You can see how Marko Widgets compares to React in performance by taking a look at the following benchmark: [Marko vs React: Performance Benchmark](https://github.com/patrick-steele-idem/marko-vs-react)

However, `morphdom` was designed to be standalone and will work with any library or framework.

# Benchmarks

Below are the results on running benchmarks on various DOM transformations for both `morphdom` and [virtual-dom](https://github.com/Matt-Esch/virtual-dom). This benchmark uses a high performance timer (i.e., `window.performance.now()`) if available. For each test the benchmark runner will run `100` iterations. After all of the iterations are completed for one test the average time per iteration is calculated by dividing the total time by the number of iterations.

To run the benchmarks:

```
npm run benchmark
```

The table below shows some sample benchmark results when running the benchmarks on a MacBook Pro (2.8 GHz Intel Core i7, 16 GB 1600 MHz DDR3). The average time per iteration for each test is shown in the table below:

<table>
    <thead>
        <tr>
            <td></td>
            <td>
                virtual-dom
            </td>
            <td>
                morphdom
            </td>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td class="test-name">
                change-tagname
            </td>
            <td>
                0.01ms
            </td>
            <td>
                0.00ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                change-tagname-ids
            </td>
            <td>
                0.02ms
            </td>
            <td>
                0.01ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                data-table
            </td>
            <td>
                0.75ms
            </td>
            <td>
                0.29ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                ids-nested
            </td>
            <td>
                0.01ms
            </td>
            <td>
                0.01ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                ids-nested-2
            </td>
            <td>
                0.02ms
            </td>
            <td>
                0.01ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                ids-nested-3
            </td>
            <td>
                0.08ms
            </td>
            <td>
                0.08ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                ids-nested-4
            </td>
            <td>
                0.03ms
            </td>
            <td>
                0.11ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                ids-nested-5
            </td>
            <td>
                0.03ms
            </td>
            <td>
                0.02ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                ids-nested-6
            </td>
            <td>
                0.02ms
            </td>
            <td>
                0.01ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                ids-prepend
            </td>
            <td>
                0.01ms
            </td>
            <td>
                0.01ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                input-element
            </td>
            <td>
                0.01ms
            </td>
            <td>
                0.06ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                input-element-disabled
            </td>
            <td>
                0.00ms
            </td>
            <td>
                0.00ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                input-element-enabled
            </td>
            <td>
                0.01ms
            </td>
            <td>
                0.01ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                large
            </td>
            <td>
                1.01ms
            </td>
            <td>
                1.61ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                lengthen
            </td>
            <td>
                0.02ms
            </td>
            <td>
                0.02ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                one
            </td>
            <td>
                0.01ms
            </td>
            <td>
                0.00ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                reverse
            </td>
            <td>
                0.02ms
            </td>
            <td>
                0.01ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                reverse-ids
            </td>
            <td>
                0.02ms
            </td>
            <td>
                0.02ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                select-element
            </td>
            <td>
                0.03ms
            </td>
            <td>
                0.02ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                shorten
            </td>
            <td>
                0.01ms
            </td>
            <td>
                0.02ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                simple
            </td>
            <td>
                0.01ms
            </td>
            <td>
                0.01ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                simple-ids
            </td>
            <td>
                0.03ms
            </td>
            <td>
                0.02ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                simple-text-el
            </td>
            <td>
                0.02ms
            </td>
            <td>
                0.01ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                svg
            </td>
            <td>
                0.01ms
            </td>
            <td>
                0.02ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                todomvc
            </td>
            <td>
                0.23ms
            </td>
            <td>
                0.31ms
            </td>
        </tr>
        <tr>
            <td class="test-name">
                two
            </td>
            <td>
                0.01ms
            </td>
            <td>
                0.01ms
            </td>
        </tr>
    </tbody>
</table>

_NOTE: Safari 8.0.7 (10600.7.12)_

# Maintainers

* [Patrick Steele-Idem](https://github.com/patrick-steele-idem) (Twitter: [@psteeleidem](http://twitter.com/psteeleidem))

# Contribute

Pull Requests welcome. Please submit Github issues for any feature enhancements, bugs or documentation problems. Please make sure tests pass:

```
npm test
```

# License

ISC

