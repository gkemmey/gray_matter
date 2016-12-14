---
layout: post
title: 'Reason #137 to Hate Javascript'
published: true
---

Load all files in folder and sub-folders: Ruby

{% highlight ruby %}
Dir.glob(File.join(Rails.root, 'folder/of_interest/', '*.rb')).each do |helper_file|
  require helper_file
end
{% endhighlight %}

Load all files in folder and sub-folders: Javascript

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

let components = allFilesIn(__dirname + "/folder/of_interest/").
  map((path) => {
    return { test: require.resolve(path), loader: `expose-loader?${camelize(filename(path))}` }
  });

let config = {
  entry: {
    application: __dirname + "/app/assets/javascripts/application_entry.js"
  },

  output: {
    path: __dirname + "/app/assets/javascripts"
  },

  module: {
    loaders: components.concat([
      {
        test: /\.(js|jsx)$/,
        include: __dirname + "/folder/of_interest/",
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

But wait, theres more--we still need to require in the entry file:

{% highlight javascript %}
(
  (context) => { context.keys().map(context) }
)(require.context("./folder/of_interest", true, /\.(js|jsx)$/));
{% endhighlight %}

You might notice, we're exposing modules in `/folder/of_interest/` globally and that's part of the complexity here. True. We're doing that to work with the `react_ujs` library that's a part of the [react-rails gem](https://github.com/reactjs/react-rails).
