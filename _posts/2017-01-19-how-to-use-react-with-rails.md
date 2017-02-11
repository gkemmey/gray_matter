---
layout: post
title: 'How to Use React with Rails'
published: true
---

### First Things First; What Not to Do

A simple search for "react + rails" will turn up the [react-rails](https://github.com/reactjs/react-rails) gem immediately. It's a first party gem you can drop into your rails app and get going immediately. It does a couple things:

1. It vendors React and a custom React UJS library for use with the Rails asset pipeline.

2. Adds a sprockets processor for handling JSX and ES6.

3. Adds a couple Rails helpers for rendering react components in our controllers and views.

With virtually 0 setup, you're able to get started with React right away, and you can make awesome components like this:

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

### So What Should We Do?

So we know we need to use npm because there isn't a gem for a good portion of libraries we may want to use. But how do we make that work with Rails? The answer is to use sprockets in conjunction with webpack and npm.

We don't want to ditch sprockets, or even gem bundled assets, completely. Rails ❤️ sprockets. The asset pipeline has great support for things like bootstrap, jquery, tubolinks, asset hashing--why throw all that away? So what we can do is use webpack to create a bundle of _some_ of our assets, and pass that bundle to our asset pipeline.

I built a version of the [TodoMVC](http://todomvc.com/) app using Rails, React, and Redux that you can see [running here](https://gray-matter-todomvc.herokuapp.com/) and the [code here](https://github.com/gkemmey/todomvc_react_rails).

#### Get Your System Ready

Since we're going to use node modules and webpack, we need to install the tool for that. I'm using [yarn](https://yarnpkg.com/en/) as opposed to npm, and it's what I'd recommend. Anyway, installing node and yarn is pretty simple with homebrew:

```
brew install node
brew install yarn
```

And now we need to use yarn to install webpack:

```
yarn global add webpack@1.14.0
```

And depending on your homebrew setup, you may need to adjust your `.bash_profile` to add packages yarn installed globally to your path. To test if you have the problem run `webpack -h`, if you get an error about webpack not being installed, you need to add this to your `.bas_profile`:

{% highlight bash %}
# -------- YARN --------
# i installed this with homebrew, and now i gotta add where it installs global packages to my path
#
if hash yarn 2>/dev/null; then # check if there's a yarn command to run
  export PATH="$PATH:$(yarn global bin)"
fi
{% endhighlight %}

Lastly, add our development and application dependencies with yarn:

```
yarn add --dev babel-core babel-loader babel-polyfill babel-preset-es2015 \
               babel-preset-react babel-preset-stage-2 expose-loader script-loader

yarn add react react-dom react-redux redux whatwg-fetch
```

#### The Directory Structure

As a reference, here's a look at what my `assets/javascripts` directory looks like:

![Directory Structure]({{ site.github.url}}/public/images/2017-01-19/react_asset_directory_structure.png)

The biggest difference is we've added a `webpack` subfolder to put our javascript files we want processed by webpack and a special `root_components` folder for use with React UJS (more on that later). We also have an `application_entry.js` file for webpack, which we'll look at later. And the bundle files webpack creates.

#### application.js

{% highlight javascript %}
//= require jquery
//= require jquery_ujs
//= require turbolinks
//= require cable

//= require react_ujs
//= require application_bundle
//= require fetch_bundle
{% endhighlight %}

Pretty normal, but hey, what's `react_ujs` still doing there?! We're still going to use it! In fact, we still install the `react-rails` gem. This gives us access to the `render_component` helper, and  bundled version of the `react_ujs` library we can include in our `application.js`. But we're not using it to include React.

#### webpack.config.js

Webpack takes it's direction from a combination of a config file, and then one or more entry files that config file specifies. Let's take our config file in pieces (you can see the whole thing [here](https://github.com/gkemmey/todomvc_react_rails/blob/master/webpack.config.js)). First, we require the packages we need and define some helper methods at the top:

{% highlight javascript %}
const webpack = require("webpack");
const fs = require("fs");

let camelize = function(string) {
  let almost = string.replace(/(\_\w)/g, matches => matches[1].toUpperCase())

  return almost[0].toUpperCase() + almost.slice(1);
};

let filename = (path) => (path.split("/").pop().split(".").slice(0, -1).join(""));

let allFilesIn = function(dir) {
  var results = [];

  fs.readdirSync(dir).forEach(function(file) {
      let path = dir + '/' + file;
      let stat = fs.statSync(path);

      if(stat && stat.isDirectory()) {
        results = results.concat(allFilesIn(path))
      }
      else {
        results.push(path);
      }
  });

  return results;
};
{% endhighlight %}

Because I'm a rails guy through and through, I prefer snakecase for files and variables, and camelcase for classes. So the `camelize` allows us to pass a string like `'todos_list'` and get back `'TodosList'`. `filename` take a path and gets just the filename without the extension. And `allFilesIn` will get all the files in a folder, recursively. We'll use these functions in the next section:

{% highlight javascript %}
let components = allFilesIn(__dirname + "/app/assets/javascripts/webpack/root_components/").
  map((path) => {
    return { test: require.resolve(path), loader: `expose-loader?${camelize(filename(path))}` }
  });
{% endhighlight %}

This gets all the components in our `root_components` folder, and attaches a loader to them. In this case, the [expose-loader](https://github.com/webpack-contrib/expose-loader).

Lastly, we're ready to generate the config object webpack will use:

{% highlight javascript %}
let config = {
  entry: {
    fetch: 'whatwg-fetch',
    application: __dirname + "/app/assets/javascripts/webpack/application_entry.js"
  },

  output: {
    path: __dirname + "/app/assets/javascripts",
    filename: "[name]_bundle.js"
  },

  module: {
    loaders: components.concat([
      {
        test: /\.(js|jsx)$/,
        include: __dirname + "/app/assets/javascripts/webpack",
        loader: "babel",
        query: {
          presets: ["es2015", "react", "stage-2"]
        }
      }
    ])
  }
};

module.exports = config;
{% endhighlight %}

We add two entry files: one for the fetch polyfill, stolen directly from [their documentation](https://github.com/github/fetch), and one for application. We specify where we want our bundled files outputted with the `output` object. And then we define our loaders. This is just an array of objects that represent a loader, at a minimum they have a `test` property which we use to match files to process and a `loader` property, which says which loader to use. So we take our `components` list and concatenate it with the [babel loader](https://github.com/babel/babel-loader). This will run against any of our `.js` files or `.jsx` files under the `include` path, or or `assets/javsacripts/webpack` folder.

#### application_entry.js

Last thing we need to do is define our entry file:

{% highlight javascript %}
require('expose?React!react');
require('expose?ReactDOM!react-dom');

(
  (context) => { context.keys().map(context) }
)(require.context("./root_components", true, /\.(js|jsx)$/));
{% endhighlight %}

The first two lines require and globally expose React and ReactDOM. The last bit uses `require.context` to require all of our `.js` and `.jsx` files in `/root_componets`, which are exposed globally when they're required because of the loader we setup in our config file.

#### Running it All

We can run webpack in development and watch modes with

```
webpack -d --watch --config webpack.config.js
```

This will re-generate our bundle files if we make changes to our javascript files. We needn't mess with any of the hot reloading features of webpack because rails and sprockets will grab those changes in development when we refresh our page. In fact, aside from running the above webpack command, we needn't change our normal Rails development workflow at all.

### Conclusion

We're now able to create React components like the rest of the Javascript world, for example:

{% highlight javascript %}
import React from 'react'
import { connect } from 'react-redux'

import { setFilter, visibleTodos, destroyCompleted } from '../stores/todos_store.js'

let TodosFooter = ({ totalTodos, incompleteTodos, completedTodos, isSelected }) => (
  // jsx stuff here
)

TodosFooter = connect(
  (state, props) => {
    // do some map state to props stuff
  }
)(TodosFooter);

export default TodosFooter;
{% endhighlight %}

But perhaps the best part, we're able to do all that without giving up the things we love about Rails. Or turning our Rails app into strictly an API. We get to use Rails in the ways we love to; mostly serving server-renderd HTML and punting to React if and when it makes sense to. We still get all the gem-ified assets we've grown accostomed to using.

And we get to use React as a first-class citizen, with access to the modern Javascript libraries are using, whether that be Redux or React extensions or testing libraries.

I know webpack is [anything but simple](https://twitter.com/thomasfuchs/status/708675139253174273?lang=en), but hopefully that wasn't too terrible.
