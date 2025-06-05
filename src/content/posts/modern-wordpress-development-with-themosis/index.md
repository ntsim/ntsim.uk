---
id: '998363fd-0aa3-45cc-99d3-20054ac1f9eb'
title: 'Modern WordPress development with Themosis'
description: 'An introduction to the Themosis framework for WordPress. Learn how you can use modern Laravel features in your WordPress projects!'
date: '2019-04-28 15:10:00'
tags:
  - Themosis
  - Laravel
  - WordPress
  - PHP
---

If you've been working in web long enough, you will most likely have run into WordPress in some
shape or form. For a seasoned developer, this experience has most likely involved throwing your
hands up in the air in frustration from its archaic architecture, badly written plugins or some
theme's spaghetti code.

I've recently had the pleasure of working on a redesign of a WordPress site for one of Hive's
clients. For a typical application following good, modern practices, this wouldn't be so difficult,
but upon reviewing the codebase it became clear that this would be anything but.

The code had already passed through several agencies in the past and was consequently riddled with
the type of problems you would expect from this kind of churn; badly written and buggy features,
inconsistent coding practices and over-reliance on third-party and internal plugins (that we had no
control over).

Fortunately, with some buy-in from the client and the rest of the team, we had the go-ahead to
rebuild the site (still in WordPress). Thankfully this posed a great opportunity to try a WordPress
framework that had recently piqued my interest.

## A better WordPress experience

[Themosis](https://framework.themosis.com) is a framework that attempts to modernise the WordPress
development experience by wrapping the WordPress core with with an MVC layer that enables us to
write clean object-oriented code instead of the typical procedural mess. It's actually been around
since 2014, and its initial implementation revolved around defining all application code in the
theme itself. This code would then be glued together by a loose set of Laravel packages.

Recently, Themosis reached its 2.0 release and has seen some major re-work. It has now moved away
from defining everything inside the theme and instead fully embraces most of the Laravel feature
set, including its project structure and general application architecture. Laravel developers will
really feel at home here, with some exceptions (more on this later).

Out of the box, we get many of Laravel's standard features, like routing, Blade templates, Eloquent
and validation. These tools allow us to take control of application-level code that WordPress would
normally handle and inject some sanity back into the workflow.

### How does it work?

Themosis is activated via a theme that has been built to work with WordPress (there is a
[default starter theme](https://framework.themosis.com/docs/2.0/installation/#install-the-theme)
that you are encouraged to use). The theme acts as an entrypoint that delegates any further request
handling to the bootstrapped Themosis application instance. From that point, you're basically
working with a Laravel application rather than WordPress!

## Themosis in action

The project we used Themosis on was a fairly simple WordPress site with standard requirements. This
included the usual assortment of custom post types that required custom meta fields, contact forms
for getting in touch and a shop for selling products.

In the next few sections we'll go through the general experience of working with Themosis on this
project.

### Dependency management

One of the areas that WordPress arguably has a deficiency is dependency management. Administrators
can easily break things as they normally have free reign to update and install plugins via the admin
UI. To counteract this, we can use Composer to install WordPress plugins instead (via [WPackagist](https://wpackagist.org/))
to control package versions. Themosis ships with this setup already and even uses Composer to
install WordPress itself.

This is a good start and WordPress projects in general should adopt this approach to take advantage
of packages in the Composer ecosystem and version control their plugin dependencies!

Additionally, as a benefit of having most Laravel features available to us, we have also been able
to keep the number of WordPress plugins down to the bare minimum. Implementing custom functionality
in Laravel is generally a breeze compared to WordPress and really reduces the need for most plugins.

I'm also not a big fan of WordPress plugins as they can have a large variance in quality and can
introduce security vulnerabilities where higher quality Composer packages generally don't.

### Interop with Laravel ecosystem

Because we're using Composer and Themosis is basically a Laravel application, we can bring in most
packages that specifically target Laravel. For example, I was able to get the very useful [Laravel Debugbar](https://github.com/barryvdh/laravel-debugbar)
and [Laravel IDE Helper](https://github.com/barryvdh/laravel-ide-helper) packages working with
fairly minimal fuss. These packages are pretty much development essentials if you're working with
Laravel and use PHPStorm, by the way!

Debugbar was a little trickier to get working, but just required the `pushMiddleware` method to be
added to the `Kernel` class:

```php
class Kernel extends HttpKernel
{
  public function pushMiddleware($middleware)
  {
    if (array_search($middleware, $this->middleware) === false) {
      $this->middleware[] = $middleware;
    }

    return $this;
  }
}
```

PHPStorm's Laravel plugin also works and successfully identifies the project as a Laravel one,
offering all the usual auto-completion and quick navigations in routes, config, etc. The development
experience has consequently been pretty seamless and is pretty much no different than working on a
Laravel project!

###Project structure

Whilst the project structure very closely resembles Laravel's, it is not a complete one-for-one
mapping. WordPress and its associated files live in a separate directory called `htdocs`. This
convention should probably be a bit familiar for people that have deployed WordPress on Apache
servers. The project structure looks like the following:

```
/project-root
	/app
	/config
	/htdocs
		/cms <-- WordPress core lives here
		/content
			/plugins
			/themes
				/your-theme <-- Your Themosis theme lives here
					/views
			/uploads
		index.php
		wp-config.php
	/resources
		/views
	/routes
```

One thing that is a little confusing with this project structure is where your views should go. The
project root has a `resources/views` directory (like Laravel) and the theme directory has a `views`
directory. Having asked Julien Lambé (the Themosis lead) what he thought, he encourages using the
theme's `views`, whilst the project root's `resources/views` directory should be used for
common/admin pages instead.

Unfortunately, I think for a relatively simple site without any common/admin pages, you don't get
much benefit from this organisation of views. It feels like it would be simpler to place everything
in the project root's `resources/views` directory to be more consistent with Laravel. It would also
save having to jump around the project to different nested directories.

### Frontend tooling

The default theme comes with Laravel Mix installed and lets you immediately get started with a
modern asset pipeline e.g. using Sass and ES6+ JavaScript. Assets are built and will be placed in
the theme's `dist` directory.

It should be noted that when you move into a production environment, direct access to the entire
theme directory should be blocked, with the exception of the `dist` directory.

For convenience, I ended up adding [Lerna](https://github.com/lerna/lerna) so we could run scripts
from the top-level (rather than have to `cd` all the way to the theme directory). This might be a
little overkill, but makes a bit more sense if you need to create some additional admin assets, as
they would most likely go in the project root's `resources` directory instead.

###Database modelling

Whilst Themosis ships with Laravel's Eloquent ORM, it doesn't provide any WordPress models out of
the box. Fortunately, Eloquent is flexible enough to map to WordPress quite easily. Query scopes
are particularly useful here as they simplify the queries you'll need in various places, such as in
relationships like posts and post meta.

Whilst writing these models wasn't hard, there was a bit of work to make some of these function like
their WordPress equivalents. It's a bit of a shame that Themosis doesn't ship with any models and is
perhaps something they should have in the future. It seems like Julien Lambé [prefers](https://github.com/themosis/framework/issues/533#issuecomment-425165666)
using WordPress' default `WP_Post` abstraction instead of Eloquent for simple use-cases, so he may
opt to keep Themosis neutral on this matter.

Something that could potentially fill that gap right now is [Corcel](https://github.com/corcel/corcel),
which _should_ work with Themosis (I cannot confirm this however). Unfortunately, we didn't end up
using it on this particular project (as I forgot about it), but I would definitely give it a look
for my next Themosis project.

### Post types, meta boxes and taxonomies

Themosis provides wrapper API that assist in creating custom post types, meta boxes and taxonomies.
These are mostly just builders that are pre-configured with some sane defaults and allow us to call
the underlying WordPress functions in a more OOP style.

Unfortunately, in the case of the `PostType` API, this still mostly works by providing a
configuration array to the builder i.e.

```php
PostType::make('team', 'Team members', 'Team member')
  ->setArguments([
    'menu_position' => 21,
    'supports' => [
      'editor',
      'excerpt',
      'thumbnail',
      'title',
    ],
    'has_archive' => false,
    'rewrite' => [
      'with_front' => false,
    ],
  ])
  ->set();
```

I feel like Themosis could go a little further here and provide additional methods that encapsulate
the configuration more so we can write it out more easily (with fewer potential mistakes) e.g.

```php
PostType::make('team', 'Team members', 'Team member')
  ->setMenuPosition(21)
  ->setSupports('editor', 'excerpt', 'thumbnail', 'title')
  ->setHasArchive(false)
  ...
```

One area that does seem to work quite nicely is the `Metabox` and `Field` APIs. Custom fields for
saving post meta can be created easily and are added to the WordPress admin as React components.
Out of the box, a fairly large number of field types are available and should cover most use-cases.

If you're wondering where to put all these declarations, I personally opted to put it in a
`Hookable` class called `PostTypes`. [`Hookable`](https://framework.themosis.com/docs/2.0/hooks/)
classes are automatically loaded and are a great place to declare any code where WordPress hooks
are concerned (as the name suggests).

### Routing

For the most part, Themosis' routing works exactly like Laravel's and is declared in a
`routes/web.php` file using the `Route` API. There are some notable differences that allow Themosis'
routing to integrate with WordPress' routing model.

Themosis has a concept of [WordPress routes](https://framework.themosis.com/docs/2.0/routing/#wordpress-routes)
and these include things such as home, page and post routes. These are also declared using the
`Route` API:

```php
Route::get('home', function () {});
Route::get('page', function () {});
Route::get('single', function () {});
```

You inevitably end up needing at least some of these if you want to load pages with dynamic content
from WordPress. For example, we used this for declaring a default page route using
`Route::get('page')`, which would render a matching WordPress page and its content in a default
template whenever a more specific route could not be found earlier in the stack.

I'm personally not a massive fan of declaring WordPress routes like this, and would have preferred
if they were declared via another API e.g. `WPRoute`. Piggybacking on-top of `Route` feels a bit
hacky and needs some trial and error before you figure out how these route definitions interact
within WordPress and Laravel.

How you define these routes also depends on how you decide to utilise the controller layer. If you
want to use Eloquent instead of WordPress loops, it will probably be better to use standard Laravel
routes. If you want to use loops, opt for Wordpress routes instead.

### View templates and directives

Themosis gives us the option to define views as either Blade or Twig templates. For this project, I
just opted to use Blade as it had the least effort involved to get started. These templates work
exactly like Laravel's, but with some additional directives to help work with WordPress.

One such example is the `@loop` directive. This is essentially an alias for the standard WordPress
loop and looks like this:

```php
@loop
  <h1>{{ Loop::title() }}</h1>
  <div>
		{{ Loop::content() }}
	</div>
@endloop
```

You typically need this whenever you're working with dynamic content that is being pulled in for a
WordPress route.

There is also the `@query` directive that can be used for creating custom loops (we avoided using
these):

```php
@query(['post_type' => 'post', 'posts_per_page' => 3])
  <h1>{{ Loop::title() }}</h1>
  <div>
		{{ Loop::content() }}
	</div>
@endquery
```

Whilst this could be useful in simpler use-cases, I opted to leverage Eloquent and place any query
logic in the controller layer (following a standard MVC pattern). This also allowed us to add
doc-block type hints for our view models using `@php` blocks:

```php
@php
  /**
   * @var \App\Post[]|\Illuminate\Database\Eloquent\Collection $posts
   */
@endphp
```

This is great for providing a manifest of what variables are in scope, and with an IDE like
PHPStorm, this also gives you auto-completion within the template. I would also recommend doing this
for standard Laravel projects as well!

### Form handling

There are two options for form handling in Themosis. You can either do it the standard Laravel way
or using Themosis' new [Form](https://framework.themosis.com/docs/2.0/form/) API. The new Form API
looks interesting and feels quite similar to Symfony Forms. For most use-cases it probably works
well enough for scaffolding up forms quickly.

For this project, I opted to build our forms using standard Laravel APIs such as `Request` and
`Validator`. I'm personally not a big fan of form builders in controllers as it feels like it starts
blurring the separation of concerns between the controller and view too much. I think a better
approach is to define form components/partials for re-use instead.

Unfortunately, the standard Laravel approach needs some tweaking to our HTTP middleware as Themosis
doesn't include all of the necessary middlewares to make errors and old input persist in the
session. We just needed to add the `StartSession` and `ShareErrorsFromSession` middlewares like so:

```php
use Illuminate\Session\Middleware\StartSession;
use Illuminate\View\Middleware\ShareErrorsFromSession;

class Kernel extends HttpKernel
{
  protected $middlewareGroups = [
    'web' => [
      'wp.bindings',
      'bindings',
      StartSession::class,
      ShareErrorsFromSession::class
    ],
  ];
}
```

Additionally, we also needed to manually add the `old` helper method to display old form inputs in
our form. Unfortunately Themosis doesn't come with this out of the box, but it's not difficult to
add in some file like `bootstrap/helpers.php` (required in through `bootstrap/app.php`):

```php
if (!function_exists('old')) {
  function old(string $key, $default = null)
  {
    return Session::getOldInput($key, $default);
  }
}
```

Other useful helpers such as `dd` are also not included with Themosis unfortunately.

### WooCommerce integration

One of the final project requirements was a simple shop for selling products. As we were using
WordPress already, it seemed pretty obvious to use WooCommerce to fulfil this requirement.

Themosis' documentation has an [entire section](https://framework.themosis.com/docs/2.0/woocommerce/)
on setting this up, as you do need to make some additional tweaks to make everything work. One thing
we noticed the documentation lacked was how to enable featured images for products. You need to
remember to add the following to your theme's `config/support.php`:

```php
return [
  'post-thumbnails' => [
    'product'
  ],
]
```

Whilst the default WooCommerce seems to work fine for most use-cases, it doesn't seem to be very
flexible if you need to override some of the more nested templates. I did some investigation and it
looks like you need to create a `woocommerce` directory in your theme and create vanilla PHP
templates that match the corresponding template.

This is how you would normally override WooCommerce templates in WordPress, but it feels a little
disappointing that Themosis doesn't provide a way to use Blade templates instead. Unfortunately, it
might also be impossible to change this (in a non-hacky way) as long as WooCommerce implements its
own template loader.

### Emails

Both Laravel and WordPress have email functionality that presents some challenges for interop.
WordPress uses PHPMailer internally (assumes you'll use SMTP), and can cause friction if you want to
use a HTTP API service instead like SendGrid.

I didn't spend too long investigating the options here, but SendGrid does offer a [plugin](https://en-gb.wordpress.org/plugins/sendgrid-email-delivery-simplified/)
that should theoretically work, however the reviews/ratings aren't particularly encouraging. If you
use another email provider, they may offer a better integration plugin.

We ended up just opting to configure WordPress to use SendGrid's SMTP service, and Laravel with
SendGrid's HTTP API (using a custom [mail driver](https://github.com/s-ichikawa/laravel-sendgrid-driver/)).
Whilst this isn't entirely consistent, at least our emails are still being offloaded to SendGrid.

This was pretty easy to implement as Themosis provides a hookable `Mail` class that configures
WordPress' PHPMailer with SMTP credentials using Laravel's mail configuration variables.

## Final thoughts

On the whole, the last few months using Themosis has generally made it enjoyable to work with
WordPress again. Being able to structure our application in a standard MVC way feels much cleaner
and is more in-line with standard web development. Consequently, I don't think I would ever want to
go back to vanilla WordPress for a greenfield project!

That said, there were still some rough edges to the entire experience and there are improvements
that can be made to streamline things further. I would particularly like to see all views and assets
moved into the project root's `resources` directory and more of an opinionated approach to modelling
the WordPress database with Eloquent.

Things have been great overall and I wouldn't hesitate to encourage other developers to pick up
Themosis for their next WordPress project. If not, it's certainly a project to keep your eye on!
