---
layout: post
title: 'Reason #244 to Hate Javascript'
published: true
---

Generate an RFC 4122 compliant UUID: Ruby

{% highlight ruby %}
require 'securerandom' # at worst, if you're using Rails you won't have to write this
SecureRandom.uuid
{% endhighlight %}

Generate an RFC 4122 compliant UUID: Javascript. It's not actually apart of the language itself or a standard lib equivalent. Instead, you need to decide on which of about a half dozen npm packages you want to use. Or you can read through this stupidly long [Stack Overflow post](http://stackoverflow.com/questions/105034/create-guid-uuid-in-javascript).

Let's say, you're like me and you settle on [broofa's](http://stackoverflow.com/users/109538/broofa) despite quickly reading through a few different comments explaining what some of the problems with that approach might be. His was by far the most readable. Anyway, you get:

{% highlight javascript %}
const uuid() = function() {
  // stolen from: http://stackoverflow.com/a/2117523/1947079
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
    var r = Math.random() * 16 | 0, v = c == 'x' ? r : (r&0x3 | 0x8);
    return v.toString(16);
  });
}
{% endhighlight %}

This sucks, I don't want to be responsible for generating UUIDs. Or making sure they adhere to the RFC. (Or even understanding it.) Or making sure they're performant. Or make my team read that code. I want my language to handle stuff like that 😡
