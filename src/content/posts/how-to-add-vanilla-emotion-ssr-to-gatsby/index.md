---
id: '43631a0f-58f6-41d2-b57d-6dea75c06821'
title: 'How to add vanilla Emotion server-side rendering to Gatsby'
description: "Like Emotion but don't want to use the React API? Want to use server-side rendering with Gatsby? Here are some simple steps to get started."
date: '2021-01-08 23:58:00'
tags:
  - React
  - JavaScript
  - Gatsby
  - Emotion
---

On the whole, having used [Emotion](https://emotion.sh/) on a few of my last projects, I'm quite happy with it compared to other CSS-in-JS libraries. I like that they offer multiple APIs, allowing you flexibility in how you want to style your components. Personally, I prefer using the plain `css` API as the others seem to have major drawbacks (one for another post).

Emotion pushes you to style elements your elements with their `css` prop, instead of `className`. There are some advantages from doing this - you get out of the box server-side rendering (SSR) and theming. Unfortunately, there's also some big disadvantages, in particular the super awkward `ClassNames` component. Colin McDonnell explores a few of the issues in much greater detail in his post [Why you shouldn't use @emotion/core](https://colinhacks.com/essays/emotion-core-vs-vanilla-emotion).

Thankfully, you can avoid the `css` prop entirely by using the vanilla `@emotion/css` package. The main caveat here is that you have to manually setup server-side rendering. This can be a little tricky with Gatsby as it's not obvious how to do this from the [documentation](https://emotion.sh/docs/ssr#gatsby). If you look closely, none of it actually describes how to integrate with vanilla `@emotion/css`. What is the correct way?

## Inject server-side rendered styles

We need a way to inject Emotion's generated styles into the server-side rendered markup. Thankfully, Gatsby exposes a [few APIs](https://www.gatsbyjs.com/docs/reference/config-files/gatsby-ssr/) that we can use.

I've found that the `replaceRenderer` API was the most concise way of doing this. Be aware that it may have its own caveats depending on if you're using plugins that rely on it too.

In `gatsby-ssr.js`, we can add something like the following:

```jsx
import { cache } from '@emotion/css';
import createEmotionServer from '@emotion/server/create-instance';
import { renderToString } from 'react-dom/server';

export const replaceRenderer = ({ bodyComponent, setHeadComponents }) => {
  const { extractCritical } = createEmotionServer(cache);
  const { css, ids } = extractCritical(renderToString(bodyComponent));

  setHeadComponents([
    <style
      key="app-styles"
      data-emotion={`css ${ids.join(' ')}`}
      dangerouslySetInnerHTML={{ __html: css }}
    />,
  ]);
};
```

This essentially renders our Gatsby app into a string on the server, before Emotion's `extractCritical` is used to pull out all of the various classes (in the `ids` variable) and styles (in the `css` variable). Using Gatsby's `setHeadComponents`, we can inject all of this into our HTML's `head` as a `style` element.

Note that in the `style` element's `data-emotion` attribute, I've used a key of `css` as this is the default used by `@emotion/css`. If you want to change it, you should check out the documentation on [Custom Instances](https://emotion.sh/docs/@emotion/css#custom-instances). It's pretty unlikely you'll need this besides for a specific set of use-cases though.

## Finishing up

Now that our styles have been injected, we just need to check to check that everything is working correctly. Build and serve your Gatsby app:

```bash
gatsby build
gatsby serve
```

Finally, disable JavaScript in your browser and visit your app URL. If everything is correct, you should be able to see that everything is styled correctly. This means that the server-side rendered styles have been injected properly. Without them the app would be unstyled as Emotion cannot run client-side with JavaScript disabled.

That's it! Enjoy using vanilla Emotion with Gatsby.
