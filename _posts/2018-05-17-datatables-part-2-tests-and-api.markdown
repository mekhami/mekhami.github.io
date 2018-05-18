---
layout: post
title: Data Tables in Vue.js - Part 2, Testing and API Thoughts
date: 2018-05-17 22:15:14 -0500
---

I started this session with a very basic starter test [(commit: 2513f51)] to make sure I wasn't absolutely insane and to give me a better place to start iterating. Generally, I use a test-first strategy so that I have some very immediate feedback for when things go wrong. It also allows me to focus up front on the API of my the components.

Speaking of component APIs, it occurs to me that it's going to be very hard to let the end user dictate the markup. How will it be possible to let them use whatever markup they want? What if they just refuse to use `<tr>` and `<td>`? I'm not sure going forward how much we'll be able to do. But it's the entire reason for this component, so we'll have to make it work somehow.

What actually goes inside our data table component? We'll probably need a slot for the end user to override the default header, the default footer, and the default row.

{% highlight vue %}
<template>
  <data-table :rows="rows" :cols="cols">
    <data-table-header>
      <thead>This table will look pretty silly I think.</thead>
    </data-table-header>
    <data-table-row>
      <tr>I refuse to render tds in my code</tr> 
    </data-table-row>
    <data-table-footer>
      <h1>This... isn't how tables work at all</h1>
    </data-table-footer>
  </data-table>
</template>
{% endhighlight %}

What's above will simply not be functional, but someone somewhere will want to mess things up.

The most important thing I want to be able to override is individual columns within a row. For instance, in many tables that I've needed, the first column is the name of the thing in that row, and it needs to link off to some detail view for that item. The solution I've had to use in most existing libraries (and things I've written myself) is ...

{% highlight vue %}
<td v-if="row[col] === 'name'">
  <a href="#">{% raw %}{{ row[col] }}{% endraw %}</a>
</td>
<td v-else>
  {% raw %}{{ row[col] }}{% endraw %}
</td>
{% endhighlight %}

But you can imagine that this gets unwieldy when we want to render a few things differently. Lots of ifs and else ifs and it's not easy to follow. I want to override JUST that cell, and I'd like to do it with slots.

{% highlight vue %}
<data-table-row>
  <td slot="name" slot-scope="{ row }">
    <a href="/detail/view">
  </td>
</data-table-row>
{% endhighlight %}

So that's what we'll work on now. The `<data-table-row>` component would need to render a default value of `<td>{{ row[col.value] }}</td>` which means it needs to take a row as a prop. But rather than taking all the cols as a prop (again), we can 'inject' that from the parent `<data-table>` component.

I stumbled across [this handy Jsfiddle] that demonstrated how this might be possible. Hell, we may not even need a `<data-table-row>` component after all.

{% highlight vue %}
<template>
  <table>
    <thead>
      <tr>
        <td v-for="col in cols" :key="`${col.fieldName}-head`">
          {% raw %}{{ col.label }}{% endraw %}
        </td>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(row, index) in rows" :key="`row-${index}`">
        <template v-for="col in cols">
          <td v-if="typeof $scopedSlots[col.fieldName] !== 'undefined'" :key="`${col.fieldName}-body`">
            <slot :name="col.fieldName" :col="col.fieldName" :row="row"></slot>
          </td>
          <td v-else :key="`${col.fieldName}-body`">
            {% raw %}{{ row[col.fieldName] }}{% endraw %}
          </td>
        </template>
      </tr>
    </tbody>
  </table>
</template>

<script>
export default {
  name: 'DataTable',
  props: {
    cols: {
      type: Array,
      required: true
    },
    rows: {
      type: Array,
      required: true
    }
  },
  data () {
    return {}
  },
  methods: {}
}
</script>
{% endhighlight %}

That actually does what we expect it to do. Not bad for a day's work. We've got conditional rendering without the v-ifs and v-elses, we've got easily customizable individual fields. What we want next is row sorting based on a selected column, by clicking on the `thead>tr>td` element. We'll tackle that in the next article.

[(commit: 2513f51)]: https://github.com/mekhami/datatables/commit/2513f5185185c06b70718c14d439544af284fe79
[this handy Jsfiddle]: https://jsfiddle.net/pyrello/odwag9mx/
