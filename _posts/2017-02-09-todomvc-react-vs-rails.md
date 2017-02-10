---
layout: post
title: 'TodoMVC on React vs Rails'
published: false
---

```
>> find app -type f -exec wc -l {} \; | sort -rn
     213 app/assets/javascripts/webpack/stores/todos_store.js
      86 app/assets/javascripts/webpack/todos/todo_container.jsx
      78 app/controllers/todos_controller.rb
      72 app/assets/javascripts/webpack/todos/todos_footer.jsx
      44 app/assets/javascripts/webpack/todos/toggle_all.jsx
      42 app/assets/javascripts/webpack/root_components/todos_container.jsx
      40 app/assets/javascripts/webpack/todos/new_todo.jsx
      28 app/assets/javascripts/webpack/todos/todos_list.jsx
      22 app/views/layouts/application.html.erb
      20 app/assets/stylesheets/application.css
      20 app/assets/javascripts/application.js
      15 app/controllers/application_controller.rb
      10 app/models/todo.rb
      10 app/assets/javascripts/application_entry.js
       7 app/assets/javascripts/webpack/helpers/async_helpers.js
       1 app/views/todos/index.html.erb
```

```
>> find app -type f -exec wc -l {} \; | sort -rn
      62 app/views/todos/index.html.erb
      60 app/controllers/todos_controller.rb
      30 app/assets/javascripts/todos.coffee
      26 app/views/layouts/application.html.erb
      20 app/assets/stylesheets/application.css
      16 app/assets/javascripts/application.js
      15 app/controllers/application_controller.rb
       7 app/models/todo.rb
```
