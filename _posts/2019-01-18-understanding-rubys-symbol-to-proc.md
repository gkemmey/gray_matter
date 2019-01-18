---
layout: post
title: "Understanding Ruby's :symbol.to_proc"
published: true
---

At work, someone posted this code to our #rails channel and asked if there's was more ruby way to skip the re-assigning of `@employees`, maybe using `yield_self`:

{% highlight ruby %}
class EmployeesController < ApplicationController
  def index
    @employees = User.employee.includes(:contact_information)
    @employees = @employees.order("#{sort_column} #{sort_direction}") if sorting?
    @employees = @employees.page(params[:page] || 1).per(20)
  end

  private

    def sort_column
      (["last_name","first_name"] & Array(params[:sort])).first
    end

    def sort_direction
      (["asc", "desc"] & Array(params[:dir])).first || "asc"
    end

    def sorting?
      !!sort_column
    end
end
{% endhighlight %}

And the bike shedding started 🚲 I think I proposed something like:

{% highlight ruby %}
class EmployeesController < ApplicationController
  def index
    @employees = User.employee.includes(:contact_information).
                               then(&apply_sorting).
                               then(&apply_pagination)
  end

  private

    def sort_column
      (["last_name","first_name"] & Array(params[:sort])).first
    end

    def sort_direction
      (["asc", "desc"] & Array(params[:dir])).first || "asc"
    end

    def sorting?
      !!sort_column
    end

    def apply_sorting
      method(:_apply_sorting)
    end

    def _apply_sorting(relation)
      return relation unless sorting?
      relation.order("#{sort_column} #{sort_direction}")
    end

    def apply_pagination
      method(:_apply_pagination)
    end

    def _apply_pagination(relation)
      relation.page(params[:page] || 1).per(20)
    end
end
{% endhighlight %}

And sure, sure there's an argument for that being far too over-engineered, but what's really interesting, and what this post is actually about, is someone proceed to ask

1. What's `method` do?
2. Why doesn't `then(&:apply_sorting)` (i.e. without the `method` bit) work?
3. And why _does_ `[1, 2].inject(&:+)` work?

> _Quick rebuttal to that 'over-engineered' argument: minus the `&` that index action sure is readable, and we could slide all those private methods into a reusable `Sortable` concern._

Anyway, I didn't have a super satisfactory answer to all those questions. I just kinda knew the `&method(:apply_sorting)` pattern would work here, and `inject`'s good with just `&:+`. So I figured I'd find out.

---

☝️ All that's sort of the premise / some context. Here's we'll flip to some simpler examples you can run right in irb.

Suppose we've defined an `apply_filter` method like so:

{% highlight ruby %}
def apply_filter(array)
  array.select { |e| e % 2 == 0 }
end
{% endhighlight %}

We could then run the following examples in irb:

{% highlight ruby %}
[1, 2, 3, 4].yield_self(&method(:apply_filter)) # => [2, 4]
{% endhighlight %}

{% highlight ruby %}
[1, 2, 3, 4].yield_self(&:apply_filter)
# ArgumentError (wrong number of arguments (given 0, expected 1))
{% endhighlight %}

{% highlight ruby %}
[1, 2, 3, 4].inject(&:+) #=> 10
{% endhighlight %}

Why? Why does `yield_self(&:apply_filter)` fail, but `inject(&:+)` work? First off, we gotta be clear about what `&` does:

>**In a method argument list**, the `&` operator takes its operand, converts it to a Proc object if it isn't already (by calling to_proc on it) and passes it to the method.

So the "in a method argument list is important". In fact outside of one, we get an error:

{% highlight ruby %}
&:+
# SyntaxError (unexpected &)
{% endhighlight %}

But that doesn't explain why `[1, 2, 3, 4].inject(&:+)` works and `[1, 2, 3, 4].yield_self(&:apply_filter)` doesn't. To explain that, we've gotta look at how `Symbol#to_proc` works.

Normal procs are associated with a `Binding` object which is responsible for capturing all the bindings (i.e. variable assignments) and the receiver from the scope in which the proc was declared. We can see that in action:

{% highlight ruby %}
x = 1
proc {}.binding.eval('x')
# => 1
{% endhighlight %}

_`eval` will run the string you give it as ruby code within that Binding object._

However, the `to_proc` method on `Symbol` is defined in C and returns a "C level Proc", and therefore doesn't have a binding.

```
:+.to_proc.binding
# ArgumentError (Can't create Binding from C level Proc)
```

See how this is different if we instead call `to_proc` on the `+` Method object from an integer:

{% highlight ruby %}
1.method(:+).to_proc.binding
# #<Binding:0x00007fcb18072430>
1.method(:+).to_proc.binding.receiver
# 1
{% endhighlight %}

Instead, the C Proc returned from `:+.to_proc` expects you to give it a receiver as the first argument:

{% highlight ruby %}
:+.to_proc.cal()
# ArgumentError (no receiver given)

:+.to_proc.call(1)
# ArgumentError (wrong number of arguments (given 0, expected 1))

:+.to_proc.call(1, 2)
# 3
{% endhighlight %}

Ok, armed with that we can take a look at the block signature for [inject](https://ruby-doc.org/core-2.2.3/Enumerable.html#method-i-inject):

```
inject(initial) { |memo, obj| block } → obj
```

It actually takes two arguments, `memo` and `object`. So when we pass it the C proc from `&:+` it lets `memo` be the receiver and `object` be the argument, so it works!

If we look a the block signature for [yield_self](https://ruby-doc.org/core-2.5.3/Object.html#method-i-yield_self):

```
yield_self { |x| block } → an_object
```

It expects only the single argument, so with `[1, 2, 3, 4].yield_self(&:apply_filter)` we wind up setting the array (or `x` from the signature above) as the receiver for the proc returned from `apply_filter.proc`, but then fail to pass it the `array` argument it's expecting (see the method definition at the top).

Using `[1, 2, 3, 4].yield_self(&method(:apply_filter))` fixes this by instead calling `to_proc` on the `Method` object returned by `method(:apply_filter)`, not the symbol `:apply_filter`. In this case, we don't get back a C Proc, we get back a normal Ruby proc which does have a binding and a receiver (which in this case is  whatever `self` is).
