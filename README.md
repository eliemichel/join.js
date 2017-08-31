*This is the code repository accompanying the blog post [http://blog.exppad.com/article/promises-without-a-then](http://blog.exppad.com/article/promises-without-a-then). Feel free to comment through the bug tracker or using [webmentions](https://indieweb.org/webmention)*

Promises without a then
=======================

If you got interested in [modern JavaScript](https://hackernoon.com/how-it-feels-to-learn-javascript-in-2016-d3a717dd577f) in the last few years, you might have heard of [Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises). They are a very elegant way of dealing with asynchronous actions and avoid callbacks within callbacks within callbacks.

A practical example
-------------------

Here is a quick reminder (as well as the recipe for French crêpes):

    fetchIngredients(function(ingredients) {
        whisk(ingredients, function(batter) {
            pan.cook(batter, function(crepes) {
                eat(crepes);
            });
        });
    });

becomes, with promises:

    fetchIngredients()
    .then(whisk)
    .then(pan.cook)
    .then(eat);

Much cleaner, you'd say. And even cooler, because one can easily wait for two things running in parallel, like:

    Promises.all([
        fetchIngredients().then(whisk),
        heatPan()
    ])
    .then([batter, pan] => pan.cook(batter))
    .then(eat);

But, lately, as I was cleaning up [a quick WebGL project](#todo), I decided to try and use exclusively promises. No callbacks. And quickly I ran into a basic issue: in a chain of `then()`, how to reuse the value of a previous `then()`? For instance, let's add some details to the recipe:

    fetchIngredients()
    .then(function([flour, eggs, milk]) {
        return whisk([flour, eggs]);
    })
    .then(function(flourAndEggsMixture) {
        // Hey, I need to get the milk, here!
        return flourAndEggsMixture.graduallyAdd(milk);
    })
    .then(pan.cook)
    .then(eat);

There is no way of adding the milk afterwards with this structure. This is terrible, because if you add the milk before whisking the flour and eggs, you'll be in danger of getting lumps in your batter!

To solve this, we should recall that promises are variables, so can be manipulated like variables. We have to use the promise to the ingredients a second time. We make the `flourAndEggsMixture.graduallyAdd` wait for both the mixture and for the ingredients using `Promise.all()`.

**Note:** Yes, the ingredients are already needed to make the mixture, so the ingredients will always be resolved first. But we don't really care, we just say "I need ingredients to be ready".

    let ingredients_promise = fetchIngredients();

    let flourAndEggsMixture_promise = ingredients_promise.then(function([flour, eggs, milk]) {
        return whisk([flour, eggs]);
    });

    let ingredients_and_flourAndEggsMixture_promise = Promise.all([
        ingredients_promise,
        flourAndEggsMixture_promise
    ]);

    ingredients_and_flourAndEggsMixture_promise.then(function([ingredients, flourAndEggsMixture]) {
        let [flour, eggs, milk] = ingredients;
        // Got the milk :)
        return flourAndEggsMixture.graduallyAdd(milk);
    })
    .then(pan.cook)
    .then(eat);

**Note:** I don't like mixing `camlCase` and `snake_case`, but it is used here for some kind of typing, so it's all right. We'll get rid of that later on.

Problem is: this makes the code much much more verbose than the callback based one, which would look like:
    
    fetchIngredients(function([flour, eggs, milk]) {
        whisk([flour, eggs], function(flourAndEggsMixture) {
            flourAndEggsMixture.graduallyAdd(milk, function(batter) {
                pan.cook(batter, function(crepes) {
                    eat(crepes);
                });
            });
        });
    });

Soooo, should we actually switch back to callbacks? No! There is a way to make this beautiful.

First, we feel that we'll have a lot of `foo_promise`, so we'll decide on a short for this. Let's say that `$foo` means "promise to `foo`", so that I can do `$foo.then(function(foo) { ... })`. (The dollar sign `$` is a valid character in JavaScript identifiers). With some additional sugar, our code become:


    let $ingredients = fetchIngredients();
    let $flourAndEggsMixture = $ingredients.then([flour, eggs, milk] => whisk([flour, eggs]));

    let $batter = Promise.all([$ingredients, $flourAndEggsMixture])
    .then(function([ingredients, flourAndEggsMixture]) {
        let [flour, eggs, milk] = ingredients;
        return flourAndEggsMixture.graduallyAdd(milk);
    });

    $batter
    .then(pan.cook)
    .then(eat);

The most important line in this is:

    let $batter = Promise.all([$ingredients, $flourAndEggsMixture])
    .then(function([ingredients, flourAndEggsMixture]) { ... });

If we push a bit further this example, we'll get a lot of such lines. For instance:

    let $crepes = Promise.all([$batter, $hotPan])
    .then(function([batter, hotPan]) { ... });

    let $lemonCrepe = Promise.all([$crepes, $lemonJuice, $sugar])
    .then(function([crepes, lemonJuice, sugar]) { ... });

So, we'll introduce a small short for it:

    function join(promises, callback) {
        return Promises.all(promises).then(callback);
    }

Actually, I also want to get rid of the extra brackets `[` and `]`, so here is a slightly more polished version (that also add an error handler):

    function join() {
        // MIT licensed, copyright (c) 2017 -- Elie Michel <elie.michel@exppad.com>
        let promises = Array.prototype.slice.call(arguments, 0, arguments.length - 1);
        let spreadResolve = arguments[arguments.length - 1];
        let resolve = function(args) { return spreadResolve.apply(this, args); }

        return Promise.all(promises)
        .then(resolve)
        .catch(console.error.bind(console));
    }

I call it `join()` to refer to [the monadic taste of Promises](https://blog.jcoglan.com/2011/03/11/promises-are-the-monad-of-asynchronous-programming/), but you might rather call it just `require()`.

    let $ingredients = fetchIngredients();
    let $flourAndEggsMixture = $ingredients.then([flour, eggs, milk] => whisk([flour, eggs]));

    let $batter = join($ingredients, $flourAndEggsMixture,
        ([flour, eggs, milk], flourAndEggsMixture) => flourAndEggsMixture.graduallyAdd(milk)
    );

    let $crepes = join($batter, $hotPan, (batter, hotPan) => hotPan.cook(batter));

    let $lemonCrepe = join($crepes, $lemonJuice, $sugar, function(crepes, lemonJuice, sugar) {
        ...
    });

    $lemonCrepe.then(eat);

And, actually, even a simple `then()` can be expressed this way:

    let $flourAndEggsMixture = join($ingredients, [flour, eggs, milk] => whisk([flour, eggs]));

So, we finally have no callbacks nor `then()`, but still use promises!


A bit of analysis
-----------------

What we finally reach is the most idiomatically asynchronous code that we can. We may switch the order of almost all lines of code without consequences! Indeed, what the code does is to **build an execution graph**, but not actually executing it until the ingredients are fetched. We describe pipes in which data will eventually flow.

This is incredibly powerful and make it very easy to do things in parallel: just express that they don't rely on each other and they will naturally be concurrent, if and when needed.

It is so comfortable that this kind of mechanism is at the heart of functional languages such as [Haskell](https://wiki.haskell.org/Monad), or libraries for optimized computing like [Theano](http://deeplearning.net/software/theano/), notably used for Deep Learning. In Theano, you first describe a computing graph, without executing it, so that it can be automatically optimized or differentiated before being ran.

The reminder of this article shows a bit how many things can be simply fitted into this framework.


Module system
-------------

Bonus: with this design pattern, you get a [requiring system](http://requirejs.org/) almost for free.

    var require = join;  // It's just the same!

    // Initiate module loading in, say, loadModules.js
    $module1 = fetch(...);
    $module2 = fetch(...);
    $module3 = fetch(...);
    $module4 = fetch(...);
    // ...

    // Then, in whatever.js, declare the dependencies.
    require($module1, $module2, function(module1, module2) {
        // ...
    });

To go further, you may want to wrap whatever is needed to load and cache modules into a `loadModule()` function returning a promise, so that you'll get something like:

    require(loadModule('module1'), loadModule('module2'), function(module1, module2) {
        // ...
    });

Now, maybe we'd like the `loadModule()` to be automatically added. No problem, let's wrap this up:

    function requireModules() {
        // Apply loadModule() to all the arguments, except the last one (the callback)
        let args = Array.prototype.map.call(arguments, function(value, i) {
            return i < arguments.length - 1 ? loadModule(value) : value;
        });

        // Then feed them to the original require/join function
        return join.apply(this, args);
    }

    requireModules('module1', 'module2', function(module1, module2) {
        // ...
    });


Plug-ins
--------

Instead of wrapping `loadModule()` into the `require()` function, we may want to plug any intermediate function. So let's build a higher-order function (i.e. a function returning a function) that *decorates* the `join()` with anything:
    
    function decoratedJoin(decorator) {
        // MIT licensed, copyright (c) 2017 -- Elie Michel <elie.michel@exppad.com>
        return function() {
            let args = Array.prototype.map.call(arguments, function(value, i) {
                return i < arguments.length - 1 ? decorator(value) : value;
            });
            return join.apply(this, args);
        }
    }

This makes the custom require function really easy to build:

    var requireModules = decoratedJoin(loadModule);

And makes a many more things easy to build. Let's say we want to check that our ingredients are fresh before using them. We can write a decorator for this:

    function ensureFresh($ingredient) {
        return join($ingredient, function(ingredient) {
            return ingredient.isFresh() ? $ingredient : Promise.reject();
        });
    }

    freshJoin = decoratedJoin(ensureFresh);

    freshJoin($milk, $eggs, function(milk, eggs) {
        // Here we are sure that milk and eggs are fresh!
    });

**Note:** The decorator, `ensureFresh()`, must return a promise!

Of course, you can still reuse the result of a decorated `join()` as a promise:

    let $dryMojito = freshJoin($mint, $greenLemon, $whiteRhum, $ice, shake);
    join($dryMojito, drink);


A more practical example
------------------------

The context in which I was looking for this mechanism was not about cooking crêpes, but a WebGL application, in which one needs:

 * to initialize the OpenGL context;
 * to load heavy remote resources, depending on each other (we need the scene description to know which textures to fetch, etc.);
 * to compile shaders;
 * to run a rendering loop;
 * to run an event loop;
 * and so forth.

The game of dependencies and potentially parallelizable operations is intricated, and the optimization can really make a noticeable difference. I won't put here a full code, but rather give elements of the main code, which links all the units together.

First, we want to wait for some events. Promises to nothing can be used for this:

    let $ready = new Promise(resolve => {
        document.addEventListener("DOMContentLoaded", resolve);
    });

    let $context = join($ready, function() {
        // This functions gets no argument, it is just called when the document becomes ready.
        // ...
    });

Then, I wrote the init steps from the end, wondering "what do I need to render a frame?".

    join($context, $gl, $object, $shaderProgram, function(context, gl, object, shaderProgram) {
        gl.useProgram(shaderProgram);
        gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
        gl.uniform2f(gl.getUniformLocation(shaderProgram, "iResolution"), context.canvas.width, context.canvas.height);
        gl.bindBuffer(gl.ARRAY_BUFFER, object.buffer);
        gl.drawArrays(gl.TRIANGLE_STRIP, 0, object.size);
    }

Ok, so I need promises to the `gl` API, the `object` to render, and the `shaderProgram`. This required me to write some joins before:

    let $gl = join($context, context => initWebGL(context.canvas));

    let $shaderProgram = join($gl, function(gl) {
        return getShaderProgram(gl, ["vertex-shader", "fragment-shader"]);
    });

    let $object = join($context, context => RessourceManager.get(context.sceneUri).asJson());

But I also needed to wait for the object's textures to be correctly loaded and bound, though I don't need any value or handle about textures in the rendering loop. I had to introduce an event `allTexturesReady`, and decided to prefix events promises, which return no value, by an underscore `_` instead of a dollar `$`:

    let $texture1 = join($object, function(object) {
        return object.loadTexture();
    });

    let _texture1Ready = join($gl, $shaderProgram, $texture1, function(gl, shaderProgram, texture1) {
        gl.useProgram(shaderProgram);

        gl.activeTexture(gl.TEXTURE0);
        texture1.bind();
        gl.uniform1i(gl.getUniformLocation(shaderProgram, "texture1"), 0);
    });

    // [...]

    let _allTexturesReady = Promise.all([_texture1Ready, _texture2Ready]);

Then, instead of modifying the `join()` responsible for the rendering by adding `_allTexturesReady` as a dependency, I made a difference between the *definition* of the rendering and its *execution*. So a first `join()` returns a promise to a rendering function, which is then executed when the event `_allTexturesReady` (and potentially others) is fired:

    // Yes, this is a promise to a function
    let $render = join($context, $gl, $object, $shaderProgram, function(context, gl, object, shaderProgram) {
        return function() {
            gl.useProgram(shaderProgram);
            gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
            gl.uniform2f(gl.getUniformLocation(shaderProgram, "iResolution"), context.canvas.width, context.canvas.height);
            gl.bindBuffer(gl.ARRAY_BUFFER, object.buffer);
            gl.drawArrays(gl.TRIANGLE_STRIP, 0, object.size);
        }
    }

    join($render, _allTexturesReady, function(render) {
        render();
        setInterval(render, 15);  // Even start the rendering loop
    })

The complete code is not ready to be released yet, but I hope that these excerpts convinced you that the `join()` pattern can really make a difference when optimizing applications with critical dependency orders.


Appendix A: Code repository
---------------------------

I put the small code of `join()` and `decoratedJoin()` on GitHub: [eliemichel/join.js](https://github.com/eliemichel/join.js). Feel free to suggest improvements, typo fixes and tooling, or to just use the bug tracker to comment on this!


Appendix B: Complete source code
--------------------------------

    // (Units are expressed in the International System)
    let n = 12;  // For 12 hungry men -- adapt to your case
    let flavor = pickOne(['rhum', 'vanilla', 'beer', 'orange flower'] + whatever);
    let $ingredients = fetchIngredients({
        'flour': 100.0 * n * 1e-3,
        'eggs': n,
        'milk': n/12.0 * 1e-3,
        'water': n/12.0 * 1e-3,
        'salted butter': estimate('some for the batter plus what you\'ll need to oil the pan'),
        flavor: estimate()
    });

    let $tools = fetchKitchenware('bowl', 'crêpière (large flat pan with low borders)');

    let $bowlWithFlour = join($ingredients, $tools, function([flour, _], [bowl, _]) {
        bowl.shed(flour);
        bowl.digALittleHollowInTheMiddle();
        return bowl;
    });

    let $eggsContent = join($ingredients, [flour, eggs, _] => eggs.open());
    let $meltedButter = join($ingredients, [flour, eggs, milk, water, butter, flavor] => some(butter).melt());
    let $hotPan = join($tools, [bowl, pan] => pan.heat());
    let $hotOiledPan = join($ingredients, $hotPan, ([flour, eggs, milk, water, butter, flavor], pan) => pan.oil(butter));

    let $batter = join($eggsContent, $bowlWithFlour, function(eggs, bowl) {
        bowl.shed(eggs);
        return bowl.whisk();
    });

    $batter = join($ingredients, $batter, function([flour, eggs, milk, water, _], batter) {
        return batter.graduallyAdd(milk, water);
    });

    $batter = join($ingredients, $meltedButter, $batter, function(ingredients, meltedButter, batter) {
        let [flour, eggs, milk, water, butter, flavor] = ingredients;
        return batter.graduallyAdd(flavor, meltedButter);
    });

    let _waitABit = new Promise(resolve =>
        join($batter, function() {
            setTimeout(resolve, 1000 * 3600 * 2);
        });
        join(_hunger, resolve);
    );

    let $crepes = join($batter, $hotOiledPan, _waitABit, (batter, pan) => pan.cook(batter));

    join($crepes, eat);


(Yes, I could have used [object matching](http://es6-features.org/#ObjectMatchingShorthandNotation).)


Legal mentions
--------------

Crêpes can be very thin. Add some water or milk if they are not enough. The first crêpe is always a mess, because the pan was not ready (likely not hot enough -- it has to be very hot). Don't bother the number of eggs. There are as many right proportions as there are people making crêpes, rangin from *no eggs at all* (yes, it does work) from *almost an omelette* (well no, you got too far at this point). Flavor the batter with rhum, vanilla, beer, orange flower, whatever (but only one out of this list).

And please, stop using those cups/spoons/oz/whatever that definitely don't sum up correctly. You just need liters and grammes. And in case you feel lost, get yourself the one and only measuring cup you need to cook with correct units.

Note to writers: As I was writing this article, I found recipes to be a very good example for programming examples! Much better than I though when just wanting to make my example a bit funnier than `doSomething()`. You have to do some things before others, some at the same time, you need to wait, to check for the cooking, etc. Plus, I suspect it to be a very good vector of soft power, in case you plan to impose your regional customs and food tradition to the world! I'm not originated from French Bretagne, though, but, you know, I do like crêpes anyway. Who doesn't?
