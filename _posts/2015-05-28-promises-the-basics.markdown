---
layout: post
title: "Promises the basics"
date: 2015-05-28 15:45:38 +0200
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


{% endhighlight %}

### The Basics

Promises are pretty cool things.

They allow you to take code like this.

{% highlight javascript %}
fetchPeople((people) => {
  getNames(people, (names) => {
    upcase(names, (upcasedNames) => {
      display(upcasedNames);
    });
  });
});
{% endhighlight %}

And turn it into this

{% highlight javascript %}
  fetchPeople
  .then(getNames)
  .then(upcase)
  .then(display);
{% endhighlight %}

Pretty nice right.

Lets take a look at how we can create a promise.

{% highlight javascript %}
let getNames = (people) {
  return new Promise((resolve) => {
    let names = people.map((person) => person.name);
    resolve(names);
  });
};
{% endhighlight %}

Now we can use our promise.

{% highlight javascript %}
getNames([{
  "name": "Steve",
  "age": 22
}, {
  "name": "Bob",
  "age": 33
}]).then((names) => {
  console.log(names); // [ 'Steve', 'Bob' ]
});
{% endhighlight %}

or when we make it break.

{% highlight javascript %}

getNames(undefined, (names) => {
  // this code is never reached
}).catch((e) => {
  console.log(e); // TypeError: Cannot read property 'map' of undefined
});

{% endhighlight %}

There is a lot going on there so lets break it down.

### Resolve

Calling resolve tells the promise that it is executed successfully

{% highlight javascript %}

let fetchPeople = (people) => {
  return new Promise((resolve) => {
    db.find('people', (people) => {
      // people have been fetched so we can resolve
      resolve(people);
    });
  });
};

{% endhighlight %}

It's like calling a callback.

{% highlight javascript %}

let fetchPeople = (done) => {
  db.find('people', (people) => {
    // people have been fetched so we can call the callback
    done(people);
  });
};

{% endhighlight %}


### Then

Then gets called when the promise is finished executing successfully.

{% highlight javascript %}

fetchPeople().then((people) => {
  // I now have people
});

{% endhighlight %}

### Catch

Catch is what makes promises especially powerful it allows you to use exceptions in asynchronous code.

Traditional callbacks suffer from clunky exception handling.

If an exception happens in the following code.

{% highlight javascript %}

let getNames = (people, done) => {
  done(people.map((person) => person.name));
};

{% endhighlight %}

We can't catch it

{% highlight javascript %}

try {
  getNames(undefined, (names) => {
    // this is never reached
  });
} catch (e) {
  // this is never reached either
}
// even though a TypeError: Cannot read property 'map' of undefined exception gets thrown

{% endhighlight %}

So we have to move the try catch code into the getNames function.

{% highlight javascript %}

let getNames = (people, done) => {
  try {
    done(null, people.map((person) => person.name));
  } catch (e) {
    done(e, null);
  }
};

{% endhighlight %}

So now we can.

{% highlight javascript %}

getNames(undefined, (err, names) => {
  if (err) {
    // deal with the error
  } else {
    // do something with names
  }
});

{% endhighlight %}

This is not exactly ideal as we are essentially duplicating the try catch logic.

What's even worse is that if we want to do exception handling within the callback we have to add an additional try catch.

{% highlight javascript %}

getNames(undefined, (err, names) => {
  if (err) {
    // deal with the error
  } else {
    try {
      // do something with names
    } catch (e) {
      // deal with something that went wrong with names
    }
  }
});

{% endhighlight %}

Compare this to a promise based version of getNames.

{% highlight javascript %}

let getNames = (people) => {
  return new Promise((resolve) => {
    resolve(people.map((person) => person.name));
  });
};

getNames(undefined).then((names) => {
  // I never get called
}).then((e) => {
  // But I do
  console.log(e); // TypeError: Cannot read property 'map' of undefined
});

getNames([{
  "name": "Steve",
  "age": 22
}, {
  "name": "Bob",
  "age": 33
}]).then((names) => {
  names.someUndefinedFunction();
}).catch((e) => {
  console.log(e); // TypeError: [ "Steve", "Bob" ].someUndefinedFunction is not a function
});

{% endhighlight %}

That's it for the basics keep an eye out for a follow up post on promise patterns.