---
id: '15d89498-e51f-4a2f-95c5-0a97e87e1d8f'
title: 'Blogging with Gatsby'
description: 'Learn how I finally deployed my blog using Gatsby. Expect to see lots of interesting content on programming and tech!'
date: '2017-12-28 20:52:12'
tags:
  - React
  - JavaScript
  - General
---

First posts are always a little awkward to write. Normally I don't have much to say other than the usual
'Hey guys, this is my blog, I hope you like it' type opening. Fortunately, I've got something more interesting to say
this time as I decided to opt for some new tech to try out to get this blog out of the door.

## The original 'plan'

A little history is probably due:

- **July 2016** - I tell myself that I'm going to start blogging properly. I decide to build the site using
  [Grav](https://getgrav.org/), a CMS built in PHP. I quite like it.
- **August 2016** - Blog site is ready to ship.
- ???
- **November 2017** - I tell myself that I'm _actually_ going to start blogging properly. I decide to _rebuild_ the
  site using [Gatsby](https://www.gatsbyjs.org/) - a Javascript static site generator.
- **December 2017** - Blog site is finally deployed.

This was over a year of dragging my feet. Looking at the repo history, I was even prepping post-launch content in
September, which seemed a bit weird given that there was no launch.

What happened?

My 'best' excuse is that I ended up freelancing a lot between December and October, which ate up 11 months of
my free time. Perhaps if something had even been released, it probably would have been unlikely that I would
have got much content published. It feels like I could have released **something** though...

Other commitments aside, I feel like the other _major_ challenge at the time was deployment. As there were a
few hard questions to answer, I found it quite easy to just ignore them. For example, where would it be hosted?
What domain name would I use? How would the deployment pipeline look?

## Starting from scratch

A few months back I decided to drop the excessive freelancing and have been chilling out a lot since. I had been
playing excessive video games instead, so I finally had no excuses to hide behind. Gathering my motivation, I
decided to **finally** get this blog into a deployed state.

Unfortunately, in that time, I had completely lost track of where I was with the original version. I didn't feel
particularly connected to it anymore and decided it would be better to just start again. This time with some even
cooler tech and a focus on getting it shipped.

Originally I wanted to believe that I would use Grav's CMS functionality to blog outside of a text editor. On
some reflection, I realized this would not be the case, so I decided to go back to basics and use a static
site generator (SSG) approach so that I could actually get something released without having to worry about deployment
and infrastructure.

A quick scan of [StaticGen](https://www.staticgen.com/) suggested I should use Jekyll. Unfortunately, I don't
particularly like Ruby, and I'm not keen on how Jekyll works (templating and such).

Looking through the alternatives, there were a number of Javascript options which sat well with my skillset.
One that particularly caught my eye was [Gatsby](https://www.gatsbyjs.org/).

## The Great Gatsby?

For the uninformed, Gatsby generates static sites using a combination of React components and GraphQL.
GraphQL is able to query data sources such as Markdown files, allowing us to write nicely formatted content which
is then processed and rendered by the React components. During the build step, the rendered output is saved
as static HTML/CSS/Javascript files giving us our shiny new site.

It comes with its own ecosystem of plugins which make our lives quite easy (for the most part). Creating this blog
was mostly installing a bunch of packages that you'll find in the blog
[starter project](https://github.com/gatsbyjs/gatsby-starter-blog). These included plugins for:

- Markdown files (uses Remark under-the-hood).
- SCSS (works well with component CSS Modules as well).
- Typography management.

In terms of what I thought about it, for a simple breakdown:

### Good parts

Working with React components is usually quite enjoyable as JSX is essentially Javascript sugar. No need to
learn syntax for templates you'll never use again. I've personally found it has been much harder to forget how to
write React than it has been with a template based framework like Angular.

As we're working in React land, components are first-class citizens meaning it's more straightforward to
re-use fragments than in most traditional templating systems. We can also leverage the full power of Javascript
(if required) instead of having to awkwardly bolt it on top of the templating system.

Hot reloading out-of-the-box is awesome as always. I don't think I could work on any frontend project without it
anymore as the feedback cycle is very fast. Plus it's always satisfying to see your work come to life!

These things combined make for a pretty nice development experience - Gatsby certainly _feels_ modern.

### Meh parts

The plugin system is very useful for getting a Gatsby project off the ground, but currently the documentation
could be improved. I've found myself having to look at source code on a few occasions to figure out how to configure a
plugin correctly.

Hot reloading doesn't always work properly and can crash out. I've found this is usually when I'm modifying a
React component and I save the component in a state that has some problematic syntax error. It would be good to get
this ironed out properly so that the dev server doesn't need to be restarted in these situations.

The inclusion of GraphQL is interesting, and maybe a bit forced (perhaps for the 'cool' factor). I think I would have
preferred a data sourcing mechanism that was a bit closer to metal; adding GraphQL as a layer of abstraction feels
somewhat overkill. Regardless, it was good to get a feel for what GraphQL is about and what we can use it for. The
in-browser [GraphiQL](https://github.com/graphql/graphiql) editor was also nice to use when you want to play around
with your queries and debug them in realtime too.

Whilst not the end of the world, the Gatsby project seems to be primarily driven forward by the lead maintainer
(Kyle Matthews), which could draw some question marks over how things will pan out long-term.

### Bad parts

Build/hot reloading errors are sometimes not communicated properly. You get blank error messages or unhelpful ones.
It would be good to see this improved to avoid developers grasping around in the dark for clues.

### Overall thoughts

Gatsby introduces a fresh perspective to the crowded SSG space and importantly leverages React components instead of
traditional templating systems (which is awesome). I think this opens up the possibility for developers to do more
with static sites than ever before! If you're familiar with React already, it's pretty much a no-brainer.

Whilst I do think there is room for improvement, Gatsby was able to hold up very well to my requirements and I
would like to see more of it in the future when working on projects of this kind.

## Pushing to production

Getting the site ready to go is one thing, but deploying is certainly another (as I've discovered before). To simplify
things, I opted to use [GitLab Pages](https://about.gitlab.com/features/pages/). If you've never heard of GitLab before,
they're an awesome alternative to GitHub and Bitbucket for hosting your Git repositories. They offer an awesome
bunch of features and are free to use (with enterprise options available).

I originally intended to utilise GitHub Pages instead, but there were a few things that didn't quite work out,
specifically:

1. No built-in CI. Would most likely have to remember to build and push the project locally before each release.
2. Deployment branch must be `master` for User/Organization pages (which I would be using).
3. Project would have have to be publicly visible (bit of an issue if I want to stage content or leave some notes).

I've had a fair amount of exposure to GitLab CI, and I think it's definitely a killer feature. It ended up being as
simple as copying a `.gitlab-ci.yml` [example](https://gitlab.com/pages) and boom, the site was building and
deploying automatically.

Life is good.

## Future directions

Now that the blog is out in the wild, I want to try and put a bit more effort into writing and getting some content
out there. I will be optimistically aiming to get new content out monthly. We'll see...

I don't think I'm an amazing writer by any stretch, but it's something I'm interested in trying to improve
on so this is a good opportunity to do so. Hopefully I'll be writing about something you too find interesting in the
near future.

Until next time!
