## Permissions-Policy to disable unload handler

## What is this?

This is a proposal for a new [Permissions-Policy](https://github.com/w3c/webappsec-permissions-policy/blob/main/permissions-policy-explainer.md) entry
that will disable unload handlers.
Permissions-Policy is a mechanism for individual pages
to opt out of features.
This proposal would add a way
to allow a page to opt out of firing unload handlers.

## Motivation

Unload handlers have been [unreliable](https://developers.google.com/web/updates/2018/07/page-lifecycle-api#the-unload-event) for a long time.
Unload handlers are not guaranteed to be fired
on all cases where a document gets destroyed,
especially on mobile platforms
where tabs are more likely to get killed in the background.

Additionally, unload handlers have a problematic relationship with [BFCache](https://web.dev/bfcache/).
When entering BFCache a page receives a pagehide event
but not an unload event.
If the user never returns to the page,
the page will be destroyed without ever receiving an unload event.
For a page with unload event handlers,
we must choose whether to allow BFCacheing
or whether to make the unload event reliable.

In some browsers like Desktop Chrome and Desktop Firefox,
the existence of unload handlers makes a page ineligible for BFCache,
hurting performance.
In others like Android Chrome & Android [Firefox](https://groups.google.com/a/mozilla.org/g/dev-platform/c/3pQRLPZnQgQ/m/dsA1w4iiAwAJ),
and WebKit-based browsers (Desktop Safari & all iOS browsers),
if the page is eligible for BFCache,
the unload handlers will not run,
further increasing the unreliability of unload handlers.

For best performance,
sites should avoid using unload handlers.
Alternatives exist for most uses,
see https://web.dev/bfcache/#never-use-the-unload-event for recommendations.

For sites that have removed all of their unload handlers,
we would like them to have way to ensure
that no new handlers are introduced in the future.
In addition to that,
sometimes getting rid of unload handlers from the page is not easy
e.g. in the case when a third party code library uses unload handlers silently.
This proposal gives the site
a way stop firing unload handlers
even if they are added by code
outside of their control.

If a page uses this new Permissions-Policy,
the page will be:

*   More likely to be eligible with BFCache
*   Protected from accidentally introducing unload handlers
*   More predictable as the unload handler will never run,
    instead of maybe running on Desktop but not on Chrome Android/Safari/etc)

## Goals

*   Provide a way to ensure that a page is free from unload handlers.
*   Provide a way to force-disable embedded iframe’s unload handlers from the iframe embedder side.

## Non-goals

*   Providing a way to make a page BFCache-able unconditionally.

## Proposed Solution

*   Add `unload` as a [policy-controlled feature](https://www.w3.org/TR/permissions-policy-1/#features).
*   Make [default allowlist](https://www.w3.org/TR/permissions-policy-1/#default-allowlists) `*` for the unload feature
    so that the existing sites behave the same as before.
    Use of unload events is allowed in documents in top-level browsing contexts by default,
    and when allowed,
    use of unload events is allowed by default to documents in child browsing contexts.

## Examples

### Example 1 - Disable unload events under specific iframe

The following sample shows how to stop firing unload handlers under a specific iframe.
The unload handlers in the nested iframes inside this iframe also stop firing.
There is no way to re-enable unload events from the child iframes
once the parent iframe decides to ignore unload handlers.

```
<iframe src="https://iframe.com/" allow="unload 'none'"></iframe>
```

### Example 2 - Disable unload events entirely

When site authors want to disable unload events for the current frame
and all of its child iframes recursively,
site authors can use the following HTTP response header
when serving the document.

```
Permissions-Policy: unload=()
```

### Example 3 - Report without disabling

When site authors want to know if unload handlers are present in their main frame,
the following header will cause that to be reported.
See [this doc](https://github.com/w3c/webappsec-permissions-policy/blob/main/reporting.md#can-i-just-trigger-reports-without-actually-enforcing-the-policy) for more details on reporting.
Unfortunately, reporting for sub-frames has privacy implications,
so it won't be permitted.

```
Permissions-Policy-Report-Only: unload=()
```

## Considered alternatives - Document-Policy

[Document-Policy](https://github.com/WICG/document-policy/blob/main/document-policy-explainer.md) is another possible way to disable unload handlers.
For example, site authors could use the following HTTP response header.

```
Document-Policy: unload=?0
```

Different from Permissions-Policy,
Document-Policy can not be used to force disabling its child frames' unload handlers recursively.
In Chrome, even if the main frame doesn’t use unload handlers at all,
the iframe’s unload handlers can block Chrome’s BFCache.

## Considered alternatives - Disabling from JS

It's possible to disable the current document's unload handlers via JavaScript
(e.g. with the following script),
but it isn't perfect.
The following script will only work
if it runs before any other script that adds unload handlers
(which is hard to guarantee),
it also can't disable unload handlers on child frames, etc.

https://chikamune-cr.github.io/how_to_disable_unload_handlers_by_monkeypatch/

```js
// Ignore the window.onunload attribute.
Object.defineProperty(window, 'onunload', {set: function(handler) {}});
if (!window.original_addEventListener) {
  window.original_addEventListener = window.addEventListener;
  function addEventListener_monkeypatch(event_type, handler, opt) {
    // Ignore unload handler.
    if (event_type !== 'unload') {
      this.original_addEventListener(event_type, handler, opt);
    } else {
      // We can also detect the usage of unload handlers.
      console.trace('unload handlers were used');
    }
  }
  window.addEventListener = addEventListener_monkeypatch;
}
```

## Considered alternatives - reloading iframes

We’ve considered unloading iframes on navigating away
if the iframes are cross-origin
and make the page ineligible for BFCache,
then reloading the unloaded iframes
when the parent frame is salvaged from BFCache.
This was rejected as it might cause unexpected behavior
when frames communicate with each other.
(We're open to suggestions)

## Frequently Asked Questions

### Will this Permissions-policy also affect the unload handlers added by extensions?

Yes.
It doesn't seem possible to special case unload handlers from extensions.
Fortunately, we believe that there are alternatives to unload
that can be used by extensions.
We've proactively reached out to popular extensions known to use unload handlers
to verify that these alternatives are viable.
We will work on increasing awareness and providing guidance
for a smooth migration.
We welcome feedback and questions
from extensions authors on this topic.
