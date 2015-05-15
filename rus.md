# ES6 In Depth: Generators

_[ES6 в деталях][1] — это цикл статей о новых возможностях языка
программирования JavaScript, появившихся в 6 редакции стандарта ECMAScript,
кратко — ES6._

Мне не терпится вам всё рассказать. Сегодня мы будем обсуждать самую волшебную
функцональность в ES6.

Что я имел в виду под словом «волшебную»? Во-первых, эта функциональность
настолько отличается от всего того, что уже есть в JS, что поначалу может
показаться колдовством. В том смысле, что она выворачивает обычное поведение
языка наизнанку! Если это не магия, но я не знаю, что это.

Но не только поэтому. Возможности этой фичи по упрощению кода и устранению
«ада коллбеков» граничат со сверхъестественными.

Я излишне нахваливаю? Давайте углубимся, и вы сами рассудите.


## Знакомьтесь, генераторы ES6

Что такое, генераторы?

Начнём с того, что рассмотрим один такой:

    function* quips(name) {
      yield "привет, " + name + "!";
      yield "я надеюсь, вам нравятся статьи";
      if (name.startsWith("X")) {
        yield "как круто, что ваше имя начинается с X, " + name;
      }
      yield "увидимся!";
    }

Это часть кода для [говорящей кошки][2], возможно, самого важного типа
приложений в интернете на сегодняшний день. (Давайте,
[нажмите на ссылку, поиграйте с кошкой][2]. Когда вы окончательно запутаетесь,
возвращайтесь сюда за объяснением.)

Выглядит как-то похоже на функцию, верно? Это называется, _функция-генератор_,
и у неё есть много общего с обычными функциями. Мы вы можете заметить два
отличия уже сейчас:

*   Обычные функции начинаются с `function`. Функции-генераторы начинаются с
    `function*`.

*   Внутри функции-генератора есть ключевое слово `yield` с синтаксисом, похожим
    на `return`. Отличие в том, что функция (даже функция-генератор) может
    вернуть значение только один раз, а функция-генератор может отдать значение
    любое количество раз. Выражение `yield` _приостанавливает выполнение
    генератора, так что его можно позже возобновить_.

Вот, именно в этом самая большая разница между обычными функциями и
функциями-генераторами. Обычные функции не могут поставить себя на паузу.
Функции-генераторы могут.


## What generators do

What happens when you call the `quips()` generator-function?

    > var iter = quips("jorendorff");
      [object Generator]
    > iter.next()
      { value: "hello jorendorff!", done: false }
    > iter.next()
      { value: "i hope you are enjoying the blog posts", done: false }
    > iter.next()
      { value: "see you later!", done: false }
    > iter.next()
      { value: undefined, done: true }

You’re probably very used to ordinary functions and how they behave. When you call them, they start running right away, and they run until they either return or throw. All this is second nature to any JS programmer.

Calling a generator looks just the same: `quips("jorendorff")`. But when you call a generator, it doesn’t start running yet. Instead, it returns a paused _Generator object_ (called `iter` in the example above). You can think of this Generator object as a function call, frozen in time. Specifically, it’s frozen right at the top of the generator-function, just before running its first line of code.

Each time you call the Generator object’s `.next()` method, the function call thaws itself out and runs until it reaches the next `yield` expression.

That’s why each time we called `iter.next()` above, we got a different string value. Those are the values produced by the `yield` expressions in the body of `quips()`.

On the last `iter.next()` call, we finally reached the end of the generator-function, so the `.done` field of the result is `true`. Reaching the end of a function is just like returning `undefined`, and that’s why the `.value` field of the result is `undefined`.

Now might be a good time to go back to [the talking cat demo page][2] and really play around with the code. Try putting a `yield` inside a loop. What happens?

In technical terms, each time a generator yields, its _stack frame_—the local variables, arguments, temporary values, and the current position of execution within the generator body—is removed from the stack. However, the Generator object keeps a reference to (or copy of) this stack frame, so that a later `.next()` call can reactivate it and continue execution.

It’s worth pointing out that **generators are not threads.** In languages with threads, multiple pieces of code can run at the same time, usually leading to race conditions, nondeterminism, and sweet sweet performance. Generators are not like that at all. When a generator runs, it runs in the same thread as the caller. The order of execution is sequential and deterministic, and never concurrent. Unlike system threads, a generator is only ever suspended at points marked by `yield` in its body.

All right. We know what generators are. We’ve seen a generator run, pause itself, then resume execution. Now for the big question. How could this weird ability possibly be useful?


## Generators are iterators

Last week, we saw that ES6 iterators are not just a single built-in class. They’re an extension point of the language. You can create your own iterators just by implementing two methods: `[Symbol.iterator]()` and `.next()`.

But implementing an interface is always at least a little work. Let’s see what an iterator implementation looks like in practice. As an example, let’s make a simple `range` iterator that simply counts up from one number to another, like an old-fashioned C `for (;;)` loop.

    // This should "ding" three times
    for (var value of range(0, 3)) {
      alert("Ding! at floor #" + value);
    }

Here’s one solution, using an ES6 class. (If the `class` syntax is not completely clear, don’t worry—we’ll cover it in a future blog post.)

    class RangeIterator {
      constructor(start, stop) {
        this.value = start;
        this.stop = stop;
      }

      [Symbol.iterator]() { return this; }

      next() {
        var value = this.value;
        if (value < this.stop) {
          this.value++;
          return {done: false, value: value};
        } else {
          return {done: true, value: undefined};
        }
      }
    }

    // Return a new iterator that counts up from 'start' to 'stop'.
    function range(start, stop) {
      return new RangeIterator(start, stop);
    }

[See this code in action.][5]

This is what implementing an iterator is like in [Java][6] or [Swift][7]. It’s not so bad. But it’s not exactly trivial either. Are there any bugs in this code? It’s not easy to say. It looks nothing like the original `for (;;)` loop we are trying to emulate here: the iterator protocol forces us to dismantle the loop.

At this point you might be feeling a little lukewarm toward iterators. They may be great to _use,_ but they seem hard to implement.

It probably wouldn’t occur to you to suggest that we introduce a wild, mindbending new control flow structure to the JS language just to make iterators easier to build. But since we _do_ have generators, can we use them here? Let’s try it:

    function* range(start, stop) {
      for (var i = start; i < stop; i++)
        yield i;
    }

[See this code in action.][8]

The above 4-line generator is a drop-in replacement for the previous 23-line implementation of `range()`, including the entire `RangeIterator` class. This is possible because **generators are iterators.** All generators have a built-in implementation of `.next()` and `[Symbol.iterator]()`. You just write the looping behavior.

Implementing iterators without generators is like being forced to write a long email entirely in the passive voice. When simply saying what you mean is not an option, what you end up saying instead can become quite convoluted. `RangeIterator` is long and weird because it has to describe the functionality of a loop without using loop syntax. Generators are the answer.

How else can we use the ability of generators to act as iterators?

*   **Making any object iterable.** Just write a generator-function that traverses `this`, yielding each value as it goes. Then install that generator-function as the `[Symbol.iterator]` method of the object.

*   **Simplifying array-building functions.** Suppose you have a function that returns an array of results each time it’s called, like this one:

        // Divide the one-dimensional array 'icons'
        // into arrays of length 'rowLength'.
        function splitIntoRows(icons, rowLength) {
          var rows = [];
          var nRows = Math.ceil(icons.length / rowLength);
          for (var i = 0; i < icons.length; i += rowLength) {
            rows.push(icons.slice(i, i + rowLength));
          }
          return rows;
        }

    Generators make this kind of code a bit shorter:

        function* splitIntoRows(icons, rowLength) {
          var nRows = Math.ceil(icons.length / rowLength);
          for (var i = 0; i < icons.length; i += rowLength) {
            yield icons.slice(i, i + rowLength);
          }
        }

    The only difference in behavior is that instead of computing all the results at once and returning an array of them, this returns an iterator, and the results are computed one by one, on demand.

*   **Results of unusual size.** You can’t build an infinite array. But you can return a generator that generates an endless sequence, and each caller can draw from it however many values they need.

*   **Refactoring complex loops.** Do you have a huge ugly function? Would you like to break it into two simpler parts? Generators are a new knife to add to your refactoring toolkit. When you’re facing a complicated loop, you can _factor out the part of the code that produces data_, turning it into a separate generator-function. Then change the loop to say `for (var data of myNewGenerator(args))`.

*   **Tools for working with iterables.** ES6 does _not_ provide an extensive library for filtering, mapping, and generally hacking on arbitrary iterable data sets. But generators are great for building the tools you need with just a few lines of code.

    For example, suppose you need an equivalent of `Array.prototype.filter` that works on DOM NodeLists, not just Arrays. Piece of cake:

        function* filter(test, iterable) {
          for (var item of iterable) {
            if (test(item))
              yield item;
          }
        }

So are generators useful? Sure. They are an astonshingly easy way to implement custom iterators, and iterators are the new standard for data and loops throughout ES6.

But that’s not all generators can do. It may not even turn out to be the most important thing they do.


## Generators and asynchronous code

Here is some JS code I wrote a while back.

              };
            })
          });
        });
      });
    });

Maybe you’ve seen something like this in your own code. [Asynchronous APIs][9] typically require a callback, which means writing an extra anonymous function every time you do something. So if you have a bit of code that does three things, rather than three lines of code, you’re looking at three _indentation levels_ of code.

Here is some more JS code I’ve written:

    }).on('close', function () {
      done(undefined, undefined);
    }).on('error', function (error) {
      done(error);
    });

Asynchronous APIs have error-handling conventions rather than exceptions. Different APIs have different conventions. In most of them, errors are silently dropped by default. In some of them, even ordinary successful completion is dropped by default.

Until now, these problems have simply been the price we pay for asynchronous programming. We have come to accept that asynchronous code just doesn’t look as nice and simple as the corresponding synchronous code.

Generators offer new hope that it doesn’t have to be this way.

[Q.async()][10] is an experimental attempt at using generators with promises to produce async code that resembles the corresponding synchronous code. For example:

    // Synchronous code to make some noise.
    function makeNoise() {
      shake();
      rattle();
      roll();
    }

    // Asynchronous code to make some noise.
    // Returns a Promise object that becomes resolved
    // when we're done making noise.
    function makeNoise_async() {
      return Q.async(function* () {
        yield shake_async();
        yield rattle_async();
        yield roll_async();
      });
    }

The main difference is that the asynchronous version must add the `yield` keyword each place where it calls an asynchronous function.

Adding a wrinkle like an `if` statement or a `try`/`catch` block in the `Q.async` version is exactly like adding it to the plain synchronous version. Compared to other ways of writing async code, this feels a lot less like learning a whole new language.

If you’ve gotten this far, you might enjoy James Long’s [very detailed post on this topic][11].

So generators are pointing the way to a new asynchronous programming model that seems better suited to human brains. This work is ongoing. Among other things, better syntax might help. [A proposal for async functions][12], building on both promises and generators, and taking inspiration from similar features in C#, is [on the table for ES7][13].


## When can I use these crazy things?

On the server, you can use ES6 generators today in io.js (and in Node if you use the `--harmony` command-line option).

In the browser, only Firefox 27+ and Chrome 39+ support ES6 generators so far. To use generators on the web today, you’ll need to use [Babel][14] or [Traceur][15] to translate your ES6 code to Web-friendly ES5.

A few shout-outs to deserving parties: Generators were first implemented in JS by Brendan Eich; his design closely followed [Python generators][16] which were inspired by [Icon][17]. They shipped in Firefox 2.0 [back in 2006][18]. The road to standardization was bumpy, and the syntax and behavior changed a bit along the way. ES6 generators were implemented in both Firefox and Chrome by compiler hacker [Andy Wingo][19]. This work was sponsored by Bloomberg.


## yield;

There is more to say about generators. We didn’t cover the `.throw()` and `.return()` methods, the optional argument to `.next()`, or the `yield*` expression syntax. But I think this post is long and bewildering enough for now. Like generators themselves, we should pause, and take up the rest another time.

But next week, let’s change gears a little. We’ve tackled two deep topics in a row here. Wouldn’t it be great to talk about an ES6 feature that _won’t_ change your life? Something simple and obviously useful? Something that will make you smile? ES6 has a few of those too.

Coming up: a feature that will _plug right in_ to the kind of code you write every day. Please join us next week for a look at ES6 template strings in depth.


 [1]: https://hacks.mozilla.org/category/es6-in-depth/
 [2]: http://people.mozilla.org/~jorendorff/demos/meow.html
 [5]: http://codepen.io/anon/pen/NqGgOQ
 [6]: http://gafter.blogspot.com/2007/07/internal-versus-external-iterators.html
 [7]: https://schani.wordpress.com/2014/06/06/generators-in-swift/
 [8]: http://codepen.io/anon/pen/mJewga
 [9]: http://www.html5rocks.com/en/tutorials/async/deferred/
 [10]: https://github.com/kriskowal/q/tree/v1/examples/async-generators
 [11]: http://jlongster.com/A-Study-on-Solving-Callbacks-with-JavaScript-Generators
 [12]: https://github.com/lukehoban/ecmascript-asyncawait
 [13]: https://github.com/tc39/ecma262
 [14]: http://babeljs.io/
 [15]: https://github.com/google/traceur-compiler#what-is-traceur
 [16]: https://www.python.org/dev/peps/pep-0255/
 [17]: http://www.cs.arizona.edu/icon/
 [18]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/New_in_JavaScript/1.7
 [19]: http://wingolog.org/