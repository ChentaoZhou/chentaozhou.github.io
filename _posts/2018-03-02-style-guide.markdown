---
layout: post
title: Style Guide
image: 8.jpg
date: 2018-03-02 13:35:20 +0200
tags:
categories: guide
---
A paragraph looks like this — 现在的手机包括iPhone在内，都有不同的屏幕尺寸，同时还存在屏幕可以旋转的情况，所以在实现UI时我们一般不能去像AbsoluteLayout那样写死组件的位置，而是需要更灵活、适应性强的布局工具，来让我们的界面可以和不同大小和比例的屏幕适配。获取屏幕的尺寸再通过计算来设置绝对布局的数值也是一种方法，但是iOS为我们提供了更好的自动布局方式来实现对组件相对位置的控制。

***

#### Headings by default:

# H1 For example
## H2 For example
### H3 For example
#### H4 For example
##### H5 For example
###### H6 For example

{% highlight markdown %}
## Heading first level
### Heading second level
#### Heading third level
{% endhighlight %}

***

#### Lists

###### Ordered list example:

1. Poutine drinking vinegar bitters.
2. Coloring book distillery fanny pack.
3. Venmo biodiesel gentrify enamel pin meditation.
4. Jean shorts shaman listicle pickled portland.
5. Salvia mumblecore brunch iPhone migas.

###### Unordered list example:

* Bitters semiotics vice thundercats synth.
* Literally cred narwhal bitters wayfarers.
* Kale chips chartreuse paleo tbh street art marfa.
* Mlkshk polaroid sriracha brooklyn.
* Pug you probably haven't heard of them air plant man bun.

{% highlight markdown %}
1. Order list item 1
2. Order list item 1

* Unordered list item 1
* Unordered list item 2
{% endhighlight %}

***

#### Quotes

###### A quote looks like this:

> Never put off till tomorrow what may be done day after tomorrow just as well. — Mark Twain

***

#### Syntax Highlighter

{% highlight js %}
  $('.top').click(function () {
    $('html, body').stop().animate({ scrollTop: 0 }, 'slow', 'swing');
  });
  $(window).scroll(function () {
    if ($(this).scrollTop() > $(window).height()) {
      $('.top').addClass("top-active");
    } else {
      $('.top').removeClass("top-active");
    };
  });
{% endhighlight %}

***

#### Images

![]({{site.baseurl}}/images/2.jpg)

***

#### Videos

###### Youtube

<iframe src="https://www.youtube.com/embed/iWowJBRMtpc" frameborder="0" allowfullscreen></iframe>
