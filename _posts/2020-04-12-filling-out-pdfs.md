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

So we can draw text boxes by specifing the top left corner (`at`), its `width`, its `height`, and optionally what to do with text that doesn't fit (`overflow`). You see that third `overflow` mode, `shrink_to_fit`? That's gonna be useful when we're filling out our forms 😄

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

I've been doing this web dev thing for a while now, and I can't believe no one's shared this with me.
