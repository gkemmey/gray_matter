---
layout: post
title: "Making a Case (long-winded) for Turbolinks"
published: true
---

[Turbolinks](https://github.com/turbolinks/turbolinks) is the coolest technology not nearly enough websites are using. And what I'd like to do, is convince you of that as we build a simple application. But first we need a baseline, so let's party like it's 2008-ish! 🎉

Not even joking a little bit here, this was the nirvana of web development. Browsers sent data to servers and servers sent back HTML. Full, round trips every time. Developers _actually_ rendered `<form>` tags. Sure, sometimes we sent a little sprinkle of JavaScript, but for any real work, we fully expected the users would send requests back to our server. Apps were largely stateless. And life was simple.

For a moment, let's go back to that. Let's build a [TodoMVC](http://todomvc.com/) app like it's 2008!

If you've never seen one of these before, they look like this:

![Todo App]({{ site.github.url}}/public/images/2019-03-07/todo_app.gif)

Ok, so some requirements:

1. Let's be a real app. A lot of (all?) the examples at TodoMVC just write to local storage. How useful is that? So we'll save todos to a database associated with the user's session such that refreshing the page doesn't lose the todos (but they can go away when the user kills their browser).
2. Add a new todo in the box at the top.
3. Complete individual todos.
4. Delete individual todos ("x" at the right of the todo).
5. Complete all showing todos (clicking down chevron at the top).
6. Clear all showing completed todos (link at the bottom).
7. Double click to edit an existing. Saves on blur. Escape cancels the edit.
8. Filter by "All", "Active", and "Completed" (links at the bottom).
9. Display a count of remaining todos out of the todos showing.

So that's not nothing. Let's build it! Using [Express](https://expressjs.com/)...like it's 2008...even though Node didn't come along until 2009...and, uh, Express...well, not until 2010...2008-ish!

### Database

Honestly, gonna gloss right over this. All versions of this app need the same one, and it doesn't really change. Code is linked at the bottom if you want to know more, but in short we'll use [Sequelize](http://docs.sequelizejs.com/), SQLite, and a single `todos` table with `session_user_id`, `title`, `completed` columns. Plus, your typical `id`, `created_at`, and `updated_at` columns, for good measure.

`next()` 👈 Heh, get it? That's an Express joke, albeit a bad one.

### Routes

Being RESTful seems reasonable. Let's put these in a table and think about the path, the HTTP verb we wanna use, and a description of what it does.

| HTTP verb | Path | Description |
| --------- | ---- | ----------- |
| GET       | /[?completed=(true\|false)] | Renders our only page with all the todos. Handles a filter param |
| POST      | /    | Creates a new todo |
| PATCH     | /:id | Updates the todo with `id == :id` |
| DELETE    | /:id | Deletes the todo with `id == :id` |
| PATCH     | /update_many | Updates many todos (we'll send a list of `ids` in the body) |
| DELETE    | /destroy_many | Deletes many todos (we'll send a list of `ids` in the body) |

Alright, let's see some codes:

```js
// app/controller/index.js

var express = require('express')
var router = express.Router()
var models = require("../models")

router.get('/', function(req, res) {
  res.render('index', { ... })
})

router.post('/', function(req, res) {
  // create todo
  res.redirect('/');
})

router.patch('/:id', function(req, res) {
  // update todo
  res.redirect('/');
})

router.delete('/:id', function(req, res) {
  // destroy todo
  res.redirect('/');
})

router.patch('/update_many', function(req, res) {
  // update many todos
  res.redirect('/');
})

router.delete('/destroy_many', function(req, res) {
  // destroy many todos
  res.redirect('/');
})

module.exports = router
```

```js
// app.js
var express = require('express');
var app = express();

app.set('port', (process.env.PORT || 5000));
app.use(require(path.join(__dirname, '/app/controllers')))

app.listen(app.get('port'), function() {
  console.log('Node app is running on port', app.get('port'));
  console.log('Running in '+ app.settings.env)
});
```

We're gonna skip over any robust error handling across all our versions of this app. Otherwise, I think that's pretty reasonable. We have one `GET` endpoint responsible for rendering a view, and five endpoints responsible for performing some action and _redirecting_ to that view.

There's an elegance in that simplicity, in my opinion 😍

### The View

#### Static Assets

We won't have much of an app without some CSS. And we'll need at least a little JavaScript to toggle our todo for editing. So let's tell Express how to handle those assets:

1. Copy [this](http://todomvc.com/examples/react/node_modules/todomvc-app-css/index.css) to `public/css/application.css`. And create an empty `public/js/application.js` file.

2. Add this to `app.js` before `listen`

    ```js
    // app.js
    app.use(express.static(path.join(__dirname, '/public')));
    ```

Cool, now we just gotta add those files to our `<head>`, which we'll get to in the next part.

#### index.ejs

Templating languages were all the rage in 2008. The idea being, we can create a dynamic experience by altering the HTML we send back based on the resources we know exist -- in our case, the todos. I hate that these have fallen out of vogue. Given that the end goal of _all_ web rendering is HTML, these sit right above that goal. Most of the template is HTML and it becomes quite easy to get an idea of the final output just by reading the template file.

Since we're building an Express app, we'll use [EJS](https://ejs.co/) templates. First, we gotta tell Express that that's the plan:

```js
// app.js
app.set('views', path.join(__dirname, '/app/views'));
app.set('view engine', 'ejs');
```

And then create our template. Let's fill in some of the skeleton our final app will render. And we can go ahead and tackle including our static assets in the `<head>` part of our HTML now, too:

```html
<!-- app/views/index.ejs -->

<!DOCTYPE html>
<html lang="en">
  <head>
    <link rel="stylesheet" media="all" href="/css/application.css">
    <script type="text/javascript" src="/js/application.js" defer></script>
  </head>

  <body>
    <section id="todoapp">
      <header id="header">
        <h1>todos</h1>
      </header>

      <section id="main">
        <ul id="todos">
        </ul>
      </section>

      <footer id="footer">
      </footer>
    </section>

    <footer id="info">
      <p>Double-click to edit a todo</p>
      <p>Created by Gray Kemmey</p>
      <p>Part of <a href="http://todomvc.com">TodoMVC</a></p>
    </footer>
  </body>
</html>
```
