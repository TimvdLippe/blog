# My experience with the Polymer `carbon-route`

At the time of writing, the Polymer [`carbon-route`](https://github.com/PolymerElements/carbon-route) is in beta.
This element is designed to allow Polymer elements data-bind to the url of
the site.
This blog post will describe some of the issues I encountered and depict
solutions for said issues.

## 0. The `Page.js` approach

The first task was to replace the old [`page.js`](https://github.com/visionmedia/page.js)
routing mechanism used in the [Polymer Starter Kit](https://github.com/PolymerElements/polymer-starter-kit)
to the new `carbon-route` element.
The two routing solutions have a significant different approach to how routing
should be integrated into your application.
`Page.js` required to have a single [`routing.html`](https://github.com/PolymerElements/polymer-starter-kit/blob/v1.3.0/app/elements/routing.html)
which contained all mappings from the url to their respective internal `app.route`.

To give an example of what was required to add a new route to your application:

* In `routing.html`
```js
  page('/store', function() {
    app.route = 'store';
  });
```
* In `index.html`
```html
  <template is="dom-bind" id="app">
    <iron-pages attr-for-selected="data-route" selected="{{route}}">
      ...
      <section data-route="store">
      </section>
      ...
    </iron-pages>
  </template>
```

As you can see, we had to define the same route (`store`) multiple times. Once
for the `page(...)` call, once to set the `app.route` and once in the `iron-pages`.
This makes maintaining routes a error-prone task, as we had to perform the same
task several times in different parts of our applications.
Moreover, we completely separated the routing from the actual elements. This requires
us to open two files with separate logic (and possibly more if your application grows).
Lastly, all routes we define must also be added as section in the `iron-pages`.

The last point is important one, as nested pages where solved (with `page.js`) in the following way:

* In `routing.html`
```js
  page('/store/:name', function(data) {
    app.route = 'store-info';
    app.params = data.params;
  });
```
* In `index.html`
```html
  <section data-route="store-info">
    <user-element params="{{params}}"></user-element>
  </section>
```

As you can see, a separate `<section>` had to be created to allow a sub-page to be shown.
Not only do you lose the context of the previous `store` page, you are also required
to now beforehand the inner structure of your pages. This lead to a convoluted
`iron-pages` with complicated data-bindings to enable sharing information from a
"parent" page to a sub-page.

## 1. The first steps with `carbon-route`

The section above shows the old solution with `page.js` and points out several
problematic practices with building `Polymer` applications. Therefore I decided
to use the new `carbon-route` element in my application.

The very first task was to figure out how this element should be used. Previously,
we invoked Javascript functions which were mapping to values in our `app`. Now,
we have to use data-binding.

**(Note: the application source code I was prototyping with can be found [here](https://github.com/TimvdLippe/polymer-lazy-application))**

### 1 level deep: the `iron-pages` selector

Let's first define an `index.html` which has 3 routes: `home`, `store` and `users`.

```html
<html>
<head>
  <link rel="import" href="bower_components/carbon-route/carbon-location.html">
  <link rel="import" href="bower_components/carbon-route/carbon-route.html">
  <link rel="import" href="bower_components/iron-pages/iron-pages.html">
</head>
<body>
  <template is="dom-bind" id="app">
    <carbon-location route="{{route}}"></carbon-location>
    <carbon-route route="{{route}}" pattern="/:page"
                  data="{{routeData}}"></carbon-route>

    <iron-pages selected="[[routeData.page]]"
                attr-for-selected="data-route"
                fallback-selection="404">
      <section data-route="">
        Home

        <a href="/store">Visit the store.</a><br />
        <a href="/users">See the users.</a>
      </section>
      <store-page data-route="store"></store-page>
      <user-page data-route="users"></user-page>
      <section data-route="404">
        Oops you hit a 404.

        <a href="/">Head back home.</a>
      </section>
    </iron-pages>
  </template>
</body>
</html>
```

So what is going on here? First of all, we define a `<carbon-location>` which
provides us a `route`. This route contains information about the current url.
This `route` is then bound to the `route` property of `<carbon-route>`.
Then we define a pattern where we match on. The `:page` denotes that we define
a variable path segment. Lastly, bind the `data` property to `routeData`.

Now to the `<iron-pages>`. We bind its `selected` property to `routeData.page`.
The `page` field of `routeData` is because we defined our variable in the
`<carbon-route>` pattern as `:page`. If we would have called it `:data`, we would
need to use `routeData.data`.

Given you load this page at `localhost:8080/`. The `route` of `<carbon-location>`
would contain the path `/`. This is matched with `/:page` and sets `routeData.page`
to `""`. As a consequence, the first `<section>` is shown by `<iron-pages>`. Hooray,
we have a homepage!

Consequently, when we visit `localhost:8080/store` or `localhost:8080/users` we show
the `<store-page>` or `<user-page>` respectively. If we enter an unmatching url,
for example `localhost:8080/asdf`, `<iron-pages>` will show the 404 section because
we set `fallback-selection="404"`.

### 2 levels deep: the store page

Now that we are able to show a page matching a certain url, I dove 1 level deeper:
how to show a page inside another page. After some fiddling with various approaches,
I figured out a good solution. `<carbon-route>` exposes a `tail` property which
contains the remaining unmatched url segments. So how does it work?

We update `<carbon-route>` to
```html
<carbon-route route="{{route}}" pattern="/:page" data="{{routeData}}"
              tail="{{tail}}"></carbon-route>
```
and we update the `<store-page>` to
```html
<store-page data-route="store" route="{{tail}}"></store-page>
```

Inside `<store-page>` we define the following `<carbon-route>`
```html
<carbon-route route="{{route}}" pattern="/:store"
              data="{{storeData}}" tail="{{subRoute}}"></carbon-route>
```

What is going on here? We pass on the `tail` from the upper-level `<carbon-route>`
and pass it on via the `<store-page>` to the sub level `<carbon-route>`. This nesting
allows the inner `<carbon-route>` to match on urls like `localhost:8080/store/Amsterdam`
or `localhost:8080/store/new-york`. We can retrieve these values in the `storeData.store`
which are `Amsterdam` and `new-york` respectively.

As you can see, this did not require us to update the `<iron-pages>` in `index.html`.
We are therefore not required to define all pages in 1 single master selector.
Consequently, the `<store-page>` is now sharable since it does not assume a certain
routing structure of its parent. You can reuse this very same `<store-page>` in
a different application and your routing will remain functional!

## 2. Updating the url with data binding

Up to this point, we have only defined static routes. We knew beforehand what
routes exist in our application. However, most of the time your urls are dynamic:
your blog has several posts with a unique identifier or urls contain the name of
a user which you need to retrieve.

Let's define a category filter for the store page. It allows the user to select
a category from a dropdownmenu and based on the provided value we want to: 1. Show the category results to the user and 2. Update the url to make the search sharable.

Our dropdownmenu:
```html
<paper-dropdown-menu label="Choose a category">
  <paper-listbox class="dropdown-content" selected="{{category.category}}"
                 attr-for-selected="data-category">
    <paper-item data-category="clothing">Clothing</paper-item>
    <paper-item data-category="food">Food</paper-item>
    <paper-item data-category="drinks">Drinks</paper-item>
  </paper-listbox>
</paper-dropdown-menu>
```

As you can see we bind the `selected` property of the inner `paper-listbox` to
`{{category.category}}`, but how is this used? This value is used in a new
`<carbon-route>`!

```html
<carbon-route route="{{subRoute}}" pattern="/:category"
              data="{{category}}"></carbon-route>
```
The `{{subRoute}}` is bound to the `tail` of the previously defined `<carbon-route>`
of the `store-page`. When we click an item in the dropdownmenu, `{{category.category}}`
is updated to the corresponding `data-category` value. However, the beauty of
the data-binding system is that it is also reflected back the to `<carbon-route>`!
That route uses the same (two-way) binding and updates the url to for example
`localhost:8080/store/Amsterdam/food`.

**(Note: at the moment of writing there is a pull-request open to fix some problematic
  behavior with updating with data-bindings. This will be fixed soon.)**

## 3. Relative links

The last topic of this blog concerns the linking to pages. Since an element does
not know what the current url is, it is not possible to statically define the
`href` of the corresponding link. This problem took me quite some time to figure
out, but in the end I discovered another neat feature of the `<carbon-route>`.

All `route` properties expose a field `prefix`. This `prefix` contains the full url
as matched by parent `<carbon-route>` or the base `<carbon-location>`. In other words:
If I define a `<carbon-route>` which matches on `/:store` and I load
`localhost:8080/store/Amsterdam/food`, the prefix will be `localhost:8080/store`
and the tail will be `/food`. If you want to link to the Amsterdam store page inside
`<store-page>` you can define the link as such:

```html
<a href$="[[route.prefix]]/Amsterdam">Amsterdam</a>"
```

Moreover, if you have a list of stores
(`[{location: 'Amsterdam'}, {location: 'new-york'}]`)
and you use a `<dom-repeat>` to enumerate through the list, you can define the link as

```html
<a href$="[[route.prefix]]/[[item.location]]">Amsterdam</a>"
```

This neat little trick allows you to modify urls in parent elements without breaking
the internal routing structure of child elements such as `<route-page>`.

## 4. Conclusion
This blog post roughly describes my experience with the new `<carbon-route>`.
There are a lot of other possible usages, which I might address in a later post.
If you have any question, feel free to contact me on the Polymer Slack (`@timvdlippe`) or on Twitter (https://twitter.com/TimvdLippe).
