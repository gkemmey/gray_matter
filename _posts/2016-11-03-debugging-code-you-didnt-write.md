---
layout: post
title: "Debugging Code You Didn't Write"
published: false
---

{% highlight ruby %}
module ActiveRecord
  module ConnectionAdapters
    class SchemaCache
      old_new = method(:new)

      define_singleton_method(:new) do |*args|
        Rails.logger.warn "Someone called new, wtf?!"
        Rails.logger.warn caller.join("\n")
        old_new.call(*args)
      end

      old_clear = instance_method(:clear!)

      define_method(:clear!) do
        Rails.logger.warn "Someone called clear!, wtf?!"
        Rails.logger.warn caller.join("\n")
        old_clear.bind(self).call
      end

      old_clear_table_cache = instance_method(:clear_table_cache!)

      define_method(:clear_table_cache!) do |table_name|
        Rails.logger.warn "Someone called clear_table_cache!, wtf?!"
        Rails.logger.warn caller.join("\n")
        old_clear_table_cache.bind(self).call(table_name)
      end
    end
  end
end
{% endhighlight %}
