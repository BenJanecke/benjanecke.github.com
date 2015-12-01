---
layout: post
title: "So you want to test your react code with mocha"
date: 2015-11-30 15:45:38 +0200
comments: true
---

So you have happily been testing your code with [jasmine](http://jasmine.github.io/) using something like [karma](http://karma-runner.github.io/) or [jest](https://facebook.github.io/jest/).
But you find yourself running into troubles with async code and getting things working on ci.
Or maybe you just want to get away from needing to have a headless browser installed just to run your tests. Or maybe you just want to get away from headless browsers period. Well then sounds like you might want to consider [mocha](https://mochajs.org/). Skip down to the getting down to brass tacks bit if you are already sold on mocha and want to get going.

## In my mind these are the primary benefits of using [mocha](https://mochajs.org/)

* [Mocha](https://mochajs.org/) is a test runner. That is all it does and it does it well. No built in assertions no built in mocking or stubbing no bloat. One single responsibility.
* Because of the above point the assertion libraries and stubbing libraries are totally up to you giving you an incredible amount of freedom.
* Seeing as [mocha](https://mochajs.org/) runs on the command line* it makes setting up ci much simpler.
* Testing asynchronous code* becomes a breeze using the done callback.
* Headless browsers are a pain JS code should be run in a JS vm ala Node*.
* [Mocha](https://mochajs.org/) is fast and makes it very apparent which tests are the ones slowing down your suite.
* Generating code coverage reports is simple

*1 yes I know it also runs in the browser when you need it to.

*2 Such as promises

*3 This is really just personal preference please don't take it too seriously

## Why not [Jasmine](http://jasmine.github.io/)/[Karma](http://karma-runner.github.io/)/[Jest](https://facebook.github.io/jest/) you rebel scum?

### [Jasmine](http://jasmine.github.io/)

My pain with [jasmine](http://jasmine.github.io/) is that testing Async code is hard(although [jasmine](http://jasmine.github.io/) 2.0 took a page out of [mocha](https://mochajs.org/)'s book and implemented a done() callback *applause* ^.^ that basically murders this point).

My other slightly bigger pain with [jasmine](http://jasmine.github.io/) is that if you want to set up CI you need to start messing around with headless browsers and down that road dragons lie. Enter [Karma](http://karma-runner.github.io/).

### [Karma](http://karma-runner.github.io/)

[Karma](http://karma-runner.github.io/) takes all of the headless browser tomfoolery and makes it simpler, simpler not simple. The biggest pain point with [karma](http://karma-runner.github.io/) is JSX compilation(we are going to get to that in a bit). Finally [karma](http://karma-runner.github.io/) and [jasmine](http://jasmine.github.io/) go hand in hand and we have already established that we want to get away from testing [jasmine](http://jasmine.github.io/) in a headless browser.

I give karma a bad rep, but it really doesn't deserve it. It's a great test runner and you can even run QUnit or Mocha(yes mocha) on it. It makes it great and easy to test on multiple devices and Integrates with many a CI.

But I don't like it because I don't like headless browser testing, and it can be very slow. Deal with it.

### [Jest](https://facebook.github.io/jest/)

[Jest](https://facebook.github.io/jest/)... [Jest](https://facebook.github.io/jest/) is special. Facebook(the folks behind react js) suggest that you use [jest](https://facebook.github.io/jest/) to test react components and I kinda like the idea that jest represents but the implementation leaves me wanting.

Jest is cool because

* JSX compilation out of the box
* Automagic mocking and stubbing

Jest is uncool because

* SLOOOOOOOOW!
* Unintelligible error messages
* Automagic mocking and stubbing is a double edged sword

## Down to brass tacks

Ok so to test React stuff we are going to need the following.

* A fake DOM (to render our comonents in to) ([js-dom](https://github.com/tmpvar/jsdom))
* Some way to compile JSX code in to plain JS ([babel](https://babeljs.io/))
* A way to bind globals like jQuery to "window"
* An assertion library ([chai](http://chaijs.com/))
* A mocking and stubbing library ([sinon](http://sinonjs.org/) & [sinon-chai](https://github.com/domenic/sinon-chai))

### A fake DOM

So for this we are going to use [js-dom](https://github.com/tmpvar/jsdom).

From the js-dom documentation: "A JavaScript implementation of the WHATWG DOM and HTML standards, for use with Node.js." All this means is that you get a DOM to use inside of your node ^.^ yay!.

So what do we need to do to get this working?

We are going to use the "hardcore" api according to the js-dom documentation but really it's rather simple.

{% highlight javascript %}
// get the "hardcore" api
const jsdom = require('jsdom').jsdom;

// Intentionally using a named function here to make stack traces more legible
module.exports = function jsDom(markup) {
  // If document is already defined on the global namespace return because we don't want to create more than one DOM
  if (typeof global.document !== 'undefined') return;
  // pass markup to js-dom to create the fake DOM and bind it to the global document
  global.document = jsdom(markup || '');
  // bind the fake window object to the global namespace
  global.window = global.document.defaultView;
  // set a navigator so libraries like jQuery don't get confused and think they are using a chrome DOM
  global.navigator = {
    userAgent: 'node.js'
  };
};
{% endhighlight %}

Now you can use it in your test like this.

{% highlight javascript %}
require('./js-dom')('<html><body></body></html>');
{% endhighlight %}

And viola you have a fake DOM you can do stuff with.
Here's an example using React testutils.

{% highlight javascript %}
require('./js-dom')('<html><body></body></html>');
const TestUtils = require('react').addons.TestUtils;
const SomeComponent = require('./some-component');
const chai = require('chai');
const expect = chai.expect;

describe('Testing a component', () => {
  it('Should do stuff with the DOM', () => {  
    // Yes I know I'm using JSX here we'll get to making that work in a few seconds
    let component = TestUtils.renderIntoDocument(<SomeComponent />);
    let stuffInsideComponent = TestUtils.scryRenderedDOMComponentsWithClass(
      component,
      'some-class-selector'
    );

    expect(stuffInsideComponent[0].getDOMNode().innerHTML).to.eq('<p class="some-class-selector">I am stuff</p>');
  });
});
{% endhighlight %}

### Compiling JSX to js for mocha

Mocha has this concept of compilers, that was originally introduced to facilitate testing [Coffeescript](http://coffeescript.org/). We are going to use that and make it work for JSX by creating a compiler.js file that uses babel to do compilation.

{% highlight javascript %}
const fs = require('fs');
const babel = require('babel-core');
const originalCode = require.extensions['.js'];

require.extensions['.js'] = function(module, filename) {
  // external code never needs compilation.
  if (filename.indexOf('node_modules/') >= 0) {
    return (originalCode || require.extensions['.js'])(module, filename);
  }
  let content = fs.readFileSync(filename, 'utf8');
  let compiled = babel.transform(content, {
    'filename': filename,
    'presets': [ 'es2015', 'react' ]
  }).code;
  return module._compile(compiled, filename);
};
{% endhighlight %}

Once you have that you can do the following

{% highlight javascript %}
mocha --compilers .:test/compiler.js
{% endhighlight %}

or add --compilers .:test/compiler.js to your mocha.opts.

The little . before the : means that we want to compile all files with this compiler we could change that to be .jsx:test/compiler.js if we only want to compile JSX.

### A way to bind globals like jQuery to "window"

Unfortunately using jQuery is not as simple as npm install jquery and require('jquery'), because the npm version of jquery has all the DOM stuff and ajax removed because "You are in a node enviroment and you don't need that" pfft yes we do. So what we are going to do is the following download the latest distribution of jQuery(preferabbly with bower) and require it as follows in our js-dom.

{% highlight javascript %}
const jsdom = require('jsdom').jsdom;

module.exports = function jsDom(markup) {
  if (typeof document !== 'undefined') return;
  global.document = jsdom(markup || '');
  global.window = global.document.defaultView;
  global.navigator = {
    userAgent: 'node.js'
  };
  // yes we want it on both global and window deal with it.
  global.jQuery = require('../vendor/jquery/dist/jquery-2.1.4.min');
  global.$ = global.jQuery;
  global.window.jQuery = global.jQuery;
  global.window.$ = global.jQuery;
};
{% endhighlight %}

And now you have jQuery go forth and $.

But what about plugins? That's simple just require() them after you included jQuery

{% highlight javascript %}
const jsdom = require('jsdom').jsdom;

module.exports = function jsDom(markup) {
  if (typeof document !== 'undefined') return;
  global.document = jsdom(markup || '');
  global.window = global.document.defaultView;
  global.navigator = {
    userAgent: 'node.js'
  };
  // yes we want it on both global and window deal with it.
  global.jQuery = require('../vendor/jquery/dist/jquery-2.1.4.min');
  global.$ = global.jQuery;
  global.window.jQuery = global.jQuery;
  global.window.$ = global.jQuery;
  // This will bind to jQuery
  require('../app/vendor/daterangepicker/dist/daterangepicker.js');
};
{% endhighlight %}

## Finally

So it really just takes two things to get set up.

* A fake dom with js-dom
* A compiler for jsx/es2015

And you are golden .

Thanks for reading ^.^
