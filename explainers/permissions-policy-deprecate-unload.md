# Disable unload handlers by default and add Permissions-Policy to opt-in to enabling them.

## What is this?

This is a proposal to disable running `unload` handlers by default
and to add a new [Permissions-Policy](https://github.com/w3c/webappsec-permissions-policy/blob/main/permissions-policy-explainer.md) entry
that will allow them to be re-enabled
by sites which cannot easily stop using them.

## Background

### Unload reliability

According to [Nic Jansma's survey](https://nicj.net/beaconing-in-practice/#beaconing-reliability-unload),
`unload` handlers run 95%+ of the time on desktop Chrome, Edge and Firefox
but considerably less (57%-68%) on mobile browsers and Safari desktop
which already prioritize BFCaching over running `unload` handlers.

`unload` handlers are unreliable on mobile
because mobile OSes kill tabs and apps in the background.
For some time,
developers have been [advised](https://developers.google.com/web/updates/2018/07/page-lifecycle-api#the-unload-event)
not to use `unload` handlers.

### Unload as specced

`unload` handlers are [currently specced](https://whatpr.org/html/4288/browsing-the-web.html#unloading-documents) to run
when the document is destroyed
*unless* "the user agent does not intend to keep document alive in a session history entry".
This means that `unload` handlers only runs if the document will not enter BFCache,
i.e. a main-frame navigation where BFCaching is not allowed or
a subframe navigation.

This was specced in this [PR](https://github.com/whatwg/html/pull/5889).

### Unload as implemented.

WebKit's implementation matches the spec.
Mozilla's and Chrome's match the spec on mobile
but on desktop they both give priority to running `unload` handlers
and block BFCache if an `unload` handler is present.
In Chrome's case we felt that since it was already unreliable on mobile
that making it moreso was not a problem.
However with 95% reliability on desktop,
reducing that significantly would be a problem.

### Whether unload handlers will run is not predictable

Whether `unload` handlers run is deterministic
but usually,
neither main frames nor subframes
have enough information
to know if they will run.

As specced the only way to reliably predict
whether an `unload` handler will run
when the main frame navigates
is to carefully force it run or to not run.
I.e.

- ensure all frames of the page are compatible with BFCache.
  This mostly precludes having frames from 3rd party origins
- ensure at least one frame of the page is not compatible with BFCache.
  This prevents BFCacheing, hurting performance.

Without doing one or the other,
whether the `unload` handler runs or not
is dependent on state that is hard or impossible to see.

### Unload is biased

If you collect data via `unload` handlers,
some fraction of this data will be missing
due to handlers that did not run.
When `unload` handlers running depends on BFCache eligibility,
and that in turns depends the activity of 3rd party subframes,
there could be significant bias
in which data is missing.
It may be that whether the data is collected
depends what actions the user took
or which ad network's ads were shown.
This bias means that compensating for this missing data
may be hard or impossible.

### Previous proposal

Chrome's [original proposed][previous] to allow sites
to opt-in to disabling `unload` handlers.
This was an attempt to move the ecosystem
away from `unload` handlers,
making it easier to align with the spec eventually.
It found no support with other vendors.

## Motivation

If we were starting from scratch,
there would not be an `unload` handler.
Keeping all other things constant,
the web platform would be better without it
If we're going to make a disruptive change,
let's aim for the best end-point.

### Reliable unload handlers are incompatible with BFCache

As they are currently specced,
the only way,
for a complex site,
to make `unload` handlers reliable, predictable and unbiased
is to force them to run by blocking BFCache.

### Unload handlers are rarely the correct choice

This [doc](https://developer.chrome.com/blog/page-lifecycle-api/#the-unload-event) on the page lifecycle
covers the problems with unload
and the better alternatives to use.

### Aligning with the current spec is disruptive

[As pointed out](https://github.com/whatwg/html/pull/5889#pullrequestreview-505683024) when the spec was being updated,
skipping `unload` handlers
is a breaking change.
On platforms where unload is 95% reliable,
it requires a careful rollout.
Making a breaking change
to get to an end-point where
unload is even more of a foot-gun
is not ideal.

If Chrome and Mozilla are to undertake a disruptive change,
this is an opportunity to reach a better end-point,
where unload is gone by default
but still available for those who opt-in.

### Sites are Trying to Keep Unload Away

Some sites have removed some or all of their `unload` handlers,
but with large complex sites,
even without 3rd-party scripts and iframes,
it is hard to ensure
that nobody introduces an `unload` handler.

By disabling `unload` handlers by default,
this will prevent new instances from introduces inadvertently.

## Proposed Solution

*   Add `unload` as a [policy-controlled feature](https://www.w3.org/TR/permissions-policy-1/#features).
*   Use [default allowlist](https://www.w3.org/TR/permissions-policy-1/#default-allowlists) `()` for the `unload` feature

### Logistics of deprecation

We do no think we can suddenly flip this switch.
We would like to roll it out gradually
so that sites who miss the announcement
still have time to react
before being heavily disrupted.

A straw-person proposal for the rollout would be

- Enable permissions-policy for `unload`
  with a default allowlist of `*`
- Flip the default allowlist to `()` for N% of navigations
- Increase N to 100 over time

Selecting the N% of navigations needs some care.

- Rolling out by user
  would inflict full breakage on a fraction of users.
- Rolling out by site/URL would suddenly inflict full breakage
  on a fraction of sites.
- Rolling out at random would lead to problems
  that are hard to detect and reproduce.
- Rolling out by some combined hash of user
  and site/URL is more complex
  but would avoid all of the problems above.

## Considered alternatives

The [original proposal][previous] had a default allowlist of `*`.
It required sites to opt in to disabling `unload` handlers.
There were concerns that this gave sites a way
to control the execution of code in iframes.

Disabling by default avoids this problem.

Several other alternatives are discussed
in the [original proposal][previous-considered-alternatives].

## Privacy Considerations

None.

## Security Considerations

There would be a period where
the default allowlist for the policy would be `*`.
This was discussed in detail in the [previous proposal].

The currernt proposal makes this a tempoarary state.

No other concerns are known.

# [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/)

01.  What information might this feature expose to Web sites or other parties,
     and for what purposes is that exposure necessary?

     > None.

02.  Do features in your specification expose the minimum amount of information
     necessary to enable their intended uses?

     > Yes. No information is directly exposed.
     > Some information about whether a subframe has an `unload` event handler
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

     > No.

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
     > 1st party can re-enable for all 3rd parties.
     > 3rd parties can disable the feature for themselves and their subframes.

14.  How do the features in this specification work in the context of a browser’s
     Private Browsing or Incognito mode?

     > No difference.

15.  Does this specification have both "Security Considerations" and "Privacy
     Considerations" sections?

     > Yes.

16.  Do features in your specification enable origins to downgrade default
     security protections?

     > No.

17.  How does your feature handle non-"fully active" documents?

     > N/A

18.  What should this questionnaire have asked?

     > ?


[previous]: permissions-policy-unload.md
[previous-considered-alternatives]: permissions-policy-unload.md#considered-alternatives
[previous-security]: permissions-policy-unload.md##security-considerations
