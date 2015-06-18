---
layout: post
title: "Testing promises with mocha"
date: 2015-06-16 15:45:38 +0200
comments: true
---

### Before we begin

This post uses es6 syntax

{% highlight javascript %}
// This
var add = function (x) {
  return x + 1;
};

// Is the same as this in es6
let add = (x) => x + 1;

// And this
class Person {
  constructor() {
    // construct
  }

  doSomething() {
    // do things
  }
}

// Is the same as this
function Person() {
  // construct
}

Person.prototype = {
  doSomething: function doSomething() {

  }
}

{% endhighlight %}

### Consider the following

{% highlight javascript %}
class Person {
  constructor(name) {
    this._name = name;
  }

  get name() {
    let that = this;

    return new Promise((resolve) => {
      setTimeout(function () {
        resolve(that._name);
      }, 10);
    });
  }
}


describe('Person', () => {
  let person = new Person('Jimmy');

  it('gets a name', () => {
    person.name.then((name) => {
      expect(name).to.eq('James');
    });
  });
});
{% endhighlight %}

you would expect that to fail, but it doesn't.

### Why?

So the it block gets run and waits for exceptions but the problem is that the it block finishes running before the promise gets called.
We need a way to tell the it block that it takes a while to run.

### Introducing done()

Update the above code like this

{% highlight javascript %}
describe('Person', () => {
  let person = new Person('Jimmy');

  it('gets a name', (done) => {
    person.name.then((name) => {
      expect(name).to.eq('James');
      done();
    });
  });
});
{% endhighlight %}

And it will know to wait for exceptions until the done callback gets called.

Will this fail? No, no it wont.

### Why?

Because expect() throws an exception and that exception gets caught by the promise effectively hiding it from the test runner.

### So how do we tell an asynchronous test that we have a failure?

By passing an error to the done() callback

{% highlight javascript %}
describe('Person', () => {
  let person = new Person('Jimmy');

  it('gets a name', (done) => {
    person.name.then((name) => {
      expect(name).to.eq('James');
      done();
    }).catch((e) => {
      done(e);
    });
  });
});
{% endhighlight %}

Or just

{% highlight javascript %}
describe('Person', () => {
  let person = new Person('Jimmy');

  it('gets a name', (done) => {
    person.name.then((name) => {
      expect(name).to.eq('James');
      done();
    }).catch(done);
  });
});
{% endhighlight %}

Will that fail?

Yes finally we have a failure.

### Why?

Because we passed an error to the done callback.

### Let's go over what we learned

* To tell a test it's async we need to have a done callback
* To tell a test it's done we need to call the done callback
* To tell a test it failed we need to call the done callback with an error object



