---
layout: post
title:  "Dry titling in Rails"
date:   2012-12-27 16:01:57
tags: [jekyll, update]
---

A typical problem in many page-based web apps is to show page titles. I've found that most every app wants this title in the document title, and only sometimes as a h1-tag somewhere in the body.

<!--more-->

My relatively DRY pattern consists of using a title content block.

```ruby
module ApplicationHelper
  def title(str=nil)
    content_for(:title){ str } unless str.nil?
  
    content_for(:title)
  end
end
```

This allows for setting of titles at the top of views, like so:

```haml
- title "A magnificent title"
```

Allows for passing it to the head title like so:

```haml
%head
  %title= title
```

And in the view, you can insert the title with an almost identical call:

```haml
%h1= title
```

### Bonus: Adapting to Twitter Bootstrap's page header format

Twitter bootstrap comes bundled with a nice little formatting tweak for designated page titles, using the following format:

```html
<div class="page-title">
  <h1>
    A magnificent title
    <small>with a magnificent tagline</small>
  </h1>
</div>
```

Note that it even includes an optional tagline. Nice!

Printing the title in this format is simple to provide in a helper method. We'll call it `print_title`.

```ruby
module ApplicationHelper
  # ...
  
  def print_title(tagline=nil)
    title_text = content_for(:title)
    
    # Add the optional tagline
    title_text += content_tag('small', " " + tagline) unless tagline.nil?
    
    # Return a h1 wrapped in a div.page-header
    content_tag('div', content_tag('h1', title_text), class: 'page-header')
  end
end
```

Now you should be able to DRY up the fancy titling in your views. Take this blog post as an example, with its category title as tagline:

```haml
- title @post.title

= print_title @post.description

- # ...
```

This pattern of calls should be simple enough to be adapted to most situations.

**If you've got a radically different way of doing this, I'd appreciate some feedback in the comments. Thanks!**

