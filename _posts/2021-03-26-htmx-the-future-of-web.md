---
layout: post
title: Htmx &amp; The Future of Web Dev, Part 1
---

**Back in the day, the web was very simple. In this series, we'll explore a tiny library that can bring back yesteryear's simplicity to the modern web, without sacrificing the parts that make the modern web feel better to use.**

-----

The web used to be pretty straight forward. URLs pointed to servers, servers mashed up their data into html, and browsers rendered that response. What a glorious era that was. Hypertext was all we needed. Simple frameworks popped up around this simple paradigm, and allowed developers to go from spending months on basic functionalities to spending hours to create relatively complex projects. More time was spent on the *business logic*, on *application design*, instead of spending untold man hours wiring up scripts to databases and concatenating strings of html together.

I'm not going to tell you we should have stopped there. The web grew, and evolved, obviously, and we needed javascript for things.

That's where our story gets dark.

Javascript is out of control now. Somehow we've decided it's acceptable to throw massive applications at our users, force their devices to run those applications, and make constant, ever-increasing quantities of network requests while the application is in use. Somehow we've decided that in order to generate HTML, we have to use JSON (what?), make dozens of network requests, discard most of the data that we get in those requests, have an increasingly opaque black box of a javascript framework turn that JSON into HTML (via an arcane arrangement of multi-level state management, CSS as a new domain-specific language, and a new client-only url routing layer) and then hopefully correctly patch that new HTML into the DOM efficiently. (Spoiler alert: [it's not efficient](https://kentcdodds.com/blog/fix-the-slow-render-before-you-fix-the-re-render).)

I haven't even started in on the complicated nightmare of javascript build systems, especially that Lovecraftian horror known as Webpack, or the security vulnerability waiting to destroy your business in token-based authentication. Or the fact that the JSON pre-requisite of all of this means you have to serialize your data in order to serve it as JSON, which is not an easy problem to solve. Overserving, underserving, JSON has its own security vulnerabilities...

Did everyone collectively forget that we could render HTML on the server? Much faster, more consistently, adjacent to your actual application state, and without shipping any unnecessary data to your user's devices?

**But without javascript, we have to reload the page on every action!**

Sure, I understand that this is an initial motivation for these complex javascript frameworks. That is a very reasonable thing to expect of modern web applications. But we don't need to do all of the bespoke javascript nonsense to solve that problem.

[Enter Htmx.](https://htmx.org)

This is only one implementation of the idea that simple *is* the future of web development. The premise is simple.

1. Issue ajax requests from any user event (clicking links, submitting forms, clicking any element, issuing keystrokes)
2. Let the server generate the HTML that represents the new application state for that request.
3. Send **that html** in the response.
4. Shove that element in the DOM where it's supposed to go.

That's it. Dead simple. Every backend framework since the beginning of the web has allowed you to generate HTML. It's the fundamental functionality of a web application framework. It's been perfected over the last decade and a half. Why not let that application do what it's *meant to do*? Your application state (probably a database) is directly connected to your application. You probably even use an ORM, and a templating system. It's all batteries included with your web framework, so *stop fighting with it* and embrace it.

-----

In this series, I'm going to show you how to embrace your web framework. I'm going to use Python and Django, but I guarantee this idea is compatible with whatever backend framework you might want to use, because it's *just HTML*.

I'm not going to walk you through the steps of setting up a new Django project. If you want to follow along, and play with it, the [Django tutorial](https://docs.djangoproject.com/en/3.1/intro/tutorial01/) is excellent. Get that started, then come back.

Let's start with the most basic of trivial examples.

Let's create a pretty basic view. It's going to serve a little template with a button.

```python
# views.py
def home(request):
    return render(request, "home.html")
```

```python
# urls.py

urlpatterns = [
	path('', views.home, name='home')
]
```

```html
<!-- home.html -->
<html>
<head></head>
<body>
    <button>Click Me To Get a Blog Post</button>
</body>
</html>
```

*Disclaimer:* I'm not showing you everything. Again, this isn't a django tutorial. I expect you to pretty much grasp what's going on here without me showing you every import and line of code.

With this, if you've set up your project correctly, you can go to your new homepage and see a button. It doesn't do anything right now. Let's fix that. First, we'll set up a new view that just returns a blog post.

```python
# views.py
def blog_post(request):
    return render(request, "blog_post.html")

# urls.py
urlpatterns = [
	path('', views.home, name='home'),
	path('blog-post', views.blog_post, name='blog-post'),
]
```

```html
<!-- blog_post.html -->
<section id="blog-post">
    <h1>This is the title of my blog post!</h1>
    <p>This is the content of my blog post.</h1>
</section>
```

Now if you go to your new blog-post url, you should see the blog post. Nothing new so far. So now, let's wire this up with htmx. Let's start by adding the script tag to the home.html template to make sure htmx is available via it's CDN, and then we can start adding a little htmx magic via directives.

```html
<!-- home.html -->
<html>
<head>
  <script src="https://unpkg.com/htmx.org@1.3.2"></script>
</head>
<body>
  <button
    hx-get="/blog-post"
    hx-target="#blog-post"
    >
    Click Me To Get a Blog Post
  </button>
  <section id="blog-post"></section>
</body>
</html>
```

So, you can check out the [Htmx documentation](https://htmx.org/docs/) to see what I've done here, but the general idea should be obvious; issue a GET request to `/blog-post`, then take the response and replace the css selector `#blog-post` with the response.

![It's really this easy.](/assets/images/blogpostvid.gif)

If you're coding along, go ahead and try it. It feels almost magical. We've done in two simple, semantic directives what takes the javascript world a massive build pipeline, dozens of megabytes of application code, hundreds of megabytes if not gigabytes of npm dependencies, massive headaches of state management... Do I need to go on?

I know this is only a trivial example. I know it's so trivial as to be almost meaningless. But that's because there's... not really any additional complexity in this paradigm. Make a request, get the response, put it somewhere in the DOM. You'd be really surprised at [what can be done](https://htmx.org/examples/) this way. In the next parts of this series, we'll explore what some of those are.
