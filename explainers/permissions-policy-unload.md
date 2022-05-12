## Permissions-Policy to disable unload handler

## What is this?

This is a proposal for a new [Permissions-Policy](https://github.com/w3c/webappsec-permissions-policy/blob/main/permissions-policy-explainer.md) entry
that will disable unload handlers.
Permissions-Policy is a mechanism for controlling individual page's
access to features.
This proposal would add a way for a page to opt out of firing unload handlers
while allowing exceptions.

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

Some browsers, like Desktop Chrome and Desktop Firefox,
have chosen to keep the unload event reliable,
so the existence of unload handlers makes a page ineligible for BFCache,
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
We are, in parallel, proposing the [unload beacon API](https://github.com/darrenw/docs/blob/main/explainers/beacon_api.md)
which makes migrating away from unload easier for some common use cases.
If there are use cases that still require unload handlers
please let us know.

For sites that have removed some or all of their unload handlers,
we would like them to have way to ensure
that no new handlers are introduced in the future.
In addition to that,
sometimes getting rid of unload handlers from the page is not easy
e.g. in the case when a third party code library uses unload handlers silently.
This proposal gives the site
a way stop firing unload handlers
even if they are added by code outside of their control.

If a page uses this new Permissions-Policy,
the page will be:

*   More likely to be eligible with BFCache
*   Protected from accidentally introducing unload handlers
*   More predictable as the unload handler will never run,
    instead of maybe running on Desktop but not on Chrome Android/Safari/etc.

## Goals

*   Provide a way to ensure that a page is free from unload handlers.
*   Provide a way to lock in progress on gradually removing all unload handlers from a site.
*   Provide a way to force-disable embedded iframe’s unload handlers from the iframe embedder side.

## Non-goals

*   Providing a way to make a page BFCache-able unconditionally.

## Proposed Solution

*   Add `unload` as a [policy-controlled feature](https://www.w3.org/TR/permissions-policy-1/#features).
*   Use [default allowlist](https://www.w3.org/TR/permissions-policy-1/#default-allowlists) `*` for the unload feature
    so that the existing sites behave the same as before.
    Use of unload events is allowed in documents in top-level browsing contexts by default,
    and when allowed,
    use of unload events is allowed by default to documents in child browsing contexts.

## Examples

### Example 1 - Disable unload events entirely

When site authors want to disable unload events for the current frame
and all of its child iframes recursively,
site authors can use the following HTTP response header
when serving the document.

```
Permissions-Policy: unload=()
```

### Example 2 - Disable unload events with an exception for one origin

When site authors have cleared unload events from most of a page
and want to ensure that none creep back in
but still have a subframe that depends on unload,
site authors can use the following HTTP response header
when serving the document.

```
Permissions-Policy: unload=(https://still-uses-unload.com)
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
Document-Policy cannot be used to force disabling its child frames' unload handlers recursively.
The spec includes the `Sec-Required-Document-Policy` [header](https://wicg.github.io/document-policy/#sec-required-document-policy-http-header),
however this is not full specified ([serialization](https://wicg.github.io/document-policy/#serialization))
and remains unimplemented in browsers.
If implemented, it would force sites to choose between broken iframes and BFCache.
This would be useful on dev and staging deployments
to detect subframes that do not comply with the policy
but deploying it in production would be risky.

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

## Privacy Considerations

None.

## Security Considerations

### Concerns about giving embedders control over the (non)execution of iframe code

If the embedder of an iframe can control whether the iframe's unload handler runs or not
then that seems to provide a cross-origin control point
that could potentially be used to exploit the iframe.

While cross-origin control of scripting is, in general, a bad thing,
unload handlers are very different to other APIs.

#### Unload handlers already often don't run

Right now unload handlers for an iframe will not be run in the following cases

- mobile browser backgrounded tab killed by OS
- most mobile browsers and some desktop browsers if they believe the page could be BFCached
- any multi-process browser where a crash occurs in the process responsible for the iframe
- implementations typically allot a time budget for running unload handlers
  which may cause some not to complete or not to run at all

This means that unload handlers are already not a guaranteed capability
that subframes can depend on,
and placing some control over this nondeterminism with the parent frame
does not create a new problem.

#### Unload handlers are invisible to users

Unload handlers run *after* navigation has occurred.
The exact timing depends on many factors.
In some cases unload handlers will block the navigation until they complete,
in other cases they run in the background
in parallel with the new page loading.
In either case,
whether the unload handler runs or not
has no bearing on the immediate user experience
apart from possibly slowing down the appearance of the next page.

It is possible that skipping the unload handler will change the user experience later on,
e.g. if unload is responsible for persisting some data
in storage or over the network.
However using unload for this is already a bad idea
due to its unreliability.

Since this policy requires opt-in by the embedder,
problematic cases can be excepted until they are fixed.
What remains are the cases where the embedder
accidentally breaks something an iframe
or (hypothetically) intentionally exploits the iframe.

#### In line with sync-xhr and document-domain

Both `sync-xhr` and `document-domain` permissions
cause a long-standing feature to throw an exception.
Existing code written to use these features
was unlikely to recover well from this exception
so it was effectively a cross-origin ability to control execution
of certain pieces of script.
This was not a problem in practice.


#### Conclusion

We believe that the danger posed by embedders maliciously disabling unload handlers in iframes is minimal
and not in the same class as an ability to disable other APIs.
Meanwhile conscientious sites are unable to safeguard their work
to improve performance for users with BFCache.

## Frequently Asked Questions

### Will this Permissions-policy cause problems for 3rd-party iframes that use unload handlers?
Possibly.
Authors should ensure that
all iframes on a page have correctly without unload handlers
before using this header.
Disabling unload handlers may impact e.g. reporting of metrics.
Such issues will tend not be visible in the page itself,
since unload handlers run after document discard.
We’ve started reaching out to common 3rd party iframe providers,
asking them to remove unload handlers.
We will also encourage them to include this header on their iframes
to ensure that they stay unload-free,
once it is available.

### What should sites do if they have 3rd-party iframes that rely on unload handlers?
Sites can provide feedback to 3rd-party iframe providers,
asking them to remove unload handlers if possible.
In parallel, sites can use this header,
yet allowing origins with remaining unload handlers.
Doing this will not make the page BFCacheable
but it will help ensure that new unload handlers do not creep in elsewhere.

### Will this Permissions-policy also affect the unload handlers added by extensions?
In Chrome's implementation, yes.
We believe that there are alternatives to unload
that can be used by extensions
and so there is no need to make an exception.
We will make an effort to make extension authors aware of this new feature
so that they can proactively migrate away from unload
before sites start using this.

# [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/)

01.  What information might this feature expose to Web sites or other parties,
     and for what purposes is that exposure necessary?

     > None.

02.  Do features in your specification expose the minimum amount of information
     necessary to enable their intended uses?

     > Yes. No information is directly exposed.
     > Some information about whether a subframe has an unload event handler
     > might be deducible.

03.  How do the features in your specification deal with personal information,
     personally-identifiable information (PII), or information derived from
     them?

     > N/A

04.  How do the features in your specification deal with sensitive information?

     > N/A

05.  Do the features in your specification introduce new state for an origin
     that persists across browsing sessions?

     > No.

06.  Do the features in your specification expose information about the
     underlying platform to origins?

     > No.

07.  Does this specification allow an origin to send data to the underlying
     platform?

     > No.

08.  Do features in this specification enable access to device sensors?

     > No.

09.  Do features in this specification enable new script execution/loading
     mechanisms?

     > !!!

10.  Do features in this specification allow an origin to access other devices?

     > No.

11.  Do features in this specification allow an origin some measure of control over
     a user agent's native UI?

     > No.

12.  What temporary identifiers do the features in this specification create or
     expose to the web?

     > No.

13.  How does this specification distinguish between behavior in first-party and
     third-party contexts?

     > This disables a feature.
     > 1st party can disable for all 3rd parties.
     > 3rd parties can disable the feature for themselves and their subframes.

14.  How do the features in this specification work in the context of a browser’s
     Private Browsing or Incognito mode?

     > No difference.

15.  Does this specification have both "Security Considerations" and "Privacy
     Considerations" sections?

     > It does now!

16.  Do features in your specification enable origins to downgrade default
     security protections?

     > No.

17.  How does your feature handle non-"fully active" documents?

     > N/A

18.  What should this questionnaire have asked?

     > Does this feature allow new cross-origin control points.
