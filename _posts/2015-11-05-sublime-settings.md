---
layout: post
title: Sublime Settings
permalink: sublime-settings
published: true
---

Mostly, this is for my own personal reference, but since I was forced to deal with it when I really wanted to write about something else, I decided to document. So, here's my current [Sublime Text 3](http://www.sublimetext.com/3) settings.

Obviously, you should be using [Package Control](https://packagecontrol.io/installation), so I won't waste any time on that.

The next thing you should use is the [Material](https://github.com/equinusocio/material-theme) theme. The `README` provides details on all the configurable options, but for simplicity here's my complete personal settings that I have found to look good. Note: some settings are related to the Material theme and others not:

{% highlight json %}
{
  "always_show_minimap_viewport": true,
  "auto_complete_commit_on_tab": true,
  "bold_folder_labels": true,
  "centurion_folder_icons": true,
  "color_scheme": "Packages/Material Theme/schemes/Material-Theme-Darker.tmTheme",
  "contrasted_quick_panel": true,
  "contrasted_sidebar": true,
  "contrasted_tabs": true,
  "find_selected_text": true,
  "font_size": 11.0,
  "icon_file_type_enable": true,
  "line_padding_bottom": 1,
  "line_padding_top": 1,
  "overlay_scroll_bars": "enabled",
  "preview_on_click": false,
  "rulers":
  [
    100
  ],
  "shift_tab_unindent": true,
  "soda_folder_icons": true,
  "tab_size": 2,
  "tabs_medium": true,
  "theme": "Material-Theme-Darker.sublime-theme",
  "translate_tabs_to_spaces": true
}
{% endhighlight %}

What slowed me down tonight was getting something reasonable for editing Markdown files working. I wasn't a huge fan of how Material worked with Markdown. So I tired [MarkdownEditing](https://github.com/SublimeText-Markdown/MarkdownEditing), but wasn't really a huge fan.

But, I did find [Markdown Extended](https://github.com/jonschlinkert/sublime-markdown-extended), which looked pretty good! However, it works best with the [Monokai Extended](https://github.com/jonschlinkert/sublime-monokai-extended) color scheme. And I didn't want to stop using Material. So, what I did was download both Markdown Extended and Monokai Extended, but only use the Monokai Extended color scheme in my `.md` files.

To do this, first, install the packages. Then open a `.md` file. Navigate to `Preferences > Settings - More > Syntax Specific - User` and add the following:

{% highlight json %}
{
  "color_scheme": "Packages/Monokai Extended/Monokai Extended.tmTheme",
  "spell_check": true
}
{% endhighlight %}

Monokai Extended blends almost perfectly with Material, and now Sublime Text will seamlessly switch your color schemes depending on the open file.

With that, you should be all set to use Sublime Text for all of your development work, as well writing for that Jekyll blog you've always meant to work on!