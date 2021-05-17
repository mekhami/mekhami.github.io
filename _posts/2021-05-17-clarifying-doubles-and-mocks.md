---
layout: post
title: Clarifying Doubles &amp; Mocks
---

**Much has been said about testing your system boundaries via mocks, doubles, stubs, fakes, and the like. I want to try to break down the difference between the two and explain them in context.**

-----

Let us begin by defining what mocking is and what test doubles are. In this article, we are going to generally stay within the realm of automated testing.

Mocking means **altering the runtime behaviour** of a piece of code, at runtime. Here's a trivial example:

```python
import requests
from unittest.mock import patch
def make_http_request():
    return requests.get('https://example.com')

def test_make_http_request():
    with patch('requests.get') as mock_http:
        mock_http.return_value = 'Hello world!'
        assert make_http_request() == 'Hello world!'
```

Dead simple. We patch the function to return a new value ('Hello World!') instead of what it would normally return, which is the markup that makes up example.com's homepage.

Ninety-nine percent of the time in 'the real world', this is how you're going to see mocks use. Take a system boundary that is unreliable, or changes often, one that is outside of your control, and give it an explicit return value so that you have total control over it.

Mocking, then, is a technique by which we control the execution of our code when involving other systems that we do not have control over ordinarily.

In doing this, we have created a **double**. Just like a body double, this double steps in during our tests to emulate the behaviour of the thing it is doubling. Most of the time, though, in reading articles and blog posts like this one on the web, you will find that mocks are not considered doubles, that doubles (and fakes and stubs) are some other classification of things in our tests.

This is technically incorrect. Practically speaking, we do mean something else when we say "double" most of the time, but technically, a double is a *concept* and a mock is the *technique* used to implement that concept.

# How mocks are typically (ab)used.

Mocking is the easy way to control the behaviour of systems outside your control. Unfortunately, with great power comes typical irresponsibility in the hands of the software development community at large.

The problem occurs when people *mock* as a *verb*. As in the example above, when we *mocked* the return value of a third party tool so we would have control over it, we were mocking as a verb. This tightly couples our test to the implementation details of the thing we've patched, in this case the `requests` library. While the requests library has a pretty stable API and this is not a cause for great concern, it is a coupling, and it is not impossible for `requests` to change. And, more often than you might imagine, you end up mocking and patching system boundaries that aren't as stable and predictable.

This coupling between test and code means that any time the implementation of `make_http_request` changes, i.e. changing to a POST instead of a GET, using an entirely different library for doing our http fetching, etc., we have to also change **each and every test that uses it**. Not because the code has changed, nor because the code was broken before, but because the technique we used to implement the tests requires us to. That coupling of test to implementation details creates a long term maintenance nightmare. Imagine you have a test suite with 5000 tests in it, and 100 tests are calling this function and mocking the return value in this way. Do you want to be the developer that tackles this issue? I absolutely do not.

The alternative is to **provide a mock, as a noun.** Let's reconfigure our example in this way.

```python
from unittest.mock import patch

def make_http_request():
    return WebsiteFetcher.get('https://example.com')

    def test_make_http_request():
        with patch(WebsiteFetcher) as mock_fetcher:
            make_http_request()
            mocked_fetcher.get.assert_called_once()
```

This test creates a mock `WebsiteFetcher`, as a noun, and replaces it (globally) over whatever you've actually implemented for `WebsiteFetcher`. This is also a test **double**. This is slightly preferrable to the mocking example above, but only semantically. The behaviour is the same, and the test looks pretty much the same. We haven't really improved our tests or our code yet. It does have one tangible benefit in code quality, which is that we have a double of *something that we own* as opposed to mocking the functionality of *something we do not own*. Changes made to our code can propagate through our tests; changes made to third party code typically cause failure and then we have to go back and fix it, sometimes at great cost.

There's one other massive downside to using mocks like this. When you use `patch` in this way, you're **altering the runtime globally.** This makes your test inherently not-threadsafe. You no longer have the option to run your tests in parallel (at least not successfully.) You've altered the runtime execution of `requests.get`, and if you mock it differently somewhere else, or don't mock it at all in some other test, you have conflicting definitions of what `requests.get` is supposed to return, and the last-applied patch is the one that will trump the rest when your test runner gets around to each of these tests. It doesn't matter what language or framework you use; if you alter the global runtime context, you cannot predictably execute that context in parallel.

I don't know about you, but altering the global state of the application does not feel like a recipe for success. There's another option though, and that's using proper doubles via dependency injection.

# Using Doubles (via Inversion)

There is a better way to provide your test doubles, and it uses the *principle* of Dependency Inversion.

I want to put a **big massive disclaimer** here, because I don't want people to get carried away. I am not explicitly talking about Dependency Injection frameworks or tools. Injection is a technique for accomplishing inversion, just as earlier we said that mocks were a technique for accomplishing test doubles. Please do not call to mind all the dependency injection tools (i.e. Angular) that you've used in the past. Chances are, it didn't feel that good, and became cumbersome and tedious. We will get into that more a little bit later.

<!--
The web used to be pretty straight forward. URLs pointed to servers, servers mashed up their data into html, and browsers rendered that response. What a glorious era that was. Hypertext was all we needed. Simple frameworks popped up around this simple paradigm, and allowed developers to go from spending months on basic functionalities to spending hours to create relatively complex projects. More time was spent on the *business logic*, on *application design*, instead of spending untold man hours wiring up scripts to databases and concatenating strings of html together.

I'm not going to tell you we should have stopped there. The web grew, and evolved, obviously, and we needed javascript for things.

That's where our story gets dark.

Javascript is out of control now. Somehow we've decided it's acceptable to throw massive applications at our users, force their devices to run those applications, and make constant, ever-increasing quantities of network requests while the application is in use. Somehow we've decided that in order to generate HTML, we have to use JSON (what?), make dozens of network requests, discard most of the data that we get in those requests, have an increasingly opaque black box of a javascript framework turn that JSON into HTML (via an arcane arrangement of multi-level state management, CSS as a new domain-specific language, and a new client-only url routing layer) and then hopefully correctly patch that new HTML into the DOM efficiently. (Spoiler alert: [it's not efficient](https://kentcdodds.com/blog/fix-the-slow-render-before-you-fix-the-re-render).)

I haven't even started in on the complicated nightmare of javascript build systems, especially that Lovecraftian horror known as Webpack, or the security vulnerability waiting to destroy your business in token-based authentication. Or the fact that the JSON pre-requisite of all of this means you have to serialize your data in order to serve it as JSON, which is not an easy problem to solve. Overserving, underserving, JSON has its own security vulnerabilities...

Did everyone collectively forget that we could render HTML on the server? Much faster, more consistently, adjacent to your actual application state, and without shipping any unnecessary data to your user's devices?

**But without javascript, we have to reload the page on every action!**

Sure, I understand that this is an initial motivation for these complex javascript frameworks. That is a very reasonable thing to expect of modern web applications. But we don't need to do all of the bespoke javascript nonsense to solve that problem.

[Enter Htmx.](https://htmx.org)

This is only one implementation of the idea that simply *is* the future of web development. The premise is simple.

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
<section id="blog-post">
    <h1>This is the title of my blog post!</h1>
    <p>This is the content of my blog post.</h1>
</section>
```

Now if you go to your new blog-post url, you should see the blog post. Nothing new so far. So now, let's wire this up with htmx. Let's start by adding the script tag to the home.html template to make sure htmx is available via it's CDN, and then we can start adding a little htmx magic via directives.

```html
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
-->
