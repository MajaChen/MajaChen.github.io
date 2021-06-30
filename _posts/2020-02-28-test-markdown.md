---
layout: post
title: Sample blog post
subtitle: Each post also has a subtitle
gh-repo: daattali/beautiful-jekyll
gh-badge: [star, fork, follow]
tags: [test]
comments: true
---

emsp;&emsp;在实习的后半期，我也有心想改这个缺点，但是苦于"不知道笔记怎么做"，这笔记不写则已，一写就又臭又长，搞得自己写的很累(记得有一次花了整整三个工作日写了一篇页的文章)，写的这么长，可能根本没有人愿意看，这也是个问题。再回溯一下，20年研一下半段的时候，也兴冲冲地在博客园上写了好几十篇文章，但是后面就放弃了，为啥？一个是当时沉溺于尚硅谷、黑马这些培训机构提供的学习视频中无法自拔，积累的知识都是些API性质的东西：Spark的这个API咋用，Hadoop的那个API咋调，导致博客不仅冗长，还没有实质性内容。(这里我想额外多说两点：1.其实这些东西在官方文档中已经不可能再详尽，也已经不可能再权威，你的博客“复刻”官方文档意义是什么？2.你都是一个硕士研究生了，就不能把什么Hadoop、Spark这些框架当作主要的学习目标了，而且要学直接去看官方文档得了，这个道理越早明白越好。下面是我总结的写文章的几个原则和方法(原则、方法不分家)。
This is a demo post to show you how to write blog posts with markdown.  I strongly encourage you to [take 5 minutes to learn how to write in markdown](https://markdowntutorial.com/) - it'll teach you how to transform regular text into bold/italics/headings/tables/etc.

**Here is some bold text**

## Here is a secondary heading

Here's a useless table:

| Number | Next number | Previous number |
| :------ |:--- | :--- |
| Five | Six | Four |
| Ten | Eleven | Nine |
| Seven | Eight | Six |
| Two | Three | One |


How about a yummy crepe?

![Crepe](https://s3-media3.fl.yelpcdn.com/bphoto/cQ1Yoa75m2yUFFbY2xwuqw/348s.jpg)

It can also be centered!

![Crepe](https://s3-media3.fl.yelpcdn.com/bphoto/cQ1Yoa75m2yUFFbY2xwuqw/348s.jpg){: .mx-auto.d-block :}

Here's a code chunk:

~~~
var foo = function(x) {
  return(x + 5);
}
foo(3)
~~~

And here is the same code with syntax highlighting:

```javascript
var foo = function(x) {
  return(x + 5);
}
foo(3)
```

And here is the same code yet again but with line numbers:

{% highlight javascript linenos %}
var foo = function(x) {
  return(x + 5);
}
foo(3)
{% endhighlight %}

## Boxes
You can add notification, warning and error boxes like this:

### Notification

{: .box-note}
**Note:** This is a notification box.

### Warning

{: .box-warning}
**Warning:** This is a warning box.

### Error

{: .box-error}
**Error:** This is an error box.
