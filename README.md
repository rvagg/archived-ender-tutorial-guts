# How Ender bundles libraries for the browser
***By [Rod Vagg](https://twitter.com/rvagg)</a>***

I was asked an interesting Ender question on IRC (#enderjs on Freenode) and as I was answering it, it occurred to me that the subject would be an ideal way to explain how Ender's multi-library bundling works. So here is that explanation!

The original question went something like this:

> When a browser first visits my page, they only get served Bonzo (a DOM manipulation library) as a stand-alone library, but on returning visits they are also served Qwery (a selector engine), Bean (an event manager) and a few other modules in an Ender build. Can I integrate Bonzo into the Ender build on the browser for repeat visitors?

## Wait, what's Ender?

Let's step back a bit and start with some basics. The way I generally explain Ender to people is that it's two different things:

1. It's a build tool, for bundling JavaScript libraries together into a single file. The resulting file constitutes a new "framework" based around the jQuery-style DOM element collection pattern: `$('selector').method()`. The constituent libraries provide the functionality for the *methods* and may also provide the selector engine functionality.
2. It's an *ecosystem* of JavaScript libraries. Ender promotes a small collection of libraries as a base, called **The Jeesh**, which together provide a large portion of the functionality normally required of a JavaScript framework, but there are many more libraries compatible with Ender that add extra functionality. Many of the libraries available for Ender are also usable outside of Ender as stand-alone libraries.

The Jeesh is made up of the following libraries, each of these also works as a stand-alone library:

* **[domReady](https://github.com/ded/domready)**: detects when the DOM is ready for manipulation. Provides `$.domReady(callback)` and `$.ready(callback)` methods.
* **[Qwery](https://github.com/ded/qwery)**: a small and fast CSS3-compatible selector engine. Does the work of looking up DOM elements when you call `$('selector')` and also provides `$(elements).find('selector')`, `$(elements).and(elements)` and `$(elements).is('selector')`.
* **[Bonzo](https://github.com/ded/bonzo)**: a DOM manipulation library, providing some of the most commonly used methods, such as `$(elements).css('property', 'value')`, `$(elements).empty()`, `$(elements).after(elements||html)`, and many more.
* **[Bean](https://github.com/fat/bean)**: an event manager, provides jQuery-style `$(elements).bind('event', callback)` and others.

The Jeesh gives you the features of these four libraries bundled into a neat package for only *11.7 kB* minified and gzipped.

## Starting with the basics, Bonzo stand-alone

Bonzo is a great way to start getting your head around Ender because it's so useful by itself. Let's include it in a page and do some really simple DOM manipulation with it.

```html
<!DOCTYPE HTML>
<html lang="en-us">
<head>
  <meta http-equiv="Content-type" content="text/html; charset=utf-8">
  <title>Example 1</title>
</head>
<body>
  <script src="bonzo.js"></script>
  <script id="scr">
    // the contents of *this* script,
    var scr = document.getElementById('scr').innerHTML

    // create a <pre></pre>
    var pre = bonzo.create('<pre>')

    // fill it with the script text, append it to body and style it
    bonzo(pre)
      .text(scr)
      .css({
        fontWeight: 'bold',
        border: 'solid 1px red',
        margin: 10,
        padding: 10
      })
      .appendTo(document.body);

  </script>
</body>
</html>
```

*You can run this as [example1](http://rvagg.github.com/ender-tutorial-guts/example1/example1.html), also available in my GitHub [repository](https://github.com/rvagg/ender-tutorial-guts) for this article.*

This should look relatively familiar to a jQuery user -- you can see that Bonzo is providing some of the important utilities you need for modifying the DOM.

## Jumping straight in, Bonzo inside Ender

Let's see what happens when we use a simple Ender build that includes Bonzo. We'll also include Qwery so we can skip the `document.getElementById()` noise, and we'll also use Bean to demonstrate how neatly the libraries can mesh together.

This is done on the command line with: `ender build qwery bean bonzo`. A file named *ender.js* will be created that can be loaded on a suitable HTML page.

Our script becomes:

```js
$('<pre>')
  .text($('#scr').text())
  .css({
    fontWeight: 'bold',
    border: 'solid 1px red',
    margin: 10,
    padding: 10
  })
  .bind('click', function () {
    alert('Clickety clack');
  })
  .appendTo('body');
```

*You can run this as [example2](http://rvagg.github.com/ender-tutorial-guts/example2/example2.html), also available in my GitHub [repository](https://github.com/rvagg/ender-tutorial-guts) for this article.*

Bonzo performs most of the work here but it's bundled up nicely into the `$` object (also available as `ender`). The previous example can be summarised as follows:

* `bonzo.create()` is now working when HTML is passed to `$()`.
* Qwery does the work when `$()` is called with anything else, in this case `$('#scr')` is used as a selector for the script element.
* We're using the no-argument variant of `bonzo.text()` to fetch the `innerHTML` of the script element.
* Bean makes a showing with the `.bind()` call, but the important point is that it's integrated into our call-chain even though it's a separate library. This is where Ender's bundling magic shines.
* `bonzo.appendTo()` takes the selector argument which is in turn passed to Qwery to fetch the selected element from the DOM (`document.body`).

Also important here, which we haven't demonstrated, is we can do all of this on multiple elements in the same collection. The first line could be changed to `$('<pre></pre><pre></pre>')` and we'd end up with two blocks, both responding to the click event.

## Getting educational, pull it apart again!

It's possible to pull Bonzo out of the Ender build and manually stitch it back together again. Just like we used to do with our toys when we were children! (Or was that just me?)

First, our Ender build is now created with: `ender build qwery bean` (or we could run `ender remove bonzo` to remove Bonzo from the previous example's `ender.js` file).  The new `ender.js` file will contain the selector engine goodness from Qwery, and event management from Bean, but not much else.

Bonzo can be loaded separately, but we'll need some special glue to do this. In Ender parlance, this glue is called an Ender **Bridge**.

### The Ender Bridge

Ender follows the basic CommonJS Module pattern -- it sets up a simple module registry and gives each module a `module.exports` object and a `require()` method that can be used to fetch any other modules in the build. It also uses a `provide('name', module.exports)` method to insert exports into the registry with the name of your module. The exact details here aren't important and I'll cover how you can build your own Ender module in a later article, for now we just need a basic understanding of the module registry system.

Using our Qwery, Bean and Bonzo build, the file looks something like this:

```
|========================================|
| Ender initialisation & module registry |
| (we call this the 'client library')    |
|========================================|
| 'module.exports' setup                 |
|----------------------------------------|
| Qwery source                           |
|----------------------------------------|
| provide('qwery', module.exports)       |
|----------------------------------------|
| Qwery bridge                           |
==========================================
| 'module.exports' setup                 |
|----------------------------------------|
| Bean source                            |
|----------------------------------------|
| provide('bean', module.exports)        |
|----------------------------------------|
| Bean bridge                            |
==========================================
| 'module.exports' setup                 |
|----------------------------------------|
| Bonzo source                           |
|----------------------------------------|
| provide('bonzo', module.exports)       |
|----------------------------------------|
| Bonzo bridge                           |
==========================================
```

To be a useful Ender library, the code should be able to adhere to the CommonJS Module pattern if a `module.exports` or `exports` object exists. Many libraries already do this so they can operate both in the browser and in a CommonJS environment such as Node. Consider Underscore.js for example, it [detects the existence of `exports`](https://github.com/documentcloud/underscore/blob/ca0df9076079a3b2c45475ddb2299fb901a29989/underscore.js#L56-63) and inserts itself onto that object if it exists, otherwise it inserts itself into the global (i.e. `window`) object. This is how Ender compatible libraries that can also be used as stand-alone libraries work too.

So, skipping over the complexities here, our libraries are registered within Ender and then we encounter the **Bridge**. Technically the bridge is just an arbitrary piece of code that Ender-compatible libraries are allowed to provide the Ender CLI tool; it could be anything. The intention, though, is to use it as a glue to bind the library into the core `ender` / `$` object. A bridge isn't necessary and can be omitted -- in this case everything found on `module.exports` is automatically bound to the `ender` / `$` object. Underscore.js doesn't need a bridge because it conforms to the standard CommonJS pattern and its methods are utilities that logically belong on `$` -- for example, `$.each(list, callback)`. If a module needs to operate on `$('selector')` collections then it needs a special binding for its methods. Many modules also require quite complex bindings to make them work nicely inside the Ender environment.

Bonzo has one of the most complex bridges that you'll find in the Endersphere, so we won't be looking into it here. If you're interested in digging deeper, a simpler bridge with some interesting features can be found in [Morpheus](https://github.com/ded/morpheus/blob/master/src/ender.js), an animation framework for Ender. Morpheus adds a `$.tween()` method and also an `$('selector').animate()` and some additional helper methods.

The simplest form of Ender bridge is one that lifts the `module.exports` methods to a new *namespace*. Consider [Moment.js](http://momentjs.com/), the popular date and time library. When used in a CommonJS environment it adds all of its methods to `module.exports`. Without a bridge, when added to an Ender build you'd end up with `$.utc()`, `$.unix()`, `$.add()`, `$.subtract()` and other methods that don't have very meaningful names outside of Moment.js.  They are also likely to conflict with other libraries that you may want to add to your Ender build. The logical solution is to lift them up to `$.moment.utc()` etc., then you also get to use the exported main function as `$.moment(Date|String|Number)`. To achieve this, Moment.js' [bridge](https://github.com/timrwood/moment/blob/master/ender.js) looks like this:

```js
$.ender({ moment: require('moment') })
```

The `$.ender()` method is the way that a bridge can add methods to the global `ender` / `$` object, it takes an optional boolean argument to indicate whether the methods can operate on DOM element collections, i.e. `$('selector').method()`.

### Bonzo in parts

Back to what we were originally trying to achieve: we're loading Bonzo as a stand-alone library and we want to integrate it into an Ender build in the browser. There are two important things we need to do to achieve this: (1) load Bonzo's bridge so it can wire Bonzo into Ender, and (2) make Ender aware of Bonzo so a `require('bonzo')` will do the right thing because this is how the bridge fetches Bonzo.

Let's first do this the easy way. With an Ender build that just contains Qwery and Bean and Bonzo's bridge in a separate file named *bonzo-ender-bridge.js*, we can do the following:

```html
<!-- the order of the first two doesn't matter -->
<script src="ender.js"></script>
<script src="bonzo.js"></script>
<script>
  provide('bonzo', bonzo)
</script>
<script src="bonzo-ender-bridge.js"></script>
```

If you look at the diagram of the Ender file structure above you'll see that we're replicating it with our `<script>` tags but replacing `provide('bonzo', module.exports)` with `provide('bonzo', bonzo)` as Bonzo has detected that it's not operating inside of a CommonJS environment with `module.exports` available. Instead, it's attached itself to the global (`window`) object. Both `provide()` and `require()` are available on the global object and can be used outside of Ender (for example, to extract Bean out of an integrated build you could simply `var bean = require('bean')`.)

We can now continue to use exactly the same script as in our fully integrated Ender build example:

```js
$('<pre>')
  .text($('#scr').text())
  .css({
    fontWeight: 'bold',
    border: 'solid 1px red',
    margin: 10,
    padding: 10
  })
  .bind('click', function () {
    alert('Clickety clack');
  })
  .appendTo('body');
```

*You can run this as [example3](http://rvagg.github.com/ender-tutorial-guts/example3/example3.html), also available in my GitHub [repository](https://github.com/rvagg/ender-tutorial-guts) for this article.*

## Trimming down the `<script>` tags

The main problem with the last example is that we have three `<script>` tags in our page with files loading (synchronously) from our server. We can trim that down to just two, and if *bonzo.js* is already cached in the browser then it'll just be loading one script.

We could achieve this by hacking our ender.js file to include the needed code, or, we could create our own Ender package that contains our code so they will persist even after the Ender CLI tool has touched the file.

First we make a new directory to contain our package. We'll include the Bonzo bridge as a separate file and also create a file for our `provide()` statement. Finally, a basic *package.json* file points to our `provide()` file as the *source* ("main") of the package and the Bonzo bridge as our *bridge* ("ender") file:

```js
{
  "name": "fake-bonzo",
  "version": "0.0.0",
  "description": "Fake Bonzo",
  "main": "main.js",
  "ender": "bonzo-ender-bridge.js"
}
```

We then point the Ender CLI to this directory: `ender build qwery bean ./fake-bonzo/` (or we could run `ender add ./fake-bonzo/` to add it to the ender.js created in the above example).

The completed page now looks like this:

```html
<!DOCTYPE HTML>
<html lang="en-us">
<head>
  <meta http-equiv="Content-type" content="text/html; charset=utf-8">
  <title>Example 4</title>
</head>
<body>
  <script src="bonzo.js"></script>
  <script src="ender.js"></script>
  <script id="scr">
    $('<pre>')
      .text($('#scr').text())
      .css({
        fontWeight: 'bold',
        border: 'solid 1px red',
        margin: 10,
        padding: 10
      })
      .bind('click', function () {
        alert('Clickety clack');
      })
      .appendTo('body');

  </script>
</body>
</html>
```

You can dig further into this and run it as [example4](http://rvagg.github.com/ender-tutorial-guts/example4/example4.html), also available in my GitHub [repository](https://github.com/rvagg/ender-tutorial-guts) for this article.

## That's all folks!

Hopefully this has helped demystify the way that Ender packages libraries together; it's really not magic. If you want to dig deeper then a good place to start would be to examine the [client library](https://github.com/ender-js/ender-js/blob/master/ender.js) that appears at the top of each Ender build&mdash;it's relatively straightforward and fairly short.

<i><a rel="license" href="http://creativecommons.org/licenses/by/3.0/deed.en_US">This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/deed.en_US">Creative Commons Attribution 3.0 Unported License</a>.</i>