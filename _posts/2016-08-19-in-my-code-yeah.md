---
layout: post
title: What?! 😳 In my code? Yes! 😁
published: true
---

Emoji's might be one of the coolest inventions to hit cell phones. Just behind the Internet. And texting itself. They're capable of adding _so much_ meaning, but probably most importantly, they're fun to use.

And clearly, people tend to agree. Most people litter their texts with them. In fact, I can tell when my wife's displeased with me just by the decreased frequency of smiley faces in her texts. Facebook [recently added them](http://newsroom.fb.com/news/2016/02/reactions-now-available-globally/). Hundreds of people took the time to [write an entire book using](http://www.emojidick.com/) nothing else. GitHub even took the time to make them a part of the [software development](https://github.com/blog/2119-add-reactions-to-pull-requests-issues-and-comments) process.

But you know where I haven't seen them? Comments. And why not? I'd be willing to guess most people (certainly millennials and younger) use them in their text message conversations all. the. time. To convey that extra level of meaning. Because "😂😂😂" shows your amusement better than "lol". And "👌" is <del>quicker</del> cooler than "ok".

What are comments if not quick text messages to your future reader? By adding emojis to our comments we can convey not only the technical details of why we wrote the code we did, but also how we felt about the code when we wrote it. And I'd argue that can be valuable to your future reader, be that yourself or someone else.

Not to mention, it can make the drudgery of documenting code more bearable. You know...slightly...

Let's look at some examples from codebases I'm juggling at work right now:

### Utterly Exasperated

{% highlight ruby %}
class Compass::History::Loan < Compass::Connection
  self.table_name = "KWHLOAN"

  # i'm not _entirely_ sure of the structure here, but it looks to me like a
  # KWHLOAN can be associated with exactly one KWHBASE row when both the HSEQ
  # and HDATE columns are equal. in other words:
  #
  # SELECT * FROM KWHBASE INNER JOIN KWHLOAN ON KWHBASE.HDATE = KWHLOAN.HDATW AND
  #                                             KWHBASE.HSEQ = KWHLOAN.HSEQ;
  #
  # we're replicating that as a has_one association by making the HDATE column
  # the primary key and adding conditions for the HSEQ portion. that's a
  # little misleading because technically there is no primary/foreign key
  # associating these two tables 😪
  has_one :past_payment, ->(loan) {
    where(hseq: loan.hseq)
  }, class_name: "Compass::History::Base", foreign_key: :hdate,
                                           primary_key: :hdate
end
{% endhighlight %}

I think adding this "exasperated sigh" emoji helps draw attention to our comment. It adds my emotions at the time of writing. I was exasperated with this code. I felt like my hands were tied. I needed an association the legacy database didn't _really_ support, and so we're stuck doing something kinda gross. The words convey the technical details of why. The emoji? That cues my future reader into how I felt about this code. All with a simple keystroke.

### An Inside Joke, Just for Fun

{% highlight ruby %}
def setup_organization # for use in testing when we need groups
  groups = {
    'married' => ['zoey@serenity.com', 'wash@serenity.com'],
    'tough' => ['zoey@serenity.com', 'mal@serenity.com', 'jayne@serenity.com'],
    'large_and_semi_muscular' => ['wash@serenity.com'] # 😂
  }

  # ...do some FactoryGirl shenanigans...
end
{% endhighlight %}

Perhaps, not the most necessary usage. But sometimes it's important to have fun with your code and your future reader. Especially, in test code, which isn't always the most fun to read or write. At least now the reader knows there's an inside [joke here](https://youtu.be/sBprC7i97-g?t=494), and, if curious enough, could look it up.

### Who Designed This?!? 😡

(See what I did there? 😂 And again.)

{% highlight ruby %}
def should_alert?
  # luckily, the type and user.alerts_type arguments don't match 😑
  !vulnerabilities.empty? &&
    (user.alerts_type == "all" || case type
                                    when "new"
                                      user.alerts_type == "new"
                                    when "updated"
                                      user.alerts_type == "updates"
                                  end)
end
{% endhighlight %}

This defines a quick helper method that lets us know if we should alert a user based on his alert settings and the type of alert we're processing. But, of course, `type` and `user.alerts_type` don't use the same verbiage so we have this complicated `case` checking. Me being me, I added some sarcastic text explaining this, and my personal "wtf?!" face emoji.

Normal prose has both voice and style. I think style comes pretty naturally when writing code. Whether good or bad, most developers have style in their code. How they indent things, when they comment, what constructs they use to do X vs Y, those are all part of a coder's style. But I don't think voice comes through as evidently. Emoji's add the voice. They make your code more personal, and ultimately more enjoyable to read.

If you're sold, [here](https://packagecontrol.io/packages/Emoji) is the emoji plugin I use with Sublime Text 3 which makes it super easy to search for and insert emojis. Here's hoping we see more 😁's and 💯's than 😑's in our comments!
