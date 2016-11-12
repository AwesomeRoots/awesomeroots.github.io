---
published: true
title: Shaking blocks and scrolling of resizing content
layout: post
---
Just solved a tricky thing: imagine you have a page of two blocks. First block can grow its height on click, but you wish the second block's top line to stay at its place. Hence, all the page should scroll down to keep the block on its place.

Sounds tricky? Just check the block below and click "expand". Use "reset" to reset the example.

<iframe width="100%" height="160" frameborder ="no" src="http://workisfun.net/content/blog/smooth-scroll/1.html"></iframe>

You see this ugly shaking blue box when it's scrolling? That's the issue.

To make it smooth, let's hack it. Remember, we want the position of the blue box to be fixed during the whole scrolling action? So let's make it fixed!

```js
FixableBlock = function (o) {
    var t = $(o),
        params = {};

    t.extend({
        fix: function () {
            // store parameters
            params = {
                top: t.offset().top,
                left: t.offset().left,
                position: t.css('position')
            };
            // apply fixed position
            t.css({
                top: t.offset().top - $(window).scrollTop(),
                left: t.offset().left,
                position: 'fixed'
            });
        },
        release: function () {
            t.css(params);
            params = {};
        }
    });

    return t;
};
```

Let's apply FixableBlock class on the blue box and see what happens.

<iframe width="100%" height="160" frameborder ="no" src="http://workisfun.net/content/blog/smooth-scroll/2.html"></iframe>

Hmmm... Oh yes, it's fixed. We should explicitely preserve dimensions of the blue box, or it collapses.

```js
FixableBlock = function (o) {
    var t = $(o),
        params = {};

    t.extend({
        fix: function () {
            // store parameters
            params = {
                top: t.offset().top,
                left: t.offset().left,
                position: t.css('position')
            };
            // apply fixed position and preserve the dimensions
            t.css({
                top: t.offset().top - $(window).scrollTop(),
                left: t.offset().left,
                width: t.css('width'),
                height: t.css('height'),
                position: 'fixed'
            });
        },
        release: function () {
            t.css(params);
            params = {};
        }
    });

    return t;
};
```

Ops.

When we make the blue block's position `fixed`, it does not affect the whole document's height and width anymore. Moreover, other blocks (like yellow one) can use its place now.

Let's fix that by creating a transparent block with exactly the same dimensions as blue one has, and have it hold the place for our main hero. That should solve the problem.

So here's our final version:

```js
FixableBlock = function (o) {
    var t = $(o),
        params = {},
        placeholder;

    t.extend({
        fix: function () {
            // store parameters
            params = {
                top: t.offset().top,
                left: t.offset().left,
                position: t.css('position')
            };
            // create placeholder
            placeholder = $('<div>');
            placeholder.css(t.css(['width', 'height']));
            // apply fixed position and preserve the dimensions
            t.css({
                top: t.offset().top - $(window).scrollTop(),
                left: t.offset().left,
                width: t.css('width'),
                height: t.css('height'),
                position: 'fixed'
            });
            placeholder.insertBefore(t);
        },
        release: function () {
            placeholder.remove();
            delete placeholder;

            t.css(params);
            params = {};
        }
    });

    return t;
};
```

<iframe width="100%" height="160" frameborder ="no" src="http://workisfun.net/content/blog/smooth-scroll/4.html"></iframe>

That's it. Of course, the solution can (and should) be tweaked: for example, this simple thing will be buggy if initially window is higher than two blocks. Just tune it for your page.
