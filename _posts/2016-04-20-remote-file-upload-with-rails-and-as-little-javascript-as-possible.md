---
layout: post
title: Remote File Upload with Rails
published: false
---

*(And as Little Javascript as Possible)*

Let's say you're creating an application that allows Hearthstone players to keep track of the decks they have built. Now you could imagine a subset of the database might look something like this:

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

    <%= hidden_field_tag(:parent_id, "import_form_container") %>
    <%= hidden_field_tag(:form_id, "import_form") %>

    <%= f.hidden_field :user_id, value: current_user..id %>

    <%= f.label :file, "Select a deck file" %>
    <%= f.file_field :file  %>

    <%= f.label :name, "Name your deck" %>
    <%= f.text_field :file %>
  <% end %>
</div>
{% endhighlight %}

The first thing you might notice are our hidden fields. We'll come back to those, but just know that we're going to utilize those fields in some Javascript that we render server-side.

Out-of-the-box, jQuery and Rails aren't able to accept file uploads in a remote form. However, we can get around this by including the [remotipart](https://github.com/JangoSteve/remotipart) gem. That's as simple as adding the gem to your Gemfile.

With that the next step is to add an action to our controller that will accept the form and present the user with a page to review. Here's what that might look like:

{% highlight ruby %}
# take the file they uploaded and save it to a tmp file so we can use it later
# without needing them to submit again due to browser security
path = File.join(
  Rails.root,
  "tmp/#{SecureRandom.urlsafe_base64}_#{params[:importer][:file].original_filename}"
)
File.open(path, "w") do |file| 
  file.write(File.read(params[:watched_item_import][:file].path))
end

# edit the params so we save the right thing
params[:importer][:file] = path

@parent_id = params[:parent_id]
@form_id = params[:form_id]

@import = Importer.new(params[:importer])

# like our typical ActiveRecord models, we use a valid? method to collect any
# errors on the object. we call this to collect those errors, but we don't
# need the typical redirection if it's invalid because we'll handle that on
# the review page
@import.valid?

respond_to do |format|
  format.js
end
{% endhighlight %}

I think at this point it's worth showing you what our `Importer` object might look like:

{% highlight ruby %}
class Importer
  include ActiveModel::Model

  attr_accessor :name, :user_id
  attr_reader :file

  def initialize(attributes={})
    attributes.each { |name, value| send("#{name}=", value) }
  end

  def file=(value)
    @file = File.new(value, "r")
  end

  def persisted?
    false
  end

  def valid?
    # check for errors, add them with errors.add(:base, "error message")
  end

  def save
    # create the records that should be created from the user's form submission
  end
end
{% endhighlight %}

That's a bare-bones form object for the form we created above. We have an initializer that we can pass the hash of inputs from the `params` object that calls each setter we defined with the `attr_accessor` definition. We did define the `file=` setter ourselves so we could go ahead and turn the filepath string we created in the `review` action into a `File` object. I've omitted the `valid?` and `save` methods for brevity.

At this point, to complete the `review` action we just need to generate the Javascript Rails will respond with.

{% highlight javascript %}
// hide the whole form
$("#<%= @parent_id %>").hide();

// normally browser file security clears the file_field so we delete it and
// add a hidden field with the path of the temp file created in the review action
$("#importer_file").remove();
$('<input>').attr({
    type: "hidden",
    id: "importer_file",
    value: "<%= @import.file.path %>",
    name: "importer[file]"
}).appendTo("#<%= @form_id %>");

// stop submitting the form remotely
$("#<%= @form_id %>").removeAttr("data-remote");
$("#<%= @form_id %>").removeData("remote");

// finally, render the view for the user to review
$("#<%= @parent_id %>").after('<%= escape_javascript(render(partial: "review")) %>');
{% endhighlight %}

TODO - explain review.js.erb
     - explain _review.html.erb
     - explain links that submit the form with either the cancel or create actions