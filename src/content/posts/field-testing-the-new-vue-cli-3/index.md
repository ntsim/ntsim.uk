---
id: '1a582a4c-b550-4575-8f3e-c9d67aae87b1'
title: 'Field testing the new Vue CLI 3'
description: 'Vue CLI 3 is a major re-work that promises zero config and an extensible plugin ecosystem. Does it live up to the hype?'
date: '2018-08-19 23:05:12'
tags:
  - Vue
  - Webpack
  - JavaScript
---

> **16th January 2021**: Vue CLI has reached v4, and some of this post may no longer apply. I haven't
> worked on a Vue project in ~2 years now, so I may re-visit this the next time I have the opportunity
> to do so.

Recently I had the opportunity to start a new project with Vue, presenting a great opportunity to
get back up-to-date with the Vue ecosystem (my last Vue project was last year).

Luckily (or unluckily) for me, the major 3.0 update for the official Vue CLI was just around the
corner so I decided to kickstart the project with the latest RC release to create a clear direction
for the project's future infrastructure and tooling. Having spent a few weeks getting to grips with
it, the official 3.0 release has just been released and I feel like it's a good time to share my
experiences/thoughts with it so far.

## What's new?

Previously, the Vue CLI 2 architecture involved the use of 'templates'. These templates were
essentially boilerplate projects that would be copied into the user's project and then placeholders
in the project files would be replaced with any required input. Nothing too unconventional here.
Many projects that auto-generate boilerplate scaffolding use this approach e.g. [JHipster](https://www.jhipster.tech/).

Vue CLI 3 turns things on its head and ditches the templates in favour of plugins and a convention
over configuration approach. The main motivations being:

- Users will have to modify the project templates to suit their use cases. This leads them to
  immediately diverge from the template (in React land, this would be when a user 'ejects' from
  `create-react-app`. Consequently, it is usually not easy for users to upgrade to future versions
  of the template once they have started customizing it.
- Templates are not easily extensible and new functionality cannot be applied in an easily automated,
  ad-hoc way when requirements change in the future.
- Templates still expose a lot of configuration complexity and this may be particularly intimidating
  for new users.

By using plugins, the new CLI hopes to keep users running along the beaten path and keep an upgrade
path open for the future. This theoretically means that more users can 'come along for the ride' and
spend less time maintaining project configurations.

Starting out with Vue CLI should be as simple as:

```bash
vue create my-project
```

If users want to add some new functionality, it should be as simple as applying a new plugin (which
bootstraps and generates its own dependencies as required).

So far so good. Let's dig a little deeper.

## Initial thoughts

Things looked promising initially. The CLI itself was fairly pleasant to use, with a nice number of
out-of-the-box options to choose from and a straightforward set of steps to run through.

Initially I tried out two setups. The first was a fairly 'default' setup using Babel, Vue Router,
Vuex, Sass, ESLint (with Airbnb) and Jest for testing. The second was more unconventional by
swapping out Babel and ESLint with Typescript and TSLint.

The Typescript setup was a little more interesting as there are still a few limitations when using
Typescript in Vue. From a tooling perspective, the main issue is the lack of TSLint support for Vue
files in Jetbrains IDEs which means that you won't get on-the-fly feedback from the linter. A bit of
a dealbreaker for me unfortunately.

On the other hand, I couldn't spot any major issues with using the Babel setup - everything works as
expected (including the linting). This is still probably the 'best' way of configuring a Vue project
currently. It doesn't seem like Typescript is quite ready for prime time in a Vue environment
(hopefully this will change soon).

Interestingly, you can actually configure a hybrid setup that transpiles Typescript using the Babel
v7 pre-releases instead of the Typescript compiler. Personally, I'm not very keen on using this
until a stable Babel v7 release though.

Digging a little deeper into the generated project, the CLI generates a `package.json` with some
default scripts:

```
"serve": "vue-cli-service serve",
"build": "vue-cli-service build",
"lint": "vue-cli-service lint",
"test:unit": "vue-cli-service test:unit"
```

From this, it appears that `vue-cli-service` is trying to abstract over all facets of development
tooling. This is presumably to make it is easier to adopt the new plugin architecture, allowing any
tools to be preconfigured in the Vue-specific way. On the other hand, it seems like you will have to
be willing to buy into `vue-cli-service` to get all of its benefits.

## Teething problems

Whilst initial project setup was pretty painless, a few issues quickly arose from use. Whilst mostly
not dealbreaking, these were worth documenting further.

### Running Jest in Jetbrains IDEs

Running Jest tests from the IntelliJ GUI would fail, whilst running them from the CLI would work
fine. The problem here it that the Vue CLI generates the project with a Babel config using a preset
called [@vue/babel-preset-app](https://www.npmjs.com/package/@vue/babel-preset-app). It turns out
that this preset is specific to Vue CLI and is intended to be ran via the `vue-cli-service`. From
the readme, you can find the following:

> **Note: this preset is meant to be used exclusively in projects created via Vue CLI and does not
> consider external use cases.**

Jest tries to pre-process any files through Babel before running tests and this can be problematic
if it is not configured correctly. As the preset only works properly when running via
`vue-cli-service`, it will not bootstrap Babel correctly. This leads to the frustrating 'unexpected
token' errors that people familiar with both tools may have come across.

This will probably be a frequent problem for new users as there will be plenty of people that also
run Jest through a Jetbrains IDE. I can understand that the Vue team has to cater for everyone, but
I'm still a bit disappointed that they haven't factored in Jetbrains users.

The current design of the preset is not very re-usable for Vue users who don't use Vue CLI.
Unfortunately, it [doesn't look](https://github.com/vuejs/vue-cli/issues/2031) like there is an
appetite from the Vue team to decouple the dependency on Vue CLI. Consequently, I would say there's
currently a gap in the market for a more explicitly configurable, dependency-free Vue preset.

To workaround the problem, we can manually set the environment variable flags that the Vue preset
relies on. We can explicitly define these in `babel.config.js`:

```javascript
if (process.env.NODE_ENV === 'test') {
  process.env.VUE_CLI_BABEL_TRANSPILE_MODULES = true;
  process.env.VUE_CLI_BABEL_TARGET_NODE = true;
}

module.exports = {
  presets: ['@vue/app'],
};
```

This feels a bit icky, but it's currently the easiest way of making sure Jest always works with the
Vue CLI generated project. Personally, I think it's questionable that they even rely on environment
variables in the first place for a Babel preset. Most presets typically have the sense to let you
configure them through their public API.

### Dependency hell

Closely related to the previous Jest issue, there was another Babel/Jest issue that involved the
NPM dependencies resolving incorrectly leading to tests failing. Again, I was faced with the
'unexpected token' errors mentioned earlier.

This only really manifested on occasions where the NPM dependencies changed e.g. when updating the
project to the latest version of Vue CLI . Part of the issue seemed to stem from having Jest
explicitly defined in the `package.json` dependencies (normally it is an implicit dependency of
the `@vue/cli-plugin-unit-jest` package). The reason for this is to allow IntelliJ to run Jest tests
from the GUI (you need it explicitly defined in `package.json`).

After some further investigation, it appeared that this was partly due to NPM messing up its
dependency resolution and importing Babel v6 instead of v7 when pre-processing test files. The
solution to this ended up being to do the following:

1. Run `npm cache verify`.
2. Remove the `package-lock.json` and `node_modules` directory.
3. Re-run `npm install`.

It was interesting to observe NPM getting the dependency resolution really wrong, but it was quite
frustrating that these issues mostly stemmed from relying on Babel v7. Again, v7 is still in
pre-release, so it's somewhat reckless to see Vue CLI using it as a key dependency. I can
appreciate that we all want v7 as soon as possible, but we should still be a little more cautious
about living on the bleeding edge.

### Opinionated environment variables

Vue CLI takes an [opinionated approach](https://cli.vuejs.org/guide/mode-and-env.html) to
environment variables by automatically loading `.env` files that are defined in the project root.
Different `.env` files can be chosen by providing a different 'mode' parameter when running any of
the `vue-cli-service` commands.

This can be pretty useful when starting out a project, but it doesn't currently consider that you
may want to define `.env` files in a different fashion. For example, I prefer to use the
[dotenv-webpack](https://github.com/mrsteele/dotenv-webpack) plugin as it allows you to validate
your `.env` file against a reference `.env.example` file. This helps you make sure that your `.env`
files have all the required variables (pretty useful).

As Vue CLI also loads in `.env` files, you have to put in a bit of thought into how these two
loading methods will interact with one another. Vue CLI's handling is a little more complex than
`dotenv-webpack`, as you have to consider how the 'mode' parameter is set (it will automatically
default to different modes depending on the task being ran). On the other hand, `dotenv-webpack`
does not make any such assumptions and allows you to configure it how you think it should be
configured for your project.

## Just another Webpack wrapper?

After spending some time with the Vue CLI and working through the initial teething problems, I
couldn't help but ponder if the abstraction was really worth it. Is there _really_ a need to wrap
Webpack? What is the _problem_ with just using Webpack?

The Javascript community already has a reputation for re-inventing the wheel and it certainly feels
like Vue CLI is another great example of this mentality. An additional API on top of Webpack seems
counterproductive to encouraging the community to consolidate on a standard set of tools. If it
uses Webpack under-the-hood, why not just use Webpack?

In my opinion, Webpack itself is not terrible to use. Every version has progressively improved on
the developer experience and it feels fairly intuitive after getting familiar with the workflow.

Is documentation the problem? Having been around for the horrible documentation of the Webpack v1
days this seems unlikely. In general, Webpack documentation is now fairly high quality and is
certainly a much better source of truth than it has been.

Is configuration complexity the problem? Vue CLI attempts to reduce configuration complexity,
however the API surface area is still fairly broad and has plenty of possible options meaning that
any large Vue project can still grow to become fairly complex. The difference here will be that that
you have to potentially learn two APIs instead of one to achieve the same thing. I can also foresee
the Vue CLI API inhibiting the ability to do things you would be able to do with vanilla Webpack.

At this point, I'm questioning if Vue CLI's opinionated convention over configuration approach is
really worth the trouble long term. From my experience, convention over configuration is great -
until it's not. You then end up spending unnecessary amounts of time investigating and implementing
hacks to workaround specific use cases. Although there is a higher upfront cost to explicit
configuration, having the flexibility to grow your solution to perfectly fit your needs can be very
powerful.

## A potentially hidden gem

To try and offer and escape hatch and provide more control over the Webpack configuration, Vue CLI
employs an interesting project called [webpack-chain](https://github.com/mozilla-neutrino/webpack-chain).
This allows you to mutate the Webpack configuration with a fairly pleasant chaining API. The example
from the docs looks like the following:

```javascript
module.exports = {
  chainWebpack: (config) => {
    config.module
      .rule('vue')
      .use('vue-loader')
      .loader('vue-loader')
      .tap((options) => {
        // modify options...
        return options;
      });
  },
};
```

At first I was a little skeptical of the utility of this, but having looked at the documentation a
little more, I'm quite impressed by the API. In fact, it's powerful enough that under-the-hood, Vue
CLI actually uses it for all of its core Webpack configuration.

To me, it seems like `webpack-chain` is more of a step in the right direction towards solving the
'Webpack problem'. The API encourages composition and re-use, whereas monolithic Webpack
configurations generally don't. I have in the past relied on `webpack-merge` to enable code sharing,
but this has not been perfect as it relies on having an understanding of how configurations will be
merged together (this is not always a simple task with large configurations).

It feels to me that `webpack-chain` could potentially avoid merging messiness, however, I can also
see that it could probably be used to paint yourself into a corner as well. Regardless, it seems to
me like there is a place at the table for `webpack-chain` and it's worth keeping an eye on it.

Perhaps it would be good to see a standard set of packages that use `webpack-chain` to provide the
basic components of most Webpack configurations in a generic way (not opinionated like Vue CLI). The
user could then compose them together in their Webpack configurations and add any finishing touches
as required.

I intend to check out `webpack-chain` more in the future

## Conclusion

Vue CLI 3 introduces some interesting new tooling ideas and features - most importantly, a
plugin-based architecture. This aims to provide users with a flexible set of packages that can be
updated and maintained more easily. Whilst this is a nice goal, it still remains to be seen if this
will pan out the way that the Vue team are envisaging.

Unfortunately, there are a bunch of teething problems that certain users will feel more than others
depending on their workflow and choice of tools. I feel Jetbrains IDE users get the short straw here
and I hope that the Vue team consider our use-cases more in the future.

There is also the elephant in the room. Is the solution to our Webpack tooling woes an opinionated
wrapper around Webpack? I personally don't feel that this is a perfect solution and will suffer from
edge cases where user requirements are quite specific.

Finally, to round up, should you use Vue CLI 3?

On the whole, I think the pros do outweigh the cons and the answer will typically be yes. I think
the Vue team have managed to achieve most of their key objectives. It provides a consistent,
beginner friendly starting point for Vue projects that can get you up and running fast. However, I
am somewhat sceptical of how it will scale for larger projects and if it will live up to the claim
of 'not needing to eject'. We will see!

For people that are already comfortable with Webpack and using it successfully in their projects, I
don't think there's a major reason to move over to Vue CLI. It will probably be easier to stick with
your current tooling setup, however, Vue CLI will no doubt be useful for new projects where you
don't need complete control over everything.
