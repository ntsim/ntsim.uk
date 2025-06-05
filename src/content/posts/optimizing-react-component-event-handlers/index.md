---
id: '0bad1ec2-413e-4b10-a923-23e389bf4dd9'
title: 'Optimizing React component event handlers'
description: "Event handlers for React components are a common source of performance issues. In this post, we'll learn how to avoid identify and avoid them."
date: '2019-07-06 16:06:12'
tags:
  - React
  - JavaScript
---

For any seasoned React developer, it should be fairly common knowledge that event handlers are one
of the main sources of performance issues in React applications. The main reasons for this are due
to the following:

1. It is easy (and often more readable) to write event handlers as inline (typically arrow)
   functions. However, the catch here is that new instances of these functions will be created every
   time the component renders.
2. Out of the box `memo` or `PureComponent` only shallow compare props. Consequently, references to
   event handler functions need to be maintained for these optimizations to actually work. This is at
   odds with using inline arrow functions as event handlers (see point 1).
3. React components do not differentiate between event handlers and other props and consequently
   it's entirely up to the developer to control how these are optimized.

In this post, we'll look at the implications of the above and how we can actually work to solve them.

## The problem

Typically, the common advice around performance is to avoid premature optimization, however, we've
mentioned earlier it is quite easy to fall into performance traps due to these pesky event handlers.
Let's look at one of the more common situations I've ran into, with the following example:

```javascript
const Accordion = ({ sections }) => {
  const [openSections, setOpenSections] = useState([]);

  return (
    <div>
      {sections.map((section) => (
        <AccordionSection
          open={openSections.includes(section.id)}
          onToggle={() => {
            const nextOpenSections = Array.from(openSections);

            if (nextOpenSections.includes(section.id)) {
              nextOpenSections.splice(nextOpenSections.indexOf(section.id), 1);
            } else {
              nextOpenSections.push(section.id);
            }

            setOpenSections(nextOpenSections);
          }}
        />
      ))}
    </div>
  );
};

const AccordionSection = ({ open, onToggle }) => (
  <div className={open ? 'section--open' : 'section--hidden'}>
    <button onClick={onToggle} type="button">
      Toggle
    </button>
  </div>
);
```

This code looks fairly innocuous on the surface, but toggling a section will cause a re-render of
the entire `Accordion` and every one of its `AccordionSection` children. If you're like me, you were
probably expecting only the specific section that was toggled to re-render. For a few sections this
probably won't cause any problem, but this can become problematic depending on the number of
sections and the amount of content in each one. Ultimately, this can cause toggling to feel sluggish
and have some visible jank.

Why do the other sections need to re-render? Their states haven't been changed, so there shouldn't
be any need to. As it turns out, React re-renders all children by default when a parent changes.

The React team are aware that this can be a performance trap, so they encourage us to use `memo` or
`PureComponent` on the child components:

```javascript
const AccordionSection = memo(({ open, onToggle }) => (
  <div className={open ? 'section--open' : 'section--hidden'}>
    <button onClick={onToggle} type="button">
      Toggle
    </button>
  </div>
));
```

Unfortunately, we're using an inline arrow function for each section's `onToggle`, and this
completely negates the usage of `memo`. We've preferred to use an inline arrow function here as the
`onToggle` event handler needs to be dynamically generated for each of our sections.

## A first attempt with class components

As we've discussed earlier, we know that just using `memo` or `PureComponent` is not going to solve
our problem as we know `Accordion` will always create new instances of the `handleToggle` function.
Some Googling might lead us down the route of converting the `Accordion` to a class component:

```javascript
class Accordion extends Component {
  state = {
    openSections: [],
  };

  handleToggle = (option) => () => {
    // code removed for brevity...
  };

  render() {
    const { sections } = this.props;
    const { openSections } = this.state;

    return (
      <div>
        {sections.map((section) => (
          <AccordionSection
            open={openSections.includes(section.id)}
            onToggle={this.handleToggle(section)}
          />
        ))}
      </div>
    );
  }
}
```

Great. As we've made the handler into a class method, this should allow us to maintain the same
function reference between renders right?

Nope!

The `handleToggle` method is a higher-order function (could also be called a 'callback factory') and
will just return a new function instance on each invocation. This is essentially the same as the
inline arrow function, but we've just moved the code around.

## Functional components and `useCallback`

How about using a functional component using the `useCallback` hook? You've probably read somewhere
that's what you should use for event handlers and it sounds promising.

Let's re-implement our component again:

```javascript
const Accordion = ({ sections }) => {
  const [openSections, setOpenSections] = useState([]);

  const handleToggle = useCallback(
    (option) => () => {
      // code removed for brevity...
    },
    [openSections],
  );

  return (
    <div>
      {sections.map((section) => (
        <AccordionSection
          open={openSections.includes(section.id)}
          onToggle={handleToggle(section)}
        />
      ))}
    </div>
  );
};
```

No luck unfortunately as re-rendering still occurs! Changing it slightly to a `useMemo`
implementation doesn't help either. What gives?

Unfortunately, using `useCallback` like this is essentially the same as the component class method
implementation from earlier. As previously discussed, `handleToggle` is a higher-order function and
returns a new function instance each time. The problem here is that `useCallback` only memoizes the
callback function itself, and not the result of that callback (in this case, another function).

## Converting to first-order functions

Our first (and simplest) option is to not work with higher-order functions at all and just convert
them back to first-order functions. For the non-mathematical, this is just the technical term for a
function that returns a value that isn't another function.

To do this, we can re-implement `handleToggle` so that our `Section` component provides more
information about the section through the callback. In this case, an `id` prop:

```javascript
const AccordionSection = memo(({ id, open, onToggle }) => (
  <div className={open ? 'section--open' : 'section--hidden'}>
    <button onClick={() => onToggle(id)} type="button">
      Toggle
    </button>
  </div>
));
```

The `Accordion` can now take advantage of `useCallback` in its originally intended way:

```javascript
const Accordion = ({ sections }) => {
  const [openSections, setOpenSections] = useState([]);

  const handleToggle = useCallback(
    (sectionId) => {
      // code removed for brevity...
    },
    [openSections],
  );

  return (
    <div>
      {sections.map((section) => (
        <AccordionSection
          id={section.id}
          open={openSections.includes(section.id)}
          onToggle={handleToggle}
        />
      ))}
    </div>
  );
};
```

The only downside to this approach is that we now have to pass more props like `id` through to
`AccordionSection` just so we can pass it back to the parent `Accordion` through the callback. If we
need several props like this, this approach can get a bit unwieldy, especially if we also have to
move those props through other children of `AccordionSection`.

If you're just looking for the easiest way of dealing with higher-order functions, I would look no
further as the following alternative is a lot more involved (although much more interesting).

You have been warned!

## An alternative using memoization

Our original implementation took advantage of the fact we already had each `section` item in scope
during the `sections.map` loop and in certain situations, this might be exactly what we need. To
this end, we need to leverage some additional memoization to make this work.

React doesn't bundle a function that fulfils our needs here, so we need to shop around a little.
There are a bunch of packages out there that provide this functionality, but I've personally had
success using [memoizee](https://github.com/medikoo/memoizee) so would probably recommend that.

Having picked our memoization function, we can now it with `useCallback` in the following way:

```javascript
const Accordion = ({ sections }) => {
  const [openSections, setOpenSections] = useState([]);

  const handleToggle = useCallback(
    memoize((section) => {
      // code removed for brevity
    }),
    [],
  );

  return (
    <div>
      {sections.map((section) => (
        <AccordionSection
          open={openSections.includes(section.id)}
          onToggle={handleToggle(section)}
        />
      ))}
    </div>
  );
};
```

Note that we don't give the `useCallback` dependency array anything. This is to prevent `useCallback`
from invalidating its cache when `openSections` changes and causing a re-render.

We're pretty close at this point, but this approach is flawed as we will find that clicking on our
`AccordionSection` button only works once and will get stuck in the open state! It might not be
super obvious, but we've been bitten by Javascript's closure behaviour. Remember that `handleToggle`
looks like this internally:

```javascript
const [openSections, setOpenSections] = useState([]);

const handleToggle = useCallback(
  memoize((section) => {
    return () => {
      const nextOpenSections = Array.from(openSections);

      if (nextOpenSections.includes(section.id)) {
        nextOpenSections.splice(nextOpenSections.indexOf(section.id), 1);
      } else {
        nextOpenSections.push(section.id);
      }

      setOpenSections(nextOpenSections);
    };
  }),
  [],
);
```

The memoized function captures the value of `openSections` in the initial render and will continue
to use the initial value whenever the function is called. We _could_ prevent this by invalidating
the `useCallback` cache (by adding `openSections` to its dependency array), but as we've mentioned
before, this would cause re-rendering and we'd just end up back at square one.

There are two approaches we can use to get around some of these issues.

### Using refs

To avoid receiving stale state in our memoized callback, we can use a ref to store the most
up-to-date state.

```javascript
const [openSections, setOpenSections] = useState([]);
const openSectionsRef = useRef(openSections);

const handleToggle = useCallback(
  memoize((section) => {
    return () => {
      // code removed for brevity
      openSectionsRef.current = nextOpenSections;
      setOpenSections(nextOpenSections);
    };
  }),
  [],
);
```

Unfortunately, this also means we have to store the current state in two places. This puts the
burden on us to remember to update the state in both these places and could be troublesome if there
are other places where the state can be changed.

### Using a reducer

An alternative to refs is to use the `useReducer` hook. Re-implementing our `Accordion` to use it
would look like this:

```javascript
const reducer = (state, action) => {
  switch (action.type) {
    case 'toggle':
      // code removed for brevity
      return { openSections };
    default:
      throw new Error();
  }
};

const Accordion = ({ sections }) => {
  const [{ openSections }, dispatch] = useReducer(reducer, {
    openSections: [],
  });

  const handleToggle = useCallback(
    memoize((section) => {
      return () => dispatch({ type: 'toggle', id: section.id });
    }),
    [],
  );

  return (
    <div>
      {sections.map((section) => (
        <AccordionSection
          open={openSections.includes(section.id)}
          onToggle={handleToggle(section)}
        />
      ))}
    </div>
  );
};
```

`useReducer` is advantageous here as we can only change state via the `dispatch` method.
Consequently, we no longer hold onto a stale version of `openSections` during `handleToggle` and we
can now freely toggle our `AccordionSections` to our heart's content.

This approach obviously incurs a lot of extra overhead, so it's definitely not ideal for most
situations. We should also bear in mind that memoization can lead to other problems (such as
stale caches) if used incorrectly.

### Using class components

If we're using class components, memoization is actually a lot simpler as we don't need to deal with
with hooks like `useCallback` or `useReducer`. We can basically just wrap our `handleToggle` method
with `memoize`, and call it a day:

```javascript
class Accordion extends Component {
  handleToggle = memoize((section) => {
    // code removed for brevity
  });

  render() {
    const { sections } = this.props;
    const { openSections } = this.state;

    return (
      <div>
        {sections.map((section) => (
          <AccordionSection
            open={openSections.includes(section.id)}
            onToggle={this.handleToggle(section)}
          />
        ))}
      </div>
    );
  }
}
```

It seems there are definitely still advantages to using class components in the post-hooks world!

## Thoughts on React and event handlers

Having worked through the various problems with memoizing event handlers, I can't help but wonder
why there isn't an easier way to do this in React.

A major part of the issue is that React encourages the use of function components. Whilst they are
a nice and succinct way of expressing our components, when combined with hooks, they also introduce
new issues with stale state due to to everything being one big closure.

The current hooks API is able to give us some of the primitives we need to deal with these issues,
e.g. using dependency arrays to invalidate caches. Unfortunately, these place the onus on developers
to manually manage these dependencies to avoid unnecessary re-rendering (a topic we might discuss
further in another post).

It would be great if there was some API that allowed us to automatically memoize our event handlers.
Perhaps if we had something similar to Vue where event handlers are given their own special syntax.
Our child component might look like this:

```javascript
<AccordionSection
  open={openSections.includes(section.id)}
  @toggle={() => {
    // handleToggle code omitted for brevity
  }}
/>
```

When binding the event handler using `@toggle` (or something similar), React could automatically
memoize the provided function. Admittedly, there are still potential issues with stale state still
being consumed within the event handler, so we also need a better way to receive the most up-to-date
state (without having to rely on refs).

I'm not entirely sure what the full implications of this would be, but having something like this
could be quite useful. Unfortunately, I doubt we'll be seeing such a dramatic syntactical addition
anytime soon though.

## Conclusion

Using higher-order functions to produce event handlers in React is tricky as we can easily run
into performance issues due to new function references being created.

We can employ a variety of memoization techniques to continue using higher-order functions, however,
we then have to deal with the additional complexity of stale state and cache invalidation.

The easiest (and recommended) way to deal with these issues is to refactor our components and
convert our higher-order event handlers into first-order event handlers. This way, we can completely
side-step having to deal with their memoization.

Like many things in programming, the best solution to a problem is not to have the problem in the
first place.
