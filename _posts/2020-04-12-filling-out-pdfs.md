---
layout: post
title: 'Filling out PDFs'
published: true
---

A req came across my desk to fill out a PDF using data our app already has / collects. Specifically, these ones: [https://www.osha.gov/recordkeeping/new-osha300form1-1-04-FormsOnly.pdf](https://www.osha.gov/recordkeeping/new-osha300form1-1-04-FormsOnly.pdf).

Luckily, I was ready. A while ago, I had saved a reddit post where `u/CaptainKabob` was talking about how they had done this using a fillable PDF and "[`pdf-forms`](https://github.com/jkraemer/pdf-forms) (a gem that wraps [pdftk](https://www.pdflabs.com/tools/pdftk-the-pdf-toolkit/))". After a half a day or so of trying to convert a non-fillable PDF to a fillable PDF using Libre Office, briefly learning about [XFA and AcroForm](https://appligent.com/what-is-the-difference-between-acroforms-and-xfa/) fillable-form standards, trying to install `pdftk` on macOS, finding [this](https://stackoverflow.com/questions/39750883/pdftk-hanging-on-macos-sierra/39814799#39814799) unpublished link, learning about the [Java rewrite](https://gitlab.com/pdftk-java/pdftk) of `pdftk` because it doesn't work on Ubuntu > 18, and trying unsuccessfully to install that on my mac, I'm here to tell you fuck. that. approach. It mighta worked for `u/CaptainKabob`, but it's fucking horse shit 🐴

Instead, I'm here to show you the absolute, #1 way to fill out an existing PDF -- any PDF, fillable-form fields or not -- and it comes to us from that very same reddit post with an unassuming comment from `u/hcollider` ([link](https://www.reddit.com/r/rails/comments/8ohntl/generating_pdf_form_with_prawn/e03k552)):

![hcollider's reddit comment]({{ site.github.url }}/public/images/2020-04-12/hcolliders_reddit_comment.png)

Doubt not good sir!

I've done PDFs before, with all the usual suspects -- [`wkhtmltopdf`](https://wkhtmltopdf.org/) and [`wicked_pdf`](https://github.com/mileszs/wicked_pdf), [`combine_pdf`](https://github.com/boazsegev/combine_pdf), [`prawn`](https://github.com/prawnpdf/prawn) -- and nothing beats this!

Here's how it works:

1. We use `prawn` for making a brand new PDF with just the form fields filled out on white paper. It's text boxes and text with no background floating in white space.
2. We the use `combine_pdf` to lay those answers on top of the original PDF, and save that as a new PDF.

That's it! Just Ruby libraries. No binary package dependencies. Simple and elegant 👌

## Meet the tools 🛠

### prawn

`prawn` let's us create PDFs from text, shapes, and images by drawing on a coordinate plane where `(0, 0)` is the bottom-left-hand corner of a page. For example, this code generates <a href="{{ site.github.url }}/public/images/2020-04-12/rectangle.pdf" target="\_blank">this</a> PDF:

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

Even more important to our use case of filling out a form, is `prawn's` text utilities. Checkout this example which generates <a href="{{ site.github.url }}/public/images/2020-04-12/text.pdf" target="\_blank">this</a> PDF:

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

Here is an image of that PDF, because it's important:

![text pdf as png]({{ site.github.url }}/public/images/2020-04-12/text_pdf_as_png.png)

So we can draw text boxes by specifying the top left corner (`at`), its `width`, its `height`, and optionally what to do with text that doesn't fit (`overflow`). You see that third `overflow` mode, `shrink_to_fit`? That's gonna be useful when we're filling out our forms 😄

### combine_pdf

Unsurprisingly, `combine_pdf` let's us...combine PDFs. For our purposes, it's the magic that lets us take the filled out content we generated with `prawn` and lay it on top of the original form. Let's take a look at an example that draws a grid on our form, like <a href="{{ site.github.url }}/public/images/2020-04-12/osha_form_300_with_grid.pdf" target="\_blank">this</a> PDF:

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

That example 1) shows all the pieces and 2) is incredibly useful for figuring out where to draw our text boxes on the real thing.

## A compete example 📝

### Hold up a second

I'm gonna show you what I did to fill out these OSHA forms. I think it's cool, but there's arguments for it being overcomplicated 🤷‍♂️ It did make it easier to do the next one though. Anyway, you don't have to keep reading -- that grid example has everything you need. At the end of the day, it's just calls to that `text_box` method with options for where to draw it on the paper like so:

```ruby
pdf.text_box "message", at: [148, 360.0],
                        width: 40,
                        height: 14,
                        valign: :bottom,
                        overflow: :shrink_to_fit,
                        size: 7
```

And examples from `combine_pdf's` [README](https://github.com/boazsegev/combine_pdf#add-content-to-existing-pages-stamp--watermark) to save the two PDFs as one. That's it!

I've been doing this web dev thing for a while now, and I can't believe this is a PDF solution I'm just now hearing about. To think it was just buried in a reddit post!

### Ok, let's over engineer 🤖

Looking at that OSHA Form 300, you can see there's lots of fields on there you're going to want to fill out the same way -- i.e. share styles. Sharing styles, as far as `prawn` is concerned, is really just options passed to `text_box`.

So that's what I wanted, some way to communicate where cells (or fields) on the PDF are and share the like styles, so we're not repeating options to the `text_box` method over and over. When I find myself in that position -- knowing roughly what I want, but not sure how to build it -- I like to start by writing the code I wish I had. Here's what I wrote:

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

That feels like we're getting somewhere! Well, ish. It doesn't work, but what we want is starting to take shape:

1. We have named individual areas of the form we need to fill out "cells"
2. We can set defaults that will apply to all cells using the `default_cell_x` methods
3. We can override those defaults with defaults that will apply to a type of cell using the `cell_type` method
4. We can use those "cell types" to identify the actual parts of the form we need to fill in, where they're located, and override any of those shared defaults on a per-cell basis

Let's keep going:

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

  page_total :classified_as_death_page_total, x: 476 # ✨new
  page_total :resulted_in_injury_page_total,  x: 680, width: 10 # ✨new
end
```

Here's those page totals for reference:

<div class="shadow-md">
![page totals]({{ site.github.url }}/public/images/2020-04-12/page_totals.png)
</div>

How clean is that?! We can share all those styles and the `y` position, and just specify the `x`! Ok, and change the width for the smaller ones 😬

The last thing I wanted to be able to do was describe a table. Each page of the form contains essentially a table of incidents. Each cell in that table not only shares styles, but also positioning (at least relative to the top-left cell). Here's the code I wrote for that:

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

We're going to fill out a `case_number` field for each incident we write on the form, but we only defined that cell once. This works because a `table` knows where the first row is (`y`) and the space between each row (`offset`). Positionally, for each field we can give it the `x` and let the table calculate the `y` using `Y_OF_TOP - (row * SPACE_BETWEEN_ROWS)`.

Last thing, checkout the setup for that `classified_as_death` checkbox. We adjusted the amount to offset in between each row by just a little a bit. That's just because the form isn't pixel perfect and the checkboxes weren't spaced out perfectly consistently with everything else. What's cool is we can override the table's `offset` value per cell -- if we need to.

Ok, up to this point, we've just ben writing the code _we wished we had_. Let's make it work! #wdd #wishlistdrivendevelopment

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

Lulz, what the fuck even is Ruby? Don't worry! If you'll allow my crude annotations we can take this in pieces:

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
