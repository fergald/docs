# Explainer: CSS Shadow ::part and ::theme

## About this doc

This explainer is adapted from [Monica Dinculescu](https://meowni.ca/about/)'s [original explainer](https://meowni.ca/posts/part-theme-explainer/)
by [Fergal Daly](mailto:fergal@chromium.org)
and updated to match the latest spec.

## Motivation

[Shadow DOM](https://www.w3.org/TR/shadow-dom/) is a spec
that gives you DOM and style encapsulation.
This is great for reusable [web components](https://meowni.ca/posts/web-components-with-otters/),
as it reduces the ability of these components’ styles
getting accidentally stomped over
(the old "I have a class called `button`
and you have a class called `button`,
now we both look busted" problem),
but it adds a barrier for styling and theming these components deliberately.

When talking about styling a component,
there are usually two different problems you might want to solve:

- *Styling*: I am using a third-party `<fancy-button>` element on my site
and I want this one to be blue.

- *Theming*: I am using many third-party elements on my site,
and some of them have a `<fancy-button>`;
I want all the `<fancy-button>`s to be blue.

### Previous attempts at solving this problem

There have been several previous attempts at solving this,
some more successful than others.
More detail is available in [this post](https://meowni.ca/posts/styling-the-dome/).
The short version is:

- First came `:shadow` and `/deep/`
  (which have since been deprecated, and removed as of Chrome 60).
  These were shadow-piercing selectors
  that allowed you to target any node in an element’s Shadow DOM.
  Apart from being terrible for performance,
  they also required the user of an element
  to be intimately familiar with some random element’s implementation,
  which was unlikely
  and lead to them just breaking the whole element by accident

- [Custom properties](https://developer.mozilla.org/en-US/docs/Web/CSS/--*) allow you to create custom CSS properties
  that can be used throughout an app.
  In particular, they pierce the shadow boundary,
  which means they can be used for styling elements with a Shadow DOM:
  If `<fancy-button>` uses a `--fancy-button-background` property to control its background,
  then:
  ```css
  fancy-button#one { --fancy-button-background: blue; } /* solves the styling problem and */
  fancy-button { --fancy-button-background: blue; } /* solves the theming problem */
  ```

- The problem with using just custom properties for styling/theming
  is that it places the onus on the element author
  to basically declare every possible styleable property as a custom property.
  As a result, `@apply` was proposed,
  which basically allowed a custom property to hold an entire ruleset
  (a bag of other properties!). 
 [Tab Atkins](https://twitter.com/tabatkins) has a [post](https://www.xanthir.com/b4o00)
  on why this approach was abandoned,
  the tl;dr; is that it interacted pretty poorly with pseudo classes and elements
  (like `:focus`, `:hover`, `::placeholder` for input),
  which still meant the element author would have to define a looooot of these bags of properties
  to be used in the right places.
  
## A different approach

The current new [proposal](https://drafts.csswg.org/css-shadow-parts-1/) is `::part` and `::theme`,
a set of pseudo-elements that allow you to style inside a shadow tree,
from outside of that shadow tree.
Unlike `:shadow` and `/deep/`,
they don’t allow you to style arbitrary elements inside a shadow tree:
they only allow you to style elements that an author has tagged as being eligible for styling.

### How ::part works

You can specify a "styleable" part on any element in your shadow tree:

```html
<x-foo>
  #shadow-root
  <div part="some-box"><span>...</span></div>
  <input part="some-input">
  <div>...</div> <!-- not styleable -->
</x-foo>
```

If you’re in a document that has an `<x-foo>` in it, then you can style those parts with:

```css
x-foo::part(some-box) { ... }
```

You *can* use other pseudo elements or selectors
(that were not explicitly exposed as shadow parts),
so both of these work:

```css
x-foo::part(some-box):hover { ... }
x-foo::part(some-input)::placeholder { ... }
```

You *cannot* select inside of those parts,
so this *doesn’t* work:

```css
x-foo::part(some-box) span { ... } nor
x-foo::part(some-box)::part(some-other-thing) { ... }
```

You *cannot* style this part more than one level up if you don’t forward it.
So without any extra work,
if you have an element that contains an `x-foo` like this:

```html
<x-bar>
  #shadow-root
  <x-foo></x-foo>
</x-bar>
```

You *cannot* select and style the `x-foo`’s part like this:

```css
x-bar::part(some-box) { ... }
```

### Forwarding parts

You *can* explicitly forward a child’s part
to be styleable outside of the parent’s shadow tree.
So in the previous example,
to allow the some-box part to be styleable by `x-bar`’s parent,
it would have to be exposed:

```html
<x-bar>
  #shadow-root
  <x-foo exportparts="some-box: foo-some-box"></x-foo>
</x-bar>
```

The `::part` forwarding syntax has options a-plenty.

- `exportparts="some-box, some-input"`:
  explicitly forward `x-foo`’s parts that you know about
  (i.e. some-box and some-input) as they are.
  These selectors *would* match:

  ```css
  x-bar::part(some-box) { ... }
  x-bar::part(some-input) { ... }
  ```

- `exportparts="some-input: foo-input"`:
  explicitly forward (some) of `x-foo`’s parts (i.e. some-input)
  but rename them.
  These selectors *would* match:

  ```css
  x-bar::part(foo-input) { ... }
  ```

  These selectors *would not* match:

  ```css
  x-bar::part(some-box) { ... }
  x-bar::part(some-input) { ... }
  x-bar::part(foo-box) { ... }
  ```

#### Forwarding with -*
*Note: `-*` forwarding is currently not in the spec.
The following is how it could work.*

- `part="* foo-*"`: implicitly forward all of `x-foo`’s parts as they are, but prefixed.
  These selectors *would* match:

  ```css
  x-bar::part(foo-some-box) { ... }
  x-bar::part(foo-some-input) { ... }
  ```

  These selectors *would not* match:

  ```css
  x-bar::part(some-box) { ... }
  x-bar::part(some-input) { ... }
  ```

  You _can_ chain these,
  as well as add a part to `x-foo` itself
  (`some-foo` below).
  This means "style this particular `x-foo`,
  but not the other one,
  if you had more".
  All of these are valid:

  ```html
  <x-bar>
    #shadow-root
    <x-foo exportparts="*: foo-*"></x-foo>
    <!-- or -->
    <x-foo exportparts="some-input: foo-input"></x-foo>
  </x-bar>
  ```

- You cannot forward all parts at once,
  i.e. `exportparts="*: *"`
  since this might break your element in the future
  (if the nested shadow element adds new parts).
  So this is invalid:

  ```html
  <x-form>
    #shadow-root
    <x-bar exportparts="*: *">
      #shadow-root
      <x-foo exportparts="*: *"></x-foo>
    </x-bar>
  </x-form>
  ```

- However, as mentioned,
  you can forward all the parts if you prefix them,
  so this is ok:
  ```html                                    
  <x-form>
    #shadow-root
    <x-bar part="* bar-*">
      #shadow-root
      <x-foo part="* foo-*"></x-foo>
    </x-bar>
  </x-form>
  ```

  This selector would be valid:

  ```css
  x-form::part(bar-foo-some-input) { ... }
  ```

## The "all buttons in this app should be blue" theming problem

Given the above prefixing rules,
to style all inputs in a document at once,
you need to ensure that all elements correctly forward their parts
and select all their parts.

So given this shadow tree:

```html
<submit-form>
  #shadow-root
  <x-form exportparts="some-input, some-box">
    #shadow-root
    <x-bar exportparts="some-input, some-box">
      #shadow-root
      <x-foo part="some-input, some-box"></x-foo>
    </x-bar>
  </x-form>
</submit-form>

<x-form></x-form>
<x-bar></x-bar>
```

You can style all the inputs with:

```css
:root::part(some-input) { ... }
```

👉 This is a lot of effort on the element author,
but easy on the theme user.

If you hadn’t forwarded them with the same name
and some-input was used at every level of the app
(the non contrived example is just an `<a>` tag that’s used in many shadow roots),
then you’d have to write:

```css
:root::part(form-bar-foo-some-input),
:root::part(bar-foo-some-input,
:root::part(foo-some-input),
:root::part(some-input) { ... }
```

👉 This is a lot of effort on the theme user,
but easy on the element author.

Both of these examples show that if an element author forgot to forward a part,
then the app can’t be themed correctly.

### How ::theme works
**Note that the ::theme Pseudo-element was never implemented in the spec and does not work.
It is uncertain if this feature will be developed going further.**
Elements can have a `theme` tag, similar to the `part` tag
but unlike `::part`, `::theme` matches elements parts with that theme name,
anywhere in the document. 
This means that even without forwarding parts, i.e.:

```html
<x-bar>
  #shadow-root
    <x-foo></x-foo>
    <x-foo></x-foo>
    <x-foo></x-foo>
</x-bar>
```

You could style all of the inputs in `x-bar` with:
              
```css
x-bar::theme(some-input) { ... }
```

This can go arbitrarily deep in the shadow tree.
So, no matter how deeply nested they are,
you could style all the inputs with `theme="some-input"` in the app with:

```css
:root::theme(some-input) { ... }
```

## Demo

Chrome 69 shipped with `::part` working behind a flag.
Starting Chrome with

```sh
google-chrome --enable-blink-features=CSSPartPseudoElement
```

will allow you to try this out.
It does not support the `-*` syntax to forward multiple parts.

## Other future work

Feedback from developers has identified several areas for future work.

### Access control

[Issue](https://github.com/w3c/csswg-drafts/issues/3506)

Component authors may want to restrict what style-properties of a part can be
set by users of the component, e.g. allowing changes to color but not height.

### Controlling theme

[Issue](https://github.com/w3c/csswg-drafts/issues/3507)

The `::theme` selector, as described above,
allows deep components to expose themselves for theming
with no control allowed by the containing components.
In real usage it's likely that component authors
will want more control than this.

This could be some kind of white-/black-listing of themes by components or
defining themes by patterns against part names.
