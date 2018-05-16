---
layout: post
title:  "Data Tables in Vue.JS - Part 1, The Right API"
date:   2018-05-16 00:41:53 -0500
categories: Vue.js Components Design
---

A brief introduction: I am Lawrence Vanderpool, a full stack web developer at L7 Informatics, a father of two, and a big fan of Vue.js. Now that that's out of the way, let's talk about the mission statement of this small write-up.

My aim is to write a better Data Tables component. You can see examples of Data Tables components elsewhere on the web, but they usually come with the massive drawback of being inflexible and difficult to style. I aim to solve that problem by leveraging Vue's fantastic scoped slots, render functions, and component composition. The goal of this series is to write a component that is powerful, flexible, intuitive, and leaves as *much of the markup as possible to the end user*. This is very important to me as a developer, and it may be very difficult, but I believe it's a goal worth chasing.

My work here is somewhat investigative. I haven't used most of these powerful Vue features before, and to be honest, my experience in Javascript is rather two-dimensional. I enjoy working in ES6 but I wouldn't call myself an expert; I can write code that solves problems, but it's rarely elegant. With this project, I am to address all of those things, and set myself up for a bigger open source project that I have planned for later this year.

Onto the topic of this Part 1: What should the API of the Data Table look like? I will be taking an iterative approach to solving this problem and writing this component library. I will take a certain feature or set of features, determine what the API should look like for those features, then write code and tests to solve those problems. We will continue to add features, refactor, rewrite tests, and hopefully at the end of this process, come up with something that looks like a component people would choose to use in their own projects.

What does the basic API of a reusable Data Table component look like? Well, it'll need to take in a set of rows, and then a set of column values with human readable formats to pair them with. That will at least give us a reusable table component. We'll have to parse the columns, which will have to include a field name and a label. But that comes later. One step at a time. What does this first step look like to an end user?

{% highlight vue %}
<template>
  <data-table :rows="rows" :cols="cols"></data-table>
</template>
{% endhighlight %}

Well, that wasn't so hard. We can probably provide a default rendering that's just `<table>`s, `<tr>`s, and `<td>`s with no classes or anything, and the user could import that in their own components and apply scoped CSS to that. It's really not a bad start.

One of the premises of this project is to make as few assumptions as possible, but as far as I can tell it's unavoidable that the user will have to supply us with information about the fields to display and the human readable names for those fields in the form of the `cols` prop.

{% highlight javascript %}
const cols = [
  { fieldName: "animalSpecies", label: "Species" },
  { fieldName: "color", label: "Color" }
]
{% endhighlight %}

Rows can be an array of whatever the user has; so long as the field names passed into `cols` are present in the objects in the array.

As few assumptions as possible.
