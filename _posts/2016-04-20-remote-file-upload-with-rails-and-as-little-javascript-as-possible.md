---
layout: post
title: Remote File Upload with Rails
published: false
---

*(And as Little Javascript as Possible)*

Let's say you're creating an application that allows Hearthstone players to keep track of the decks they have built. For those of you who don't know, Hearthstone is a collectible card game that I play more than I probably should.

Now you could imagine a subset of the database might look something like this:

![Database]({{ site.github.url}}/public/images/2016-04-20/database.png)

Now, let's say you'd like to allow a user to upload a file that represents a deck, where each line is a card name followed by the number of times that card is in your deck. That might look something like the following:

{% highlight text %}
Coldlight Oracle x2
Youthful Brewmaster x1
Antique Healbot x2
{% endhighlight %}

Ideally, it would be nice if as part of the importing process the application 

1. Parsed the uploaded file and gives the user a chance to review the actions it's about to take before confirming (or canceling) the import.

2. Shows which entires (lines) the system couldn't parse. In our example, this would be because we can't map the card name like "Coldlight Oracle" to an actual `Card` object.

3. Download any entries we couldn't parse. OK, that's actually a useless feature in this example, but it was important to the client's application that prompted this post, and I think we did something cool to handle this need 😁

*Disclaimer: It wasn't a requirement that the user be able to edit the actions the importer was going to take during the review process. If that's a need, you're probably going to have to do something more complex like saving temporary records to the database.*

So, how would you do all that? First let's create a controller for this importing process:

{% highlight ruby %}
class ImportsController < ApplicationController
  def new
    @import = Importer.new
  end
end
{% endhighlight %}

That `Importer.new` looks kind of strange, but it's pretty common Rails pattern called a [form object](https://robots.thoughtbot.com/activemodel-form-objects). I'm not going to cover those here, though. Other than that, pretty standard. Let's make the form.

{% highlight erb %}
<!-- imports/new.html.erb -->

<div id="import_form_container">
  <%= form_for @import, url: review_imports_path, remote: true,
                        html: { id: "import_form" } do |f| %>

    <%= hidden_field_tag(:parent_id, parent_id) %>
    <%= hidden_field_tag(:form_id, form_id) %>

    <%= f.hidden_field :user_id, value: current_user..id %>

    <%= f.label :file, "Select a deck file" %>
    <%= f.file_field :file  %>

    <%= f.label :name, "Name your deck" %>
    <%= f.text_field :file %>
  <% end %>
</div>
{% endhighlight %}

TODO - time to talk about remotipart gem and our hidde fields