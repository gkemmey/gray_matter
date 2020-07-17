---
layout: post
title: 'Filling out PDFs'
published: true
---

A req came across my desk to fill out a PDF using data our app already has / collects. Specifically, these ones: [https://www.osha.gov/recordkeeping/new-osha300form1-1-04-FormsOnly.pdf](https://www.osha.gov/recordkeeping/new-osha300form1-1-04-FormsOnly.pdf).

Luckily, I was ready. A while ago, I had saved a reddit post where `u/CaptainKabob` was talking about how they had done this using a fillable PDF and "[`pdf-forms`](https://github.com/jkraemer/pdf-forms) (a gem that wraps [pdftk](https://www.pdflabs.com/tools/pdftk-the-pdf-toolkit/))". After a half a day or so of trying to convert a non-fillable PDF to a fillable PDF using Libre Office, briefly learning about [XFA and AcroForm](https://appligent.com/what-is-the-difference-between-acroforms-and-xfa/) fillable-form standards, trying to install `pdftk` on macOS, finding [this](https://stackoverflow.com/questions/39750883/pdftk-hanging-on-macos-sierra/39814799#39814799) unpublished link, learning about the [Java rewrite](https://gitlab.com/pdftk-java/pdftk) of `pdftk` because it doesn't work on Ubuntu > 18, and trying unsuccessfully to install that on my mac, I'm here to tell you fuck. that. approach. It mighta worked for `u/CaptainKabob`, but it's fucking horse shit 🐴

Instead, I'm here to show you the absolute, #1 way to fill out an existing PDF -- any PDF, fillable-form fields or not -- and it comes to us from that very same reddit post with an unassuming comment from `u/hcollider` ([link](https://www.reddit.com/r/rails/comments/8ohntl/generating_pdf_form_with_prawn/e03k552)):

![hcollider's reddit comment]({{ site.github.url }}/public/images/2020-07-17/hcolliders_reddit_comment.png)

Doubt not good sir!

I've done PDFs before, with all the usual suspects -- [`wkhtmltopdf`](https://wkhtmltopdf.org/) and [`wicked_pdf`](https://github.com/mileszs/wicked_pdf), [`combine_pdf`](https://github.com/boazsegev/combine_pdf), [`prawn`](https://github.com/prawnpdf/prawn) -- and nothing beats this!

Here's how it works:

1. We use `prawn` to make a brand new PDF with just the form fields filled out on white paper. It's text boxes and text with no background floating in white space.
2. We the use `combine_pdf` to lay those answers on top of the original PDF, and save that as a new PDF.

That's it! Just Ruby libraries. No binary package dependencies. Simple and elegant 👌

## Meet the tools 🛠

### prawn

`prawn` let's us create PDFs from text, shapes, and images by drawing on a coordinate plane where `(0, 0)` is the bottom-left-hand corner of a page. For example, this code generates <a href="{{ site.github.url }}/public/images/2020-07-17/rectangle.pdf" target="\_blank">this</a> PDF:

```ruby
Prawn::Document.generate("out/rectangle.pdf") do
  stroke_axis
  stroke_circle [0, 0], 10

  bounding_box([100, 300], width: 300, height: 200) do
   stroke_bounds
   stroke_circle [0, 0], 10
  end
end
```

Even more important to our use case of filling out a form, is `prawn's` text utilities. Checkout this example which generates <a href="{{ site.github.url }}/public/images/2020-07-17/text.pdf" target="\_blank">this</a> PDF:

```ruby
Prawn::Document.generate("out/text.pdf") do
  string = "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do " \
           "eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut " \
           "enim ad minim veniam, quis nostrud exercitation ullamco laboris " \
           "nisi ut aliquip ex ea commodo consequat."

  stroke_color "4299e1"

  [:truncate, :expand, :shrink_to_fit].each_with_index do |mode, i|
    text_box string, at: [i * 150, cursor],
                     width: 100,
                     height: 50,
                     overflow: mode
    stroke_rectangle([i * 150, cursor], 100, 50)
  end
end
```

Here is an image of that PDF, so we can talk about it:

<div class="img-bordered">
![text pdf as png]({{ site.github.url }}/public/images/2020-07-17/text_pdf_as_png.png)
</div>

`text_box` is the single most important method `prawn` gives us. It let us draw a text boxes by specifying its top left corner (`at`), its `width`, its `height`, and optionally what to do with text that doesn't fit (`overflow`). Also, it's worth noting that third `overflow` mode, `shrink_to_fit`. I don't know what dark magic `prawn` is using to do automagically shrink text for us, but it's incredibly useful when filling out form fields on our PDFs 🔮

### combine_pdf

Unsurprisingly, `combine_pdf` let's us...combine PDFs. Specifically, it lets us take the field content we generated with `prawn` and lay it on top of the original form. Let's take a look at an example that draws a grid on our form, like <a href="{{ site.github.url }}/public/images/2020-07-17/osha_form_300_with_grid.pdf" target="\_blank">this</a> PDF:

```ruby
# make a grid sheet
Prawn::Document.generate("out/grid_sheet.pdf", page_layout: :landscape,
                                               margin: 0,
                                               page_size: "LETTER") do # 8.5.in x 11.in

  height = 612
  width = 792

  (0..height).step(10).each do |y_pos|
    horizontal_line 12, width, at: y_pos
    stroke_color "e2e8f0" and stroke

    fill_color "4a5568"
    text_box("#{y_pos}", at: [0, y_pos + 3], width: 12, height: 6, size: 6, align: :right)

    (0..width).step(10).each do |x_pos|
      vertical_line 10, height, at: x_pos
      stroke_color "e2e8f0" and stroke

      fill_color "4a5568"
      text_box("#{x_pos}", at: [x_pos - 2, 0], width: 12, height: 6, size: 6, align: :right, rotate: 90)
    end
  end
end

# put it on our unfilled form
form = CombinePDF.load("templates/osha_form_300.pdf")
grid = CombinePDF.load("out/grid_sheet.pdf").pages[0]
grid.rotate_right

form.pages[0] << grid

form.save("out/osha_form_300_with_grid.pdf")
```

So that's all the pieces put together -- `prawn` and `combine_pdf` 🤝 As a bonus, being able to lay a grid over our original PDF is incredibly useful for figuring out where we should draw our text boxes when we're filling out the form.

## A more compete example 📝

### Hold up a second

I'm wanna show you what I did to fill out these [OSHA Forms](https://www.osha.gov/recordkeeping/new-osha300form1-1-04-FormsOnly.pdf). I think there's some interesting Ruby involved, but there's arguments for it being overcomplicated 🤷‍♂️

Anyway, you don't have to keep reading -- that grid example has everything you need.

At the end of the day, all you need are:

1. Calls to `prawn`'s text_box method with options for where to draw it on the page like so:

    ```ruby
    pdf.text_box "message", at: [148, 360.0],
                            width: 40,
                            height: 14,
                            valign: :bottom,
                            overflow: :shrink_to_fit,
                            size: 7
    ```

2. And to use `combine_pdf` to save the two PDFs as one, which they literally provide as an example in their [README](https://github.com/boazsegev/combine_pdf#add-content-to-existing-pages-stamp--watermark)

That's it!

I've been doing this web dev thing for a while now, and I can't believe this is a PDF solution I'm just now hearing about. To think it was just buried in a reddit post!

### Ok, let's over engineer 🤖

Go look at that OSHA Form 300 in the link above. If you do, you might notice there's lots of fields on there you're going to want to fill out the same way -- i.e. share styles. Sharing styles, as far as `prawn` is concerned, is really just sharing options passed to `text_box`.

So it'd be nice if whatever we come up with has some way to share like styles amongst different fields on our form, so we're not repeating options to the `text_box` method over and over. Additionally, we also need to specify at least some options on a case-by-case basis. For example, most fields probably have their own `x` and `y`.

<div class="message">
  <div class="message-body">
It's worth noting, I took a first pass at generating that form that did not share styles. It was essentially individual calls to `text_box`, each with their own options hash. Implementing it once (badly) does help illuminate what's missing 😬
  </div>
</div>

When I find myself knowing roughly what I want, but not sure how to implement it, I like to start by writing the code I wish I had. So let's start there. Let's write code that looks like could do more or less what we outlined above:

```ruby
class PDF::Form300
  default_cell_height 14
  default_cell_font_size ->(options) { options[:height] }
  default_cell_valign :bottom
  default_cell_overflow :shrink_to_fit

  cell_type :field, font_size: 7

  field :establishment_name, x: 658, y: 465, width: 110, height: 12
end
```

None of that works of course, but it does communicate some things about our eventual solution:

1. Individual areas of the form we need to fill out are called "cells"
2. Defaults that will apply to all cells can be created using the `default_cell_height`, `default_cell_font_size`, `default_cell_valign`, and `default_cell_overflow` methods
3. Different types of cells with their own defaults can be created using the `cell_type` method
4. Creating a "cell type" creates a new method -- `field` in the code above -- that can be used to specify a named cell we need to fill in with its own options like where it's located and it's width

While the code we wrote doesn't indicate this is true, let's also go ahead and say each set of options overrides the ones that have come before it. Meaning, options passed to a named cell override options set on the cell type, which override options set as defaults of all cells.

Let's add some more, still just wishfully thinking:

```ruby
class PDF::Form300
  Y_OF_TOP_LEFT_CORNER_FOR_PAGE_TOTAL_CELLS = 142 # ✨new

  default_cell_height 14
  default_cell_font_size ->(options) { options[:height] }
  default_cell_valign :bottom
  default_cell_overflow :shrink_to_fit

  cell_type :field,      font_size: 7
  cell_type :page_total, y: Y_OF_TOP_LEFT_CORNER_FOR_PAGE_TOTAL_CELLS, width: 15,    # ✨new
                                                                       height: 10,   # ✨new
                                                                       align: :right # ✨new
  cell_type :check_box, width: 6, height: 6, style: :bold, align: :center, valign: :center # ✨new

  field :establishment_name, x: 658, y: 465, width: 110, height: 12

  page_total :classified_as_death_page_total, x: 476            # ✨new
  page_total :resulted_in_injury_page_total,  x: 680, width: 10 # ✨new
end
```

<div class="message">
  <div class="message-body">
If you haven't yet looked at the [OSHA Form 300](https://www.osha.gov/recordkeeping/new-osha300form1-1-04-FormsOnly.pdf), go do so. This code will make more sense if you have an idea of what that PDF looks like in your head.
  </div>
</div>

Notice we've defined a new cell type, `page_total`, and then we used it to create to new named cells: `classified_as_death_page_total` and `resulted_in_injury_page_total`. As a bit of foreshadowing and to help better visualize, here's what those `page_totals` (and all the others) look like filled out -- you know, after we make all this code work:

<div class="img-bordered">
![page totals]({{ site.github.url }}/public/images/2020-07-17/page_totals.png)
</div>

Take a look at how styles are being shared for our `page_totals`. In the call to `cell_type`, we give them all the same `y`, `width`, `height`, and text alignment (`align`). Then when we define the cell for `classified_as_death_page_total` all we have to give it is the `x`! And that's true for the first six cells in the image.

Additionally, when we define the cell for `resulted_in_injury_page_total` we give it a `width` in addition to the `x`. Look at the image again. Notice the last six cells are little thinner. The `width` we passed to `resulted_in_injury_page_total` (and will pass to the other five cells, too) _overrides_ the `width of 15` in our cell type. But all of those `page_total` cells in that image are (or will be) created using the `page_total` method.

Ok, let's add one last snippet of wishlist code. If you look at the OSHA Form 300 one more time, you might notice that the form is essentially a table of incidents. The borders aren't drawn, but there's columns like "Case no." and "Employee's name", and there's thirteen rows where we can put incident information. In fact, there's eighteen columns in that table. So we could think of that as 234 (`13 * 18`) different cells on our form, but we don't have to.

Consider the first column, "Case no.": All thirteen cells for "Case no." on our form are going to share the same styles except one -- the `y`. Right? Their top-left corner's will all have the same `x`, they'll all have the same `height`, `width`, etc.

So for our last bit of wishlist code, instead of defining each case number cell like:

```ruby
field :case_number_one, y: 360,   x: 29, width: 18
field :case_number_two, y: 344.5, x: 29, width: 18
# ... 11 more times ....
```

Let's make it so we can define it once, but as a part of a table. Like this:

```ruby
class PDF::Form300
  Y_OF_TOP_LEFT_CORNER_OF_FIRST_INCIDENT_CELL = 360
  SPACE_IN_BETWEEN_INCIDENT_ROWS = 16.5

  # ... all the stuff from our last example ...

  table y:      Y_OF_TOP_LEFT_CORNER_OF_FIRST_INCIDENT_CELL,
        offset: SPACE_IN_BETWEEN_INCIDENT_ROWS do |t|

    field :case_number,   x: 29, width: 18
    field :employee_name, x: 53, width: 82

    check_box :classified_as_death, x: 480, y: 353, offset: (t.offset + 0.1)
  end
end
```

A `table` knows two things: 1) they `y` of its first (top-left-most) cell and 2) how much space to put in between each row (`offset`). From there, if indicate the `row` of the `case_number` we want to fill out, the `table` can calculate the `y` for us using `Y_OF_TOP - (row * SPACE_BETWEEN_ROWS)`.

Last thing, our table wishlist snippet does is make it possible to override the `offset` a column should use. So when we're calculating the y for a `classified_as_death` cell, instead of using `row * 16.5`, we'll use `row * 16.6`. Turns out those check boxes in that column have just a little more space in between each row, and it's surprisingly noticeable if we don't adjust the `offset`:

<div class="img-scaled height-down-25">

| `offset = 16.5` | `offset = 16.6` |
| --------------- | --------------- |
| ![checkboxes bad]({{ site.github.url }}/public/images/2020-07-17/checkboxes_with_bad_offset.png) | ![checkboxes good]({{ site.github.url }}/public/images/2020-07-17/checkboxes_with_good_offset.png) |

</div>

Ok, that's everything...that we...uh..._wish_ we could do, lol 🌠 Let's make it work! #wishdrivendevelopment

### Making it work ✅

To do so, we're gonna create a mixin -- a module, a concern, they go by many names -- called `PDF::Layout`. But first, a design constraint: whenever we go to fill in cell, I want that to call a distinct method. For example, filling in the `establishment_name` should call `fill_in_establishment_name(name)`. If we get an error, the field that caused it should be easily discoverable in the stack trace. It's a Rails app generating these things from user-supplied data after all.

Given that, calling `field :establishment_name, x: 658, y: 465, width: 110, height: 12` in our `PDF::Form300` class should define an instance method `fill_in_establishment_name` that looks roughly like so:

```ruby
def fill_in_establishment_name(name)
  fill_in(name, options) # we dunno where options is gonna come from yet
end
```

That `fill_in` method would be the thing that finally calls `pdf.text_box`. Let's start there:

```ruby
module PDF
  module Layout
    extend ActiveSupport::Concern

    def fill_in(value, options = {})
      _options = options.transform_values { |v|
                           v.respond_to?(:call) ? v.call(options) : v
                         }

      _options[:at] = [_options.delete(:x), _options.delete(:y)]
      _options[:size] = _options.delete(:font_size)

      if outline_text_boxes?
        pdf.stroke_color "4299e1"
        pdf.stroke_rectangle(_options[:at], _options[:width], _options[:height])
      end

      pdf.text_box(value.to_s, _options)
    end
  end
end
```

First things first, `fill_in` takes the `options` it got and calls any of the "callable" values giving them those same `options`. That let's us take an option like `{ font_size: ->(options) { options[:height] } }` and resolve it to `{ font_size: whatever_height_is_set_to }` just before we finally ready to write to the PDF.

Then we turn our `x` and `y` options in to a single `at` option that takes them as an array, and we turn `font_size` into `size`. These changes just improve (in my opinion) the API of `prawn`'s `text_box` method.

Then we check if we should draw the outline of the `text_box`, and if so, do so. I find that's helpful in development.

Lastly, we call `pdf.text_box` passing along those options.

Great, that's the final stop of our DSL -- we're kinda working backwards. Let's add what we need to make DSL methods work. Next up, the `default_cell_x` methods:

```ruby
module PDF
  module Layout
    extend ActiveSupport::Concern

    included do
      class_attribute :defaults, instance_accessor: false
      self.defaults = {}
    end

    class_methods do
      def default_cell_height(value)
        self.defaults[:height] = value
      end

      def default_cell_valign(value)
        self.defaults[:valign] = value
      end

      def default_cell_overflow(value)
        self.defaults[:overflow] = value
      end

      def default_cell_font_size(value)
        self.defaults[:font_size] = value
      end
    end

    # ... fill_in ...
  end
end
```

So when `PDF::Layout` is included, we setup a class variable `defaults` to our defaults that apply to all cells. Each `default_cell_x` method just sets a key in that hash equal to the value you gave it.

Ok, here's where things kinda go to shit 💩implementing that DSL. If you look back at our `PDF::Form300` class we had code like this:

```ruby
class PDF::Form300
  cell_type :field, font_size: 7

  field :establishment_name, x: 658, y: 465, width: 110, height: 12
end
```

Do you see what happened there? After calling `cell_type :field` we have access to a newly-defined class method `field`. DSL's are pure fucking magic, don't let anyone tell you otherwise 🔮 Anyway, let's attempt to demystify a bit:

```ruby
module PDF
  module Layout
    extend ActiveSupport::Concern

    included do
      # ... defaults class variable setup ...
    end

    class_methods do
      # ... default_cell_x methods ...

      def cell_type(type, type_defaults = {})
        define_method "defaults_for_#{type}" do                 # 1
          type_defaults.dup                                     # 1
        end                                                     # 1

        class_eval <<-RUBY, __FILE__, __LINE__ + 1              # 2
          def self.#{type}(name, defaults_for_cell = {})        # 2
            define_method "fill_in_\#{name}" do |value|         # 2 # 3
              fill_in(value, **self.class.defaults,             # 2 # 3
                             **defaults_for_#{type},            # 2 # 3
                             **defaults_for_cell)               # 2 # 3
            end                                                 # 2 # 3
          end                                                   # 2
        RUBY
      end
    end

    # ... fill_in ...
  end
end
```

Lulz, what the fuck even is Ruby? Don't worry! If you'll allow my crude annotations, we can take this in pieces:

1. (Lines with `#1`) -- This section defines a new instance method called `defaults_for_#{type}`. In our case, let's imagine we've just called `cell_type :field, font_size: 7`. `type` is `:field`, so this creates an instance method called `defaults_for_field`.

    Importantly, it defines this method using the method `define_method` that takes a block. This block is closure that maintains access to our second argument to `cell_type`, the `type_defaults` hash. In our case, `type_defaults` is a hash like `{ font_size: 7 }`.

    What this means is any instance of our `PDF::Form300` class can call `defaults_for_field` to get back a copy of that hash that was passed to `cell_type :field`.

2. (Lines with `#2`) -- This section defines the `field` class method we know calling `cell_type :field` needs to create for us. `class_eval` takes a string of Ruby code -- all that yellow text ☝️ -- and evaluates it for us in the context of the class. That string of Ruby code defines our `field` method.

    If we ditch the lines marked with a `#3` for a minute, and substitute `:field` for `type`, here's what that same section looks like:

    ```ruby
    class_eval <<-RUBY, __FILE__, __LINE__ + 1
      def self.field(name, defaults_for_cell = {})
      end
    RUBY
    ```

    Nothing wild there, we're defining a class method `field` that takes a `name` and a hash of `defaults_for_cell`.

3. (Lines with `#3`) -- Now, when we call `field` (or any of our methods defined by calling `cell_type`) we need to create an instance method for filling in that single cell. Assuming we had called `field :establishment_name`, the instance method we're creating is called `fill_in_establishment_name`, and it takes a single argument, `value`.

    Once again, we turn to `define_method` and a block closed over `defaults_for_cell` so the instance method `fill_in_establishment_name` can maintain access to the options we passed when we called `field :establishment_name`.

    When you call `fill_in_establishment_name`, it passes `value` to the `fill_in` method, and spreads in (`**`) all the different option hashes starting with our `defaults`, then any `defaults_for_field`, then any `defaults_for_cell` -- each, in turn, overriding any options that were set before.

Ok, that explains everything but `table`. If you remember our `table` example it looked like this:

```ruby
class PDF::Form300
  Y_OF_TOP_LEFT_CORNER_OF_FIRST_INCIDENT_CELL = 360
  SPACE_IN_BETWEEN_INCIDENT_ROWS = 16.5

  table y:      Y_OF_TOP_LEFT_CORNER_OF_FIRST_INCIDENT_CELL,
        offset: SPACE_IN_BETWEEN_INCIDENT_ROWS do |t|

    field :case_number,   x: 29, width: 18
    field :employee_name, x: 53, width: 82

    check_box :classified_as_death, x: 480, y: 353, offset: (t.offset + 0.1)
  end
end
```

As we've just seen, normally calling `field :case_number` would create `fill_in_case_number` method takes just a `value` to write into the PDF. But notice `table` takes a block. Inside the block, we're gonna make it so calling `field :case_number` creates a `fill_in_case_number` that _not only_ takes a `value` argument, but a `row` keyword argument too. It'll then use that `row` argument and the `y` and `offset` options passed to `table` to calculate the `y` for the cell.

Ok, here's the `table` class method defined in our mixin:

```ruby
module PDF
  module Layout
    extend ActiveSupport::Concern

    included do
      # ... defaults class variable setup ...
    end

    class_methods do
      # ... default_cell_x methods ...

      # ... cell_type method ...

      def table(y:, offset:, &block)
        Table.new(klass: self, y: y, offset: offset).evaluate(&block)
      end
    end

    # ... fill_in ...
  end
end
```

Ignoring the `Table` class we haven't seen yet, the `table` method's not too bad. It creates a new `Table` passing along `self` (the class that has included the mixin and called `table` -- `PDF::Form300` in our case), `y`, and `offset` to the new `Table` instance, and asks the `Table` instance to evaluate the block.

Here's the `Table` class:

```ruby
module PDF
  module Layout
    extend ActiveSupport::Concern

    class Table
      attr_accessor :klass, :y, :offset

      def initialize(attributes = {})
        attributes.each { |name, value| send("#{name}=", value) }
      end

      def evaluate(&block)                                              # 1
        instance_eval(&block)                                           # 1
      end                                                               # 1

      def define_fill_in_method(type, name, defaults_for_cell = {})
        _offset = defaults_for_cell.delete(:offset) || offset
        _y = defaults_for_cell.delete(:y) || y

        klass.send(:define_method, "fill_in_#{name}") do |value, row:|  # 3
          fill_in(value, **self.class.defaults,                         # 3
                         **send("defaults_for_#{type}"),                # 3
                         **defaults_for_cell,                           # 3
                         y: _y - (row * _offset))                       # 3
        end                                                             # 3
      end

      def method_missing(name, *args, &block)
        # is this a type defined by a call to cell_type? if so, it will
        # have told the klass to define this method.
        if klass.method_defined?("defaults_for_#{name}")                # 2
          # a little confusing, name here is the method name. for us,
          # that's a cell_type.
          define_fill_in_method(name, *args)                            # 2
        else
          super
        end
      end
    end

    included do
      # ... defaults class variable setup ...
    end

    class_methods do
      # ... default_cell_x methods ...

      # ... cell_type method ...

      def table(y:, offset:, &block)
        Table.new(klass: self, y: y, offset: offset).evaluate(&block)
      end
    end

    # ... fill_in ...
  end
end
```

One more set of crude annotations:

1. (Lines with `#1`) -- The `evaluate` method. This just passes the block to `instance_eval` which runs the block, but makes it so while executing the block `self` is set to our `Table` instance. Meaning, inside the block we pass to `table`, when we call `field :case_number`, our `Table` instance receives that call to `field`.

    Again, lulz, what the fuck even is Ruby?

2. (Lines with `#2`) -- But of course, `Table` doesn't have a `field` instance method, so we handle that in `method_missing`. Whenever a `Table` instance receives a method it doesn't have defined, it asks `klass`, "Hey, is this method really a `cell_type` you know about?". If so, our `Table` instance can do the work of defining a `fill_in_x` instance method _on `klass`_ that takes a `row` argument and automagically calculates the `y` argument for `pdf.text_box`.

3. (Lines with `#3`) -- And this section does just that. Note, it's nothing we haven't seen before. Assuming we had called `field :case_number` inside our `table` block, we're still using `define_method` to create an instance method called `fill_in_case_number`, and when that's called passing a `value` and our merged options along to the `fill_in` method. The only new thing, `fill_in_case_number` takes a `row` argument that it uses with `Table#y` and `Table#offset` to calculate the `y` of this cell and provide that as an option to the `fill_in` method.

Ok, that's it for our `table` method. With that, the DSL we wished we could right works!

### Putting it all together

Here's our final PDF class (with some redactions for brevity):

```ruby
class PDF::Form300
  include PDF::Layout

  Y_OF_TOP_LEFT_CORNER_FOR_PAGE_TOTAL_CELLS = 142
  Y_OF_TOP_LEFT_CORNER_OF_FIRST_INCIDENT_CELL = 360
  SPACE_IN_BETWEEN_INCIDENT_ROWS = 16.5
  INCIDENT_ROWS_PER_SHEET = 13

  default_cell_height 14
  default_cell_font_size ->(options) { options[:height] }
  default_cell_valign :bottom
  default_cell_overflow :shrink_to_fit

  cell_type :field,      font_size: 7
  cell_type :page_total, y: Y_OF_TOP_LEFT_CORNER_FOR_PAGE_TOTAL_CELLS, width: 15,
                                                                       height: 10,
                                                                       align: :right
  cell_type :check_box, width: 6, height: 6, style: :bold, align: :center, valign: :center

  field :establishment_name, x: 658, y: 465, width: 110, height: 12
  page_total :classified_as_death_page_total, x: 476

  table y: Y_OF_TOP_LEFT_CORNER_OF_FIRST_INCIDENT_CELL,
        offset: SPACE_IN_BETWEEN_INCIDENT_ROWS do |t|

    field :case_number, x: 29,  width: 18
    check_box :classified_as_death, x: 480, y: 353, offset: (t.offset + 0.1)
  end

  # ... more layout stuffs ...

  def self.generate(incidents, attributes = {})
    new(incidents, attributes).generate
  end

  # ... initializer / accessor stuffs ...

  def pdf
    @pdf ||= Prawn::Document.new(page_layout: :landscape, page_size: "LETTER", margin: 0)
  end

  def generate
    incidents.each_slice(INCIDENT_ROWS_PER_SHEET).
              each_with_index do |incidents_group, page|

      pdf.start_new_page unless page.zero?

      fill_in_establishment_name(location.name)

      fill_in_classified_as_death_page_total \
        incidents_group.count(&:classified_as_death?)

      incidents_group.each_with_index do |incident, row|
        fill_in_case_number          incident.case_number,
                                     row: row
        fill_in_classified_as_death  check_mark(incident.classified_as_death?),
                                     row: row
      end
    end

    write_pdf
  end

  private

    def check_mark(boolean)
      boolean ? "X" : ""
    end
end
```

Finally, making the PDF is just a matter of calling the `fill_in_x` -- like `fill_in_establishment_name` -- methods our DSL creates. In this example, that happens in the `generate` method. You can see a full, working version of generating this OSHA Form 300 [here](), as well as here's direct links to the [mixin]() and [pdf class](). Also, here's the [final, filled-out PDF]() running that full version creates.
