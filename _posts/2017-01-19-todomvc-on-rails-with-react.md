---
layout: post
title: 'TodoMVC on Rails with React'
published: true
---

If you're anything like me, you found Ruby and Rails a while back, and at this point it'll be a cold day in hell before you write Java code again. But you've also come to learn there's an unfortunate need to be able to write some Javascript every now and then.

Enter React. If you haven't heard of React, it's one of the web technologies currently eating lunch at the cool kid table. Rails isn't anymore, and it isn't exactly obvious how we get these two to play nicely. But here's a few things that didn't work so well and few that do.

### Cool Kids Can Tell When You're Faking It

A simple search for "react + rails" will turn up the [react-rails](https://github.com/reactjs/react-rails) gem immediately. It's a first party gem you can drop into your rails app and get going immediately. It does a couple things:

1. It vendors React and a custom React UJS library for use with the Rails asset pipeline.

2. Adds a sprockets processor for handling JSX and ES6.

3. Adds a couple Rails helpers for rendering react components in our controllers and views.

With virtually 0 setup, you're able to get started with React right away, and you make awesome components like this:

{% highlight javascript %}
class MyComponent extends React.Component
  render() {
    return (
      <p>stuffs</p>
    )
  }
}
{% endhighlight %}

And that works for a while, until you want to start writing Javascript like the rest of the world, something like:

{% highlight javascript %}
import { DopeLibrary } from 'some_awesome_npm_thing'

class MyComponent extends React.Component
  // ...
}

export default MyComponent;
{% endhighlight %}

Sssssshhhhhhhiiiiiiitttttt...how do I import npm libraries...? 🤔 Answer: you have to to use npm.

`react-rails` isn't all bad, though. Remember that React UJS library I mentioned? Turns out that thing is pretty sweet. It lets us write code like this:

{% highlight erb %}
<div>
  <p>normal server rendered html stuffs</p>
</div>

<%= react_component 'MyComponent', props, html_options %>
{% endhighlight %}

This works by rendering a `<div>` with a bunch of data attributes that represents the component to mount and the props to pass to it. Then the React UJS library listens for the appropriate turbolinks and/or `$(document).ready` equivalents, and uses ReactDOM to render our component at that time.

The upside to this, we can easily mix our normal Rails views with our React components. The downside, React UJS expects React, ReactDOM, and any components rendered with `react_component` to be defined globally.
