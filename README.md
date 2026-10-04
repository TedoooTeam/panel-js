# @tedooo/panel-js

The browser SDK for [Tedooo Panel](https://panel.tedooos.com) analytics.

It sends events straight to Tedooo's capture service, so there is no host to configure and no reverse proxy to run.

## Install

```bash
npm install @tedooo/panel-js
```

## Use

```js
import panel from '@tedooo/panel-js'

panel.init('<your project key>')

panel.capture('signed_up', { plan: 'pro' })
panel.identify('user-123', { email: 'ada@example.com' })
```

Call `init` once, in the browser. Importing the package in server-side code is safe, but do not call `init` or `capture` there: on a server the SDK would send events with one visitor identity shared by everyone.

When a user logs out, call `panel.reset()` so the next visitor is not attributed to them.

## Options

`init` takes an options object as its second argument. The ones most integrations use:

| Option              | Default                  | What it does                                                                                                                                         |
| ------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api_host`          | `'https://p.tedooo.com'` | Where events are sent. Set it only if you route traffic through your own domain.                                                                     |
| `autocapture`       | `true`                   | Capture clicks, form submissions and other interactions automatically.                                                                               |
| `capture_pageview`  | `true`                   | Send a `$pageview` event on load. Pass `'history_change'` for single-page apps.                                                                      |
| `capture_pageleave` | `'if_capture_pageview'`  | Send a `$pageleave` event when the visitor leaves.                                                                                                   |
| `person_profiles`   | `'identified_only'`      | Which visitors get a person profile: `'always'`, `'identified_only'` or `'never'`.                                                                   |
| `persistence`       | `'localStorage+cookie'`  | Where the visitor ID is stored: `'localStorage+cookie'`, `'localStorage'`, `'cookie'`, `'sessionStorage'` or `'memory'`.                             |
| `landing`           | none                     | The page the visit started on, as `{ url, referrer, at }` recorded by your code at page load. Pass it when `init` can run after the URL has changed. |
| `loaded`            | none                     | Callback that runs once the SDK is ready.                                                                                                            |

All options are typed; your editor lists the rest.

## Campaigns and tracking links

When `init` runs, the SDK records the page the visitor arrived on: the `utm_*` tags and ad click IDs in its URL, the referrer, and the `$initial_*` person properties. Every event of the visit then carries the campaign tags and the referrer, even if your app changes the URL before the first event is sent. (A tracking link is credited differently; see "What a tracking link needs" below.)

If capturing is off when `init` runs (`opt_out_capturing_by_default`, or `cookieless_mode: 'on_reject'` with no answer yet), nothing is stored then. The same page is recorded once capturing starts: at `opt_in_capturing()`, at `opt_out_capturing()` in `on_reject` mode, or at the first event if consent arrived another way, such as from another tab. A visitor who navigates within a single-page app before answering the consent banner keeps the page they arrived on. The page is held in memory only, so a full page load before consent runs `init` again and the new page becomes the landing page. Calling `opt_in_capturing()` again later changes nothing. `reset()` discards the landing page along with the rest of the visitor's state, also when another tab called it and this tab adopts the new visitor (`cookieWinsOnConflict`); the next visitor is recorded from the page they are on, without its campaign tags if an event was already sent from that URL, as in posthog-js, which reads the tags of a URL once. A `reset()` in this tab before anything was recorded for the visitor, as a consent manager may call before `opt_in_capturing()`, keeps the landing page for the visitor it is then recorded for.

The `$fbc` person property for Meta's Conversions API is derived from the `fbclid` of the page the visitor arrived on, so it is set even when the first event is sent from a page whose URL no longer carries it. A `_fbc` cookie set by the Meta pixel takes precedence, as before: one that holds the same click, or a click the pixel recorded after the visitor arrived, wins over the landing page's `fbclid`. When you pass `landing`, its `at` is the time the cookie is compared with; without it, a cookie for another click wins whatever its time. A setting changed with `set_config` before the first event is applied to the landing page's values as if `init` had run with it: masking (`mask_personal_data_properties`), `custom_campaign_params`, and `save_campaign_params` or `save_referrer` turned off, which removes what was recorded for it. While capturing is off (`opt_out_capturing()`), a change removes what it must but stores nothing new; the landing page is recorded under the settings in force once capturing is back.

Tedooo Panel tracking links add two parameters to the URL they redirect to, `tp_link` and `tp_click`. The SDK captures both as event properties under those names, with nothing to configure. If you pass your own `custom_campaign_params`, those are captured as well.

If your app loads the SDK late, for example in a lazily loaded bundle of a single-page app, the URL may have changed by the time `init` runs. Record the landing page yourself as early as you can and hand it over:

```js
// in your entry bundle, before any client-side navigation
const landing = { url: location.href, referrer: document.referrer, at: Date.now() }

// later, when the SDK has loaded
panel.init('<your project key>', { landing })
```

`url` and `referrer` may each be left out; the SDK then reads it from the document when `init` runs. `at` is when your code recorded the landing, in milliseconds since the epoch; pass it, so that a click the Meta pixel recorded before the SDK loaded is weighed against the right time.

`landing` sets the event-level properties (`utm_*`, `$referrer`, the `$search_engine` and `ph_keyword` of a search referrer, the `$initial_*` person properties). The `$session_entry_*` properties, and the `$current_url`, `utm_*` and `$referrer` the SDK adds to `$set_once` for a session, are read from the page at session start, as in posthog-js, and ignore `landing`. Tedooo Panel attributes a visit from the event-level properties of its entry event, not from `$session_entry_*`.

### What a tracking link needs

Tedooo Panel does not use the two event properties to credit a visit to a tracking link. It reads `tp_link` and `tp_click` from the page URL (`$current_url`) of the visit's first `$pageview`, or of its first event if the visit sends no pageview. That event has to be sent while the two parameters are still in the address bar.

`landing` does not change the page URL an event reports. It keeps the campaign tags and the referrer of the visit; on its own it does not keep the tracking link.

If your app rewrites or redirects the URL before that first pageview or event is sent, do one of these:

- Keep `tp_link` and `tp_click` in the URL until it has been sent.
- If you send that event yourself (`capture_pageview: false`), pass the landing URL with it. A property you pass replaces the SDK's own value:

```js
panel.capture('$pageview', { $current_url: landing.url, $pathname: new URL(landing.url).pathname })
```

If you use the `segment` option, the landing page is recorded when the first event is captured, as in 0.1.0, and `landing` has no effect.

## What the SDK requests

On `init` the SDK fetches its settings from `https://p.tedooo.com/array/<project key>/config` and then posts events to `https://p.tedooo.com/e/`.

It posts to `https://p.tedooo.com/flags/` to ask which [A/B groups](#ab-groups-and-feature-flags) the visitor is in:

- on `init`, when the settings say the project serves a test or a flag, or when the browser still holds groups from an earlier visit;
- on `identify`, `reset`, `setPersonProperties` and `group`;
- every five minutes while the tab is open and visible, less often when nobody is using the page.

The request carries the project key, the visitor ID, the device ID, whether the visitor is identified (`$is_identified`), the person properties you have set, the visitor's first landing page, referrer and campaign tags (the `$initial_*` properties) and the time zone. Pass `advanced_disable_flags: true` if you want none of that traffic; the settings are then not fetched either.

No scripts are loaded at runtime.

If your site sets a Content Security Policy, allow the capture host in `connect-src`:

```
connect-src https://p.tedooo.com
```

`script-src` needs no entry.

## A/B groups and feature flags

A/B tests and on/off flags are created in Tedooo Panel. The SDK asks the capture service which group the visitor is in, keeps the answer in storage so the next page load can read it at once, and attaches it to every event.

Groups are assignment, not access control. Anyone holding the project key, which is public, can ask for any visitor's groups and can pick the IDs it sends. Never decide a price, a paid feature or a permission by a group.

### Reading a group

```js
if (panel.getABGroup('ab_checkout') === 'target') {
    // show the new checkout
}
```

`getABGroup(key)` returns the name of the variant the visitor is in. It returns `undefined` when the project serves nothing under that key, and on a first visit until the capture service has answered.

The first read of a key sends a `$feature_flag_called` event, which records that the visitor saw the test. Pass `{ send_event: false }` to read without it. A key that returns `undefined` sends nothing.

`getABGroups()` returns every group by key and sends no event:

```js
panel.getABGroups() // { ab_checkout: 'target', new_checkout: false }
```

### On/off flags

A flag is a key whose value is `true` or `false`. It is read the same way:

```js
if (panel.getABGroup('new_checkout') === true) {
    // the flag is on for this visitor
}
```

`false` means the flag is running and is off for this visitor. `undefined` means the flag is not served (it is a draft, paused or deleted) or the groups are not known yet.

### Reacting to changes

```js
const unsubscribe = panel.onABGroups((groups, { errorsLoading }) => {
    render(groups.ab_checkout)
})
```

The callback gets the same object as `getABGroups()`. It runs at once if groups are already known, from storage or from an answer on this page, and again whenever they change: after `identify` or `reset`, when a test is started or paused, or when another tab of the same site loads new ones. An answer that changes nothing does not call it.

`errorsLoading` is `true` when the request failed. The groups are then the ones the SDK already had, and the callback runs again once a request succeeds.

Subscribe after `init`. In React, subscribe in an effect and return `unsubscribe` from it.

### Who gets which group

By default an anonymous visitor is grouped by browser, so the group survives `reset()`. An identified user is grouped by account: the first browser an account identifies on decides its groups, and the account keeps them on every other device. Call `identify` as soon as you know who the user is.

### Groups on events

Once groups are known, every event carries one property per key, named `$feature/<key>`, holding the variant name or `true` / `false`, plus `$active_feature_flags`, the list of keys that are not `false`. Use `$feature/<key>` to filter and break down reports in Tedooo Panel. Events sent on a first visit before the capture service has answered do not have them.

### Reading groups on a server

With `persistence` set to `'localStorage+cookie'` (the default) or `'cookie'`, the SDK writes a cookie named `ph_<project key>_posthog`. It is not `HttpOnly`, lasts a year, and is shared with your subdomains (pass `cross_subdomain_cookie: false` to keep it on the current host). Its value is URL-encoded JSON that includes `distinct_id`, `$device_id` and `$user_state` (`'anonymous'` or `'identified'`).

A server that renders the page can read that cookie and ask for the same groups the browser will get:

```js
const state = JSON.parse(cookieValue) // most frameworks have already URL-decoded it
const response = await fetch('https://p.tedooo.com/flags/?v=2', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({
        token: '<your project key>',
        distinct_id: state.distinct_id,
        $device_id: state.$device_id,
        $is_identified: state.$user_state === 'identified',
    }),
})
const { featureFlags, errorsWhileComputingFlags } = await response.json()
// featureFlags is { ab_checkout: 'target', new_checkout: false }
```

Treat a failed request, or an answer with `errorsWhileComputingFlags: true`, as "unknown" and let the browser decide. A visitor's first page view has no cookie yet, so there is nothing to ask with.

### posthog-js names

`getFeatureFlag`, `isFeatureEnabled`, `getFeatureFlagResult` and `onFeatureFlags` read the same groups. They differ in two ways: reading a key that is not served sends a `$feature_flag_called` event marked as a missing flag, and `onFeatureFlags` leaves out flags that are `false`. Flags have no payloads.

## Moving from posthog-js

The API is the same as posthog-js. Change the import and remove `api_host` and `ui_host` if they pointed at PostHog:

```diff
-import posthog from 'posthog-js'
+import posthog from '@tedooo/panel-js'

-posthog.init('<your project key>', { api_host: 'https://us.i.posthog.com' })
+posthog.init('<your project key>')
```

Only the main entry exists. There is no `@tedooo/panel-js/react` yet: if you used `posthog-js/react`, call `init` yourself (in React, inside a `useEffect`) and pass the `panel` object around or import it where needed.

Returning visitors keep their identity as long as the project key stays the same, because the visitor ID is stored under a name that includes the key (`ph_<project key>_posthog`). To keep your existing key, paste it when you create the project in Tedooo Panel. If you use a new key instead, pass `persistence_name: '<old project key>_posthog'` to `init` so the SDK keeps reading the old storage.

## Not available yet

Tedooo Panel does not support these yet, and their options should be left off:

- **Session replay**, **surveys**, **product tours**, **conversations** and the **toolbar** are not in this package. Their options are accepted but nothing happens.
- **Heatmaps**, **web vitals**, **dead clicks** and **exception autocapture** are off by default. Do not turn them on: the capture service does not store their events.

## License

See [LICENSE](./LICENSE). This package is a modified version of [posthog-js](https://github.com/PostHog/posthog-js) and includes its Apache-2.0 and MIT license texts.
