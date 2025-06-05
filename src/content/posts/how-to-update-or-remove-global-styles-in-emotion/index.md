---
id: '0f3aff11-5937-467b-9d93-c9c6b1350da0'
title: 'How to update or remove global styles in Emotion'
description: "Learn how to change globally injected styles in Emotion without breaking the rest of your app's styles."
date: '2021-05-03 17:15:00'
updatedDate: '2024-05-03 17:15:00'
tags:
  - React
  - JavaScript
  - Emotion
---

Recently, I was working with themes in Emotion and needed to update the global styles based on the user's current theme. In my case, I wanted to change things like the global typography and background colours.

I initially created something that looked like this:

```tsx
import { css } from '@emotion/css';

const themeStyles = (theme) => css`
  // Some styles...
`;

const GlobalStyles = () => {
  const theme = useTheme();

  useEffect(() => {
    injectGlobal`
			${themeStyles(theme)};
		`;
  }, [theme]);

  return null;
};
```

The intention was that whenever `theme` changed, `injectGlobal` would be called with new global styles. Unfortunately, whilst the global styles were added correctly the first time the theme changed, they would not be removed if we wanted to change the theme back later.

In hindsight, this probably wasn't too surprising given that the `injectGlobal` function name doesn't really suggest that it is destructive in nature. Regardless, it did surprise me that Emotion has no built-in way to remove global styles. Some other people have thought so too as there is a [GitHub issue](https://github.com/emotion-js/emotion/issues/2131) on this very problem.

## The initial workaround

Thankfully, Mateusz Burzyński (lead maintainer for Emotion) had already suggested a [workaround](https://github.com/emotion-js/emotion/issues/2131#issuecomment-732744875) for this problem that involved creating a custom `injectGlobal`:

```js
import { serializeStyles } from '@emotion/serialize';
import { StyleSheet } from '@emotion/sheet';
import { serialize, compile, middleware, rulesheet, stringify } from 'stylis';

function injectGlobal(...args) {
  const { name, styles } = serializeStyles(...args);
  const sheet = new StyleSheet({
    key: `global-${name}`,
    container: document.head,
  });
  const stylis = (styles) =>
    serialize(
      compile(styles),
      middleware([
        stringify,
        rulesheet((rule) => {
          sheet.insert(rule);
        }),
      ]),
    );
  stylis(styles);
  return () => sheet.flush();
}
```

I didn't really understand what was going on here at first, but combining this with the `GlobalStyles` component, I got something like this:

```jsx
const GlobalStyles = () => {
  const theme = useTheme();

  useEffect(() => {
    const removeGlobalStyles = injectGlobal`
			${themeStyles(theme)};
		`;

    return () => removeGlobalStyles();
  }, [theme]);

  return null;
};
```

Whilst this seemed promising, it unfortunately didn't work during server-side rendering (I'm using NextJS) as errors would be thrown due to the use of the `document` in the `StyleSheet`. It also didn't look like there would be an officially supported solution until [at least Emotion v12](https://github.com/emotion-js/emotion/issues/2131#issuecomment-733138383).

This was fairly uncharted waters, so I needed to do some more digging to figure out what to do next.

## A quick look at how Emotion works

Having spent a little time with the Emotion codebase, I've found it wasn't too complicated and thought it would be worth quickly going over some key things it does internally.

Emotion uses a cache object that inserts styles into the DOM via `style` elements. The cache itself doesn't directly manipulate the DOM, but instead, bootstraps and delegates that responsibility to a `StyleSheet` instance (which we saw earlier). Other than this, the cache is responsible for:

- Keeping track of what styles have been inserted to avoid duplicate style insertions
- Re-using styles that have already been inserted whenever we use `cx` or interpolations

When we use the `@emotion/css` or `@emotion/react` packages, we basically interact with a default cache instance that is created automatically. The `css`, `styled` and `injectGlobal` APIs are just slightly different ways to call the cache's `insert` method.

## What about updating or removing styles?

Whilst style insertion is fairly straightforward, mutating styles is a little more complicated. The difficulty lies in the fact that the cache only exposes a `flush` function that removes any tracked styles from the DOM.

I naively tried to use this function in my `GlobalStyles` component:

```javascript
import { css, flush } from '@emotion/css';

const GlobalStyles = () => {
  const theme = useTheme();

  useEffect(() => {
    injectGlobal`
			${themeStyles(theme)};
		`;

    return () => flush();
  }, [theme]);

  return null;
};
```

Unfortunately, when the theme changed, most of our app's styles would get removed from the DOM and leave everything unstyled. The global theme styles were applied, but this was mostly the opposite of what I was looking for!

What's missing?

## The "aha!" moment

After some playing around, it became obvious that we needed a way to remove only specific global styles (rather than everything). The missing ingredient was a separate cache for our theme's global styles!

Thankfully, this is fairly easy using the `@emotion/cache` package, and after inspecting how `@emotion/css` sets up the default cache, I settled on the following code:

```javascript
import createCache from '@emotion/cache';
import { cache } from '@emotion/css';
import { serializeStyles } from '@emotion/serialize';

export const globalThemeCache = createCache({ key: 'global-theme' });

export function injectThemedGlobal(...args) {
  const serialized = serializeStyles(args, cache.registered);

  if (!globalThemeCache.inserted[serialized.name]) {
    globalThemeCache.insert('', serialized, globalThemeCache.sheet, true);
  }
}

export function flushThemedGlobals() {
  globalThemeCache.sheet.flush();
  globalThemeCache.inserted = {};
  globalThemeCache.registered = {};
}
```

This is pretty much the same as the default cache, but instead, we create a separate `globalThemeCache` that we insert global styles into using the `injectThemedGlobal` function.

To make this a little nicer to work with, notice that I've used the default `cache` instance when serializing the styles (instead of our `globalThemeCache`). This allows us to continue interpolating styles that have been created using the default cache (e.g. using the `css` function), into our global styles. As a reminder, my global styles looked like this:

```javascript
import { css } from '@emotion/css';

const themeStyles = (theme) => css`
  // Some styles...
`;

injectThemedGlobal`
	${themeStyles(theme)};
`;
```

Without this, we would find that the `themeStyles` would not be interpolated correctly into `injectThemedGlobal` and the styles would basically be broken.

An interesting alternative would be to create a custom `css` function that hooks into our `globalThemeCache`. However, this is a lot of extra code and has a major downside in that we can't easily re-use other styles created for the default cache.

Anyway, the final code for my `GlobalStyles` component looked something like this:

```jsx
export const GlobalStyles = () => {
  const theme = useTheme();

  useEffect(() => {
    injectThemedGlobal`
      ${themeStyles(theme)};
    `;

    return () => flushThemedGlobals();
  }, [theme]);

  return null;
};
```

Upon changing the theme, we only update the styles passed to `injectThemedGlobal` and we no longer remove all of the app's styles like previously.

As a side note, it's a little disingenuous to say that the styles were updated. In reality, we are simply removing all of the `globalThemeCache` styles from the DOM using `flushThemedGlobals`, before re-inserting a new set of styles.

## Final thoughts on Emotion

Emotion is one of my favourite CSS-in-JS libraries, but it's a little unfortunate it doesn't handle global style updates out of the box. Theme changing isn't a really uncommon scenario, so it would be good for it to handle this internally.

Having gained a little understanding of how it works under-the-hood, the main problem is that there is no way to update or remove specific styles. It seems that we need a more granular way to interact with the cache, other than `flush`. This would allow us to remove or update only our theme's global styles without needing an entirely separate cache.

Hopefully this is something Emotion will provide for us in an upcoming version!
