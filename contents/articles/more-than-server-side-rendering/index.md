---
title: "More than Server-Side Rendering"
author: JC Ivancevich
date: 2015-07-15
template: article.jade
---

**Server-Side rendering** *(SSR)* in the world of [SPA](https://en.wikipedia.org/wiki/Single-page_application) frameworks is being a trendy topic these days. I think it's been thanks to [React](https://facebook.github.io/react/), which has *SSR* support out-of-the-box and makes it very easy, because of the [Virtual-DOM](http://tonyfreed.com/blog/what_is_virtual_dom). Also, the new version of [Ember](http://emberjs.com/blog/2015/01/08/inside-fastboot-faking-the-dom-in-node.html) is going to support *SSR* and even [Angular 2](https://www.reddit.com/r/angularjs/comments/2zowc4/angular_2_supports_serverside_rendering_and/) will have it (hopefully).

## But, is it enough? Is it only about [SEO](https://en.wikipedia.org/wiki/Search_engine_optimization)?

I think **rendering the first request on the server and then turning into a regular SPA** is great. **It helps for SEO**: the first download has actually data and not only useless templates. **It speeds up the perceived performance**: the user is able to see the content while the scripts, styles and templates are being downloaded in the background. And may be a few other things.

## So, there's no reason not to do SSR. But, **we can do better**.

Let me ask a few more questions:

- What happens if **the scripts were not completely downloaded when the user tries to interact** with your shinny server-side rendered page?
- Ok, they are downloaded, but if **they explode**?
- And if the user is a security obsessed person and has **JS disabled**?

Well, there's not much we can do with frameworks like the mentioned above *(I like to call them "**mainstream frameworks**")*.

But, in fact, there's something we can do: **regular web apps** (YES! the ones with redirects and flash messages).

## Are you saying that we should go back to the '90s?

Not really, but we can **build on top of that**. We can add **progressive enhancements** features on top of regular web apps. And guess what? There's a framework that does most of the work for you: [Taunus](http://taunus.io/)

**Taunus** allows *SSR* plus a couple of interesting features:

- Works with both [Express.js](https://github.com/taunus/taunus-express) and [Hapi.js](https://github.com/taunus/taunus-hapi)
- Lets you use the templating language of your choice
- Makes it easy to build apps that don't depend *(100%)* on client-side JavaScript
- Adding real-time is a cinch

Let's deep dive into **Taunus greatest features**. To do so, we're going to explore some of the [Taunus TodoMVC](https://github.com/taunus/taunus-todomvc)'s code.

---

### Building SPAs that don’t depend on client-side JavaScript

Ok, let's forget about client-side JavaScript for a bit, and go back to the '90s. How would you build a web app back then?

Easy peasy, you'd have a server-side powered app with some of these characteristics:
- Responses will be **HTML instead of JSON**.
- You'd use the `GET` method to retrieve a page and the `POST` method to submit a form. No `PUT`, `PATCH` or `DELETE`.
- After form sumission, you'd respond with **[302](https://en.wikipedia.org/wiki/HTTP_302) to redirect** the user to a different page.
- And probably, **flash messages** will be used to let the user know about fields that were filled wrongly, among other things.

#### A `GET` request in depth

Here's part of a template that would be rendered on the server:
```html
<ul class="filters">
  <li>
    <a {{#all}}class="selected"{{/all}} href="/">All</a>
  </li>
  <li>
    <a {{#active}}class="selected"{{/active}} href="/active">Active</a>
  </li>
  <li>
    <a {{#completed}}class="selected"{{/completed}} href="/completed">Completed</a>
  </li>
</ul>
```

Notice the `href` attribute will be in charge of navigating to different pages. Nothing new, ha? That's exactly how a *non-SPA* web application would work.

In the server, you'd have some router that takes care of these routes. Here's how they look in Taunus:
```javascript
// app.js

var app = require('express')();
var taunus = require('taunus');
var taunusExpress = require('taunus-express');
var options = {
  // templates must be pre-compiled
  // into `.bin/views/` in this case.
  layout: require('./.bin/views/layout'),
  routes: [
    { route: '/', action: 'todos/index' },
    { route: '/active', action: 'todos/index' },
    { route: '/completed', action: 'todos/index' }
  ]
};

taunusExpress(taunus, app, options);
```

The `action` for a `route` is just the path to the controller that's going to handle that route. In this case, all of the routes are using the same controller, which will be located in `controllers/todos/index.js`.

This controller is the responsible for providing the data with which the template will be rendered. Let's take a look at a simplified version of it:
```javascript
var todosService = require('./todosService');

module.exports = function getTodosController (req, res, next) {
  var currentPath = req.route.path.slice(1);

  todosService.getAll(getAllHandler);

  function getAllHandler (err, todos) {
    if (err) { return next(err); }

    var viewModel = {
      model: {
        all: currentPath === '',
        active: currentPath === 'active',
        completed: currentPath === 'completed',
        todos: todos
      }
    };

    res.viewModel = viewModel;

    next();
  }
};
```

Notice how we put the data to render the template into the `viewModel` property of the response object. You may be wondering where is the template this controller should render. And also, what happens when `next()` is executed.

Remember the `action` for this route was `todos/index`?
Well, the template will be placed in `views/todos/index.html` then. So, when `next()` is called, the template is rendered with the data in `res.viewModel` and the user gets the nice looking HTML page.

> Note: the template is inserted into the layout template (`views/layout.html`) when rendered. This is how the layout template should look:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Taunus • TodoMVC</title>
  <link rel="stylesheet" href="/css/styles.css">
</head>
<body>
  <main>{{{ partial }}}</main>
</body>
</html>
```

#### A `POST` request in depth

Here's another template. Also being rendered on the server:
```html
<form action="/form/todo" method="post">
  <input name="title" class="new-todo" placeholder="What needs to be done?" autofocus>
</form>
```

Notice the `action` and `method` attributes of the `form`. That's all we'd need to do in order to create a new TODO. Again, no client-side JS here!

So, what's going to happen when the user submits the form? The browser is going to fire a `POST` request, and we'll need an endpoint (`POST /form/todo`) to be able to handle it.

Somewhere in our code we need to create an Express endpoint, more or less like the following:
```javascript
var url = require('url');
var app = require('express')();
var taunus = require('taunus');
var todosService = require('./todosService');

app.post('/form/todo', createTodo);

function createTodo (req, res, next) {
  todosService.add(req.body, addHandler);

  function addHandler (err, todo) {
    if (err) { return next(err); }

    var referer = url.parse(req.headers.referer).path;
    taunus.redirect(req, res, referer);
  }
}
```
> Note: don't forget to add the `body-parser` Express middleware to be able to access the `req.body` property.

What the endpoint above does is super simple. It just receives the new TODO in the request body, and adds it using the `todosService`. Once this is done, it redirects back to the referer path using `taunus.redirect`. So, a new `GET` request will be fired, and the page is re-rendered.


Ok. At this point we only have a regular web app, which could be powered by [Rails](http://rubyonrails.org/), [Django](https://www.djangoproject.com/), [Laravel](http://laravel.com/), or whatever instead. Now, it's time to fast-forward to present day and add some client-side JavaScript.

### **Here's when the Taunus magic happens!**

Let's create a file in `client/js/main.js` to bootstrap Taunus client-side.

```javascript
var taunus = require('taunus');

// the wiring file is the result of running `taunus --output`.
// it just requires all the necessary files for Browserify.
var wiring = require('../../.bin/wiring');

var main = document.getElementsByTagName('main')[0];

taunus.mount(main, wiring);
```

Browerify it:
```bash
browserify client/js/main.js -do .bin/public/js/all.js
```

And include it in the HTML:
```html
// views/layout.html

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Taunus • TodoMVC</title>
  <link rel="stylesheet" href="/css/styles.css">
</head>
<body>
  <main>{{{ partial }}}</main>
  <script src="/js/all.js"></script>
</body>
</html>
```

> Note: remember to serve the files inside `.bin/public/`. You may use the `serve-static` Express middleware.

Just by doing so, **we've transformed our regular web app into a SPA**.

How? What's going to happen from now on?

Once Taunus is mounted on the client:
- **Navigating** to routes will use the **[History API](https://developer.mozilla.org/en-US/docs/Web/API/Window/history)**, and data will be retrieved using **AJAX**. And, because **Taunus** knows that we just want JSON, it **will respond with JSON** instead of HTML.
- **Submitting forms** will also be done **using AJAX**. And, on the server, `taunus.redirect` will respond with a JSON document (instead of a `302`) which will tell the Taunus router it has to navigate to some route.

Apart from that, we'll be able to create client-side controllers for each page. Here it's a silly example:

```javascript
// client/js/controllers/todos/index.js

var $ = require('dominus');

module.exports = function (viewModel, container, route) {
  var btn = $('.btn', container);
  btn.on('click', onBtnClick);

  function onBtnClick (event) {
    alert('Button clicked!');
  });
};
```

As well as having controllers for specific parts of a template (think of these as **directives** or **components**) using the [Taunus Actions](https://github.com/taunus/taunus-actions) plugin.

Also, **Taunus can handle flash messages and real-times updates**, among other things. But I'll let those things for a future blog post, since this one is quite extensive I think.

See how easy it is to add **progressive enhancements on top of regular web apps**? It's all about **[stop breaking the web](http://ponyfoo.com/articles/stop-breaking-the-web)**.

> Now, it's time for you to try [Taunus](http://taunus.io/). Also, visit the [Taunus TodoMVC](http://todomvc.taunus.io/) implementation and turn off JavaScript in your browser to see what happens!
