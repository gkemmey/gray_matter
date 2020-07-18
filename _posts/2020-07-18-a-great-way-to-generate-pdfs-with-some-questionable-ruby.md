---
layout: post
title: 'A Great Way to Generate PDFs with Some Questionable Ruby'
published: true
---

A while ago, I bookmarked this [reddit post](https://www.reddit.com/r/rails/comments/8ohntl/generating_pdf_form_with_prawn/e03k552) about generating PDFs. There were some cool sounding ideas in there, and as a web developer, it's only a matter of time before you're asked to fill out a paper version of a form.

I've done PDFs before, with the usual suspects -- [wkhtmltopdf](https://wkhtmltopdf.org/) and [wicked_pdf](https://github.com/mileszs/wicked_pdf) -- and it works ok, but never perfectly. It always feels like you're fighting some styling or page break issue. Plus, when a PDF version form already exists, do you really want to have to recreate it? So I was eager to try something else.

The first approach I tried was using [pdf-forms](https://github.com/jkraemer/pdf-forms) to programmatically fill out a fillable PDF. Spoiler: this ssssuuuccckkkeedd. After far too long of trying to convert a non-fillable PDF to a fillable PDF using Libre Office, learning more than I ever cared to about [XFA and AcroForm](https://appligent.com/what-is-the-difference-between-acroforms-and-xfa/) fillable-form standards, trying to install [pdftk](https://www.pdflabs.com/tools/pdftk-the-pdf-toolkit/) (the binary `pdf-forms` wraps) on macOS, finding [the real version of pdftk](https://stackoverflow.com/questions/39750883/pdftk-hanging-on-macos-sierra/39814799#39814799) you need to install on stackoverflow, learning about the [Java rewrite](https://gitlab.com/pdftk-java/pdftk) of `pdftk` because the original doesn't work on Ubuntu > 18, and trying unsuccessfully to install that on my mac -- I'm here to tell you just skip this one.

You gotta know when to cut bait, amirite? 🎣

Luckily, buried in that very same reddit post was this rather unassuming comment from `u/hcollider` ([link](https://www.reddit.com/r/rails/comments/8ohntl/generating_pdf_form_with_prawn/e03k552)):

![hcollider's reddit comment]({{ site.github.url }}/public/images/2020-07-18/hcolliders_reddit_comment.png)

Doubt not good sir! This approach is great! That comment is a little light on detail though. Actually, it's all inspiration, but sometimes that's enough. Here's how we can do it:

1. We'll use `prawn` to make a brand new PDF with just the form fields filled out on white paper. It's text boxes and text with no background floating in white space.
2. We'll the use `combine_pdf` to lay those answers on top of the original PDF, and save that as a new PDF.

That's it! Just Ruby libraries. No binary package dependencies. Simple and elegant 👌

## Meet the tools 🛠

### prawn

`prawn` let's us create PDFs from text, shapes, and images by drawing on a coordinate plane where `(0, 0)` is the bottom-left-hand corner of a page. For example, this code generates <a href="{{ site.github.url }}/public/images/2020-07-18/rectangle.pdf" target="\_blank">this</a> PDF:

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

More importantly for filling out forms, are `prawn's` text utilities. Checkout this example which generates <a href="{{ site.github.url }}/public/images/2020-07-18/text.pdf" target="\_blank">this</a> PDF:

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

For reference, here's a screenshot of that PDF:

<div class="img-bordered">
![text pdf as png]({{ site.github.url }}/public/images/2020-07-18/text_pdf_as_png.png)
</div>

`text_box` lets us draw a text box by specifying its top left corner (`at`), its `width`, its `height`, and optionally what to do with text that doesn't fit (`overflow`). The third `overflow` mode in that picture, `shrink_to_fit`, is especially useful when filling out form fields on a PDF.

### combine_pdf

Unsurprisingly, `combine_pdf` let's us...combine PDFs. Ultimately, we'll use it take a PDF of all the form's content, generated with `prawn`, and lay it on top of the original PDF form. Here's an example of doing something similar, but instead of filling out the form, we draw a grid on it (<a href="{{ site.github.url }}/public/images/2020-07-18/osha_form_300_with_grid.pdf" target="\_blank">the PDF</a>):

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
    text_box("#{y_pos}", at: [0, y_pos + 3], width: 12, height: 6, size: 6,
                                                                   align: :right)

    (0..width).step(10).each do |x_pos|
      vertical_line 10, height, at: x_pos
      stroke_color "e2e8f0" and stroke

      fill_color "4a5568"
      text_box("#{x_pos}", at: [x_pos - 2, 0], width: 12, height: 6, size: 6,
                                                                     align: :right,
                                                                     rotate: 90)
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

This isn't a pointless example. We can use a grid like that to help us properly lay out text boxes when we fill out the form for real. Also, while we haven't filled out the form per se, that example shows everything you need for doing so yourself. Instead of drawing lines and text boxes for your axis labels you would draw text boxes for your data fields, but that's all the tools -- `prawn` and `combine_pdf` 🤝 -- put together.

If all we wanted to do was showcase the methodology, we'd be done, but where's the fun in that? Let's do the whole form!

## Filling out a PDF form for reals 📝

### Hold up a second!

You don't have to keep reading -- that grid example has everything you need. Past here, we're mostly just having fun with Ruby in the context of filling out a PDF. At the end of the day, all you need are:

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

That's it! I can't believe this is a PDF solution I'm just now hearing about. It feels really robust. No binaries needed, just two Ruby libraries. You don't even need the PDFs to be already available -- you could create those in whatever your favorite PDF software is. To think it was just buried in a reddit post!

### Starting with something verbose and repetitive, but simple

We're gonna fill out the [OSHA Form 300](https://www.osha.gov/recordkeeping/new-osha300form1-1-04-FormsOnly.pdf). Partially, because it's the very form I had to fill out recently, but mostly because can you think of a more enthralling example!? Ok, it ain't the height of excitement, but it's a mildly complex form that'll let us write some flashy Rubies.

Let's start with the basics, minus a few helper methods, here's roughly what a simple first pass might look like:

```rb
class PDF::Form300
  def generate
    pdf.text_box(establishment_name, at: [658, 465],
                                     width: 110,
                                     height: 12,
                                     font_size: 7,
                                     min_font_size: 0,
                                     overflow: :shrink_to_fit,
                                     valign: :bottom)

    pdf.text_box(city,  at: [622, 452],
                        width: 80,
                        height: 12,
                        font_size: 7,
                        min_font_size: 0,
                        overflow: :shrink_to_fit,
                        valign: :bottom)

    pdf.text_box(classified_as_death_page_total, at: [476, 142],
                                                 width: 15,
                                                 height: 10,
                                                 font_size: 10,
                                                 min_font_size: 0,
                                                 overflow: :shrink_to_fit,
                                                 valign: :right)
    # ... rinse and repeat ....
  end
end

Pdf::Form300.new(data).generate
```

You can take that to it's logical conclusion, and it's less than spectacular.

<div class="message">
  <div class="message-body">
In fact, I _did_ take it to its logical conclusion -- that was my first pass. I had some helper methods to share some of those styles or help fill out fields based on their row in essentially the giant table that is this form (spoiler!), but it wasn't appreciably different than what we wrote above.

Writing the ~~first~~ bad version was necessary to guide the better version we're gonna build below.
  </div>
</div>

Go look at that OSHA Form 300 in the link above. If you do, you might notice there's lots of fields on there you're going to want to fill out the same way -- i.e. share styles. Sharing styles, as far as `prawn` is concerned, is really just sharing options passed to `text_box`.

Three form fields in, you can already see that in the code. They all set `overflow: :shrink_to_fit`. The first two share everything but their `at` and `width`.

We can surely do better, but whatever we come up with has to have some way to share like styles amongst different fields on our form so we're not repeating options to the `text_box` method over and over. Additionally, we also need to specify at least some options on a case-by-case basis. For example, most fields probably have their own `x` and `y`.

### Writing the code we wish we had

When I find myself knowing vaguely what I want, but not sure how to implement it, I like to write the code I wish I had. So let's start there. Let's write code that just _looks_ like it could maybe fill out our form:

```ruby
class PDF::Form300
  default_cell_height 14
  default_cell_font_size ->(options) { options[:height] }
  default_cell_valign :bottom
  default_cell_overflow :shrink_to_fit
  default_cell_min_font_size 0

  cell_type :field, font_size: 7

  field :establishment_name, x: 658, y: 465, width: 110, height: 12
end
```

None of that works of course, but it still hints at facets of our eventual solution:

1. Individual areas of the form we need to fill out are called "cells"
2. Defaults that will apply to all cells can be created using the `default_x` methods
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
  default_cell_min_font_size 0

  cell_type :field,      font_size: 7
  cell_type :page_total, y: Y_OF_TOP_LEFT_CORNER_FOR_PAGE_TOTAL_CELLS, # ✨new
                         width: 15,                                    # ✨new
                         height: 10,                                   # ✨new
                         align: :right                                 # ✨new
  cell_type :check_box, width: 6, height: 6, style: :bold,   # ✨new
                                             align: :center, # ✨new
                                             valign: :center # ✨new

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

We've defined a new cell type, `page_total`, and then we used it to define two new cells on the form: `classified_as_death_page_total` and `resulted_in_injury_page_total`. As a bit of foreshadowing and to help better visualize, here's the cells on the form we're calling `page_totals`:

<div class="img-bordered">
![page totals]({{ site.github.url }}/public/images/2020-07-18/page_totals.png)
</div>

You can see all twelve of those boxes are super similar. They're all aligned right, all have the same font size, all have the same height, they're all horizontally aligned meaning they're left-hand corners all have the same `y` value.

Now, look at the code where we create this `page_total` cell type -- the line that starts `cell_type :page_total` -- we set all those same options. Then we can use it to define the cell for `classified_as_death_page_total` by specifying just the `x`. And we can use it to define the cell for `resulted_in_injury_page_total` by specifying the `x` and a new `width`.

Look at the image again. Notice the last six cells are little thinner. The `width` we passed to `resulted_in_injury_page_total` _overrides_ the `width` of `15` in our call to `cell_type`. So we can create all twelve of those page total cells using our `page_total` method, it's just for six of them we'll specify a new width.

Ok, let's add one last snippet of wishlist code. If you look at the OSHA Form 300 one more time, you might notice that the form is essentially a table of incidents. The borders aren't drawn, but there's columns like "Case no." and "Employee's name", and there's thirteen rows where we can put incident information. In fact, there's eighteen columns in that table. So we could think of that as 234 (`13 * 18`) different cells on our form, but we don't have to.

Consider the first column, "Case no.": All thirteen cells for "Case no." on our form are going to share the same styles except one -- the `y`. Right? Their top-left corners will all have the same `x`, they'll all have the same `height`, `width`, etc.

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

A `table` knows two things: 1) the `y` of its first (top-left-most) cell and 2) how much space to put in between each row (`offset`). From there, if just indicate the row of the `case_number` we want to fill out, the `table` can calculate the `y` for us using `Y_OF_TOP - (row * SPACE_IN_BETWEEN_INCIDENT_ROWS)`.

Last thing our table wishlist snippet does is make it possible to override the `offset` a column should use. So when we're calculating the `y` for a `classified_as_death` cell, instead of using `row * 16.5`, we'll use `row * 16.6`. Turns out those check boxes in that column have just a little more space in between each row, and it's surprisingly noticeable if we don't adjust the `offset`:

<div class="img-scaled height-down-25">

| `offset = 16.5` | `offset = 16.6` |
| --------------- | --------------- |
| ![checkboxes bad]({{ site.github.url }}/public/images/2020-07-18/checkboxes_with_bad_offset.png) | ![checkboxes good]({{ site.github.url }}/public/images/2020-07-18/checkboxes_with_good_offset.png) |

</div>

Ok, that's everything...that we...uh..._wish_ we could do, lol 🌠 Let's make it work!

### Making it work

Right now, we have a DSL of sorts for defining text boxes we want to draw on a blank PDF. To make it work, we're gonna create a concern called `PDF::Layout`. But first, a design constraint: whenever we go to fill in cell, let's make that call a distinct method. For example, filling in the `establishment_name` should call `fill_in_establishment_name(name)`. If we get an error, the field that caused it should be easily discoverable in the stack trace.

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
        with_color(stroke: "#4299e1") do
          pdf.stroke_rectangle(_options[:at], _options[:width], _options[:height])
        end
      end

      pdf.text_box(value.to_s, _options)
    end
  end
end
```

First, `fill_in` takes the `options` it got and calls any of the callable values giving them those same `options`. That let's us take an option like `{ font_size: ->(options) { options[:height] } }` and resolve it to `{ font_size: whatever_height_is_set_to }` just before we finally ready to write to the PDF. If you remember, in one of our very first "wishful thinking" code snippets, we passed a lambda like that to `default_cell_font_size`. Now you know why.

Second, we turn our `x` and `y` options in to a single `at` option that takes them as an array, and we turn `font_size` into `size`. This allows us to have a slightly nicer API than `prawn`'s `text_box` method -- `size` is a confusing option next to `height` and `width`, and for us its useful to break out `x` and `y` separately from each other.

Lastly, we call `pdf.text_box` passing along those options.

<div class="message">
  <div class="message-body">
But what about `outline_text_boxes?` and `with_color` I hear you asking. If you're going to use something like this for your own PDF forms, it's insanely helpful to outline the text boxes for each cell, but you of course don't want to do that for reals. You can imagine one definition might be:

```rb
def outline_text_boxes?; Rails.env.development?; end
```

`with_color` does exactly what you might think -- the implementation is [here]().
  </div>
</div>

Great, that's the final stop of our DSL -- we're kinda working backwards. Let's add what we need to make our DSL methods work. Next up, the `default_cell_x` methods:

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

      def default_cell_min_font_size(value)
        self.defaults[:min_font_size] = value
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

That wasn't too bad. Let's try something ~~harder~~ Ruby-ier. If you look back at our `PDF::Form300` class we had code like this:

```ruby
class PDF::Form300
  cell_type :field, font_size: 7

  field :establishment_name, x: 658, y: 465, width: 110, height: 12
end
```

Do you see what happened there? After calling `cell_type :field` we have access to a newly-defined class method `field`. So calling `cell_type :field` needs to define `field` on the class:

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
          type_defaults                                         # 1
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

That covers everything but `table`. If you remember our `table` example it looked like this:

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

As we've just seen, normally calling `field :case_number` would create a `fill_in_case_number` method that takes just a `value` to write into the PDF. But notice `table` takes a block. Inside the block, we're gonna make it so calling `field :case_number` creates a `fill_in_case_number` that _not only_ takes a `value` argument, but a `row` keyword argument too. It'll then use that `row` argument, the table's `y`, and the table's `offset` to calculate the `y` for the cell.

Ok, let's start with just the `table` class method:

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

Ignoring the `Table` class we haven't seen yet, the `table` method's not too bad. It creates a new `Table` passing along `self` (the class that has included the `Layout` concern and called `table` -- `PDF::Form300` in our case), `y`, and `offset` to the new `Table` instance. Then it asks the `Table` instance to evaluate the block.

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

3. (Lines with `#3`) -- And this section does just that. It's nothing we haven't seen before. Assuming we had called `field :case_number` inside our `table` block, we're still using `define_method` to create an instance method called `fill_in_case_number`, and when that's called passing a `value` and our merged options along to the `fill_in` method.

    This time the block we pass to `define_method` is closed over the `y` and `offset` we need to calculate the cell's `y` using the `row`, as well as the `defaults_for_cell`. `fill_in_case_number` then provides that cell's `y` as an option to the `fill_in` method.

Ok, that's it for our `table` method. With that, the DSL we wrote while we were writing the code we wished had should work!

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
  default_cell_min_font_size 0

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

I ❤️ this. If you go look at the [full class](https://github.com/gkemmey/filling_out_pdf_form_examples/blob/master/pdfs/form300.rb), you'll see there's about 60 lines of layout code at the top. That's 60 lines of code to layout 234 cells on this PDF. Then there's another 100 for tallying counts, iterating over collections, and calling all those `fill_in` methods with the right values.

That's not bad at all.

Changes to either the layout or the logic of filling in the cells are easy and separate. Also, this makes it simple to handle multiple PDFs.

Is the code that enables our PDF form DSL gross? Yeah. It really fucking is. But I think the API in our `PDF::Form300` is worth it. Regardless, hopefully it was a fun look at some wild Ruby 🦁

Anyway, you can see a full, working version of generating this OSHA Form 300 with dummy data [here](https://github.com/gkemmey/filling_out_pdf_form_examples), as well as here are direct links to the [concern](https://github.com/gkemmey/filling_out_pdf_form_examples/blob/master/pdfs/layout.rb) and [pdf class](https://github.com/gkemmey/filling_out_pdf_form_examples/blob/master/pdfs/form300.rb). Lastly, here's the <a href="{{ site.github.url }}/public/images/2020-07-18/osha_form_300_filled.pdf" target="\_blank">final, filled-out PDF</a> running that full version creates.
