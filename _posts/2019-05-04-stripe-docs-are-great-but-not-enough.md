---
layout: post
title: 'Stripe docs are great, but not enough'
published: true
---

![Stripe Tabs Open]({{ site.github.url }}/public/images/2019-05-04/stripe_tabs_open.png)

{:.caption}
The number of tabs it takes to add Stripe to a website, and I've done it twice before 😪

Integrating Stripe is really hard. Or I guess, knowing if you've done it right is. I understand that's an unfair complaint, because Stripe is so much easier than what came before it, but still 🤷‍♂️

You know that motivational poster that pictures an iceberg cut by the waterline, and the small portion above the water is labeled "success", but a much larger portion below water is labeled things like "late nights", "perseverance", "ridicule", etc.? That's what working with Stripe is like -- "Checkout: Quick Start" above water and "Checkout vs Checkout (beta) vs Stripe.js and Elements", "automated testing", "subscription states", "webhooks" all below. There's a ton of complexity hidden by those "start getting paid in 10 minutes" type tutorials. But don't worry, it's only payment processing 😬

Documentation seems overly focussed on the simplest of workflows. I guess I'd rather have to read a single document for a week that is _explicit_ in pitfalls / scenarios / things you gotta do / has a structure, then read a smattering of scattered docs all showing me just the simplest thing that "works" and have to discover all the other things over the course of many weeks.

"Gray, are you really complaining about Stripes docs?!" A little, yeah. I mean, yes, in many ways they're second to none -- a masterclass in how to do it. But it still feels like some things are missing. For example, the headings under "Checkout" in the documentation sidebar are "Client Quickstart", "Server Quickstart", "Purchase Fulfillment", "Usage with Connect", "Going Live", and "Migration Guide". You know what's not listed? "Updating a user's credit card information". That seems important. [Here](https://stripe.com/docs/recipes/updating-customer-cards) there's a mention that using the old Checkout, you could update card info by not specifying an amount, but I couldn't find an equivalent way to do that in the new Checkout. The API for `redirectToCheckout` seems to suggest a product or plan is required... 🤷‍♂️ There's even a link to the new Checkout on that page now, but it certainly doesn't link to an equivalent recipe. You know what else is missing? A list of recommended Stripe events your webhook(s) should listen for in your average SaaS app. Or even better, a reference implementation for a typical SaaS app!

And documentation isn't the only issue. Testing is basically left entirely as an exercise to the reader. I've used Stripe on three different personal projects. Sometimes my tests have been more rudimentary than I'd like. On my [latest project](https://www.skilltree.us), however, I tried to improve the testing methodology around my Stripe integration. And it was surprisingly difficult!

The actual test environment Stripe gives you is amazing, but it doesn't seem your automated tests should hit that. After `bundle install`-ing `stripe-ruby`, wouldn't it be cool if you could do [something like this](https://github.com/gkemmey/stripe_testing_poc):

![stripe_testing]({{ site.github.url }}/public/images/2019-05-04/example_run.gif)

Stripe lacks support for this kind of testing. So much so that there's at least [two](https://github.com/adrienverge/localstripe) [projects](https://github.com/rebelidealist/stripe-ruby-mock) I've seen just attempting to reverse-engineer a mock API server.

<div class="message yellow">
  <div class="message-body">
    I actually used the second link, and I intend to write more about that implementation. I think it's good (and if it's not, I'd love to learn why), but it still required a PR upstream and monkey patch just to make testing work 😬 Stay tuned!
  </div>
</div>

Payment integration seems too important for that. My example is _definitely_ just a proof of concept, but it uses Stripe's first-party [stripe-mock](https://github.com/stripe/stripe-mock) and [stripe-ruby](https://github.com/stripe/stripe-ruby) projects.

I'm not saying it'd be easy, but it's certainly doable, as indicated by those third party efforts to do it. It'd just be nice if it came backed and recommended by Stripe.

Anyway, just thinking aloud, and venting a little. Sorry!

</😤>
