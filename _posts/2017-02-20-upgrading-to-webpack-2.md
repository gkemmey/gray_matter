---
layout: post
title: 'Upgrading to Webpack 2'
published: true
---

_This is a follow up to [this post]({{ site.github.url }}/2017/01/19/how-to-use-react-with-rails/) where we looked at setting up Webpack with Rails._

Honestly, upgrading was super easy, though we don't have the most complicated of setups. I used the [official migration guide](https://webpack.js.org/guides/migrating/) and went through it step-by-step.

### What got upgraded?

A quick look of what webpack-related dependencies were updated:

{% highlight diff %}
{
  "devDependencies": {
-    "babel-core": "^6.20.0",
+    "babel-core": "^6.23.1",
    "babel-jest": "^18.0.0",
-    "babel-loader": "^6.2.9",
-    "babel-polyfill": "^6.20.0",
-    "babel-preset-es2015": "^6.18.0",
-    "babel-preset-react": "^6.16.0",
-    "babel-preset-stage-2": "^6.18.0",
+    "babel-loader": "^6.3.2",
+    "babel-polyfill": "^6.23.0",
+    "babel-preset-es2015": "^6.22.0",
+    "babel-preset-react": "^6.23.0",
+    "babel-preset-stage-2": "^6.22.0",
    "css-loader": "^0.26.1",
-    "expose-loader": "^0.7.1",
+    "expose-loader": "^0.7.3",
    "jest": "^18.1.0",
    "script-loader": "^0.7.0",
    "style-loader": "^0.13.1",
-    "webpack": "^1.14.0"
+    "webpack": "^2.2.1"
  }
}
{% endhighlight %}

### The new 'module.rules'

In `webpack.config.js` the `loaders` option has been renamed to `rules`. Additionally, the options each loader object takes are slightly different. Two biggest changes here are one, loaders now need the `-loader` suffix both in config and in `require` statements. And two, loader chaining now uses an array and the `use` option, as opposed to the exclamation point.

{% highlight diff %}
module: {
-    loaders: components.concat([
+    rules: components.concat([
    {
      test: /\.(js|jsx)$/,
      include: __dirname + "/app/assets/javascripts/webpack",
-        loader: "babel",
+        loader: "babel-loader",
      query: {
-          presets: ["es2015", "react", "stage-2"]
+          presets: [["es2015", { "modules": false }], "react", "stage-2"]
      }
    },
    {
      test: /\.css$/,
-        loader: 'style-loader!css-loader'
+        use: ['style-loader', 'css-loader']
    }
  ])
}
{% endhighlight %}

### Webpack handles ES2015, CommonJS, and AMD modules

You'll notice the `{ "modules": false }` in the `babel` options above. We now let webpack handle any module symbols, and tell babel not to parse them. You'll need to adjust this in your config or `.babelrc` or `package.json`--wherever you have it.

### Some defaults of the UglifyJsPlugin changed

This only applies to our `webpack.production.config.js`, and we just went ahead and turned back on those defaults by specifying like so:

{% highlight diff %}
-config.plugins.push(new webpack.optimize.UglifyJsPlugin())
+config.plugins.push(new webpack.optimize.UglifyJsPlugin({
+  warnings: true,
+  sourceMap: true
+}))
{% endhighlight %}

### Fix require statements in entry file

Just like in our config file, we need to make sure to use the `-loader` suffix when requiring in our `application_entry.js` file:

{% highlight diff %}
-require('expose?React!react');
-require('expose?ReactDOM!react-dom');
+require('expose-loader?React!react');
+require('expose-loader?ReactDOM!react-dom');
{% endhighlight %}

### Conclusion

And that was it! All in all, not too bad. Again, if you want to know how / why we set things up the way we did, you can see that [post here]({{ site.github.url }}/2017/01/19/how-to-use-react-with-rails/). And, if you want to check the code out yourself, you can do that [here](https://github.com/gkemmey/todomvc_react_rails).
