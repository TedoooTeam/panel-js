# Changelog

## 0.2.0

A/B groups and feature flags.

- New `getABGroup(key)`, `getABGroups()` and `onABGroups(callback)` read the groups the capture service assigns to the visitor. `getABGroup` returns the variant name of an A/B test or `true` / `false` for an on/off flag, and sends no event for a key the project does not serve. `onABGroups` reports every group, including flags that are off, when they change and at once when they are already known.
- Every event carries `$feature/<key>` for each group once groups are known.
- The `/flags/` request carries one new field, `$is_identified`, so the capture service can keep an account in the same groups on every device.
- `identify()` asks for groups again when the current ID becomes identified without changing.
- On `init` the SDK also asks for groups when the browser holds some from an earlier visit, even if the settings report none, so pausing a project's last test clears them.
- The README covers A/B groups, including how a server can read the SDK's cookie to render the same groups. Groups are assignment, not access control.

Campaigns and tracking links.

- The landing page is recorded when `init` runs, not at the first event. If the URL changes before the first event is sent, the events of the visit still carry the campaign parameters, the referrer and the `$initial_*` person properties of the page the visitor arrived on. When capturing is off at `init` (`opt_out_capturing_by_default`, `cookieless_mode: 'on_reject'`), the same page is recorded once capturing starts, at `opt_in_capturing()` or at `opt_out_capturing()` in `on_reject` mode. `reset()` discards it with the rest of the visitor's state, unless nothing was recorded for the visitor yet; the next visitor is recorded from the page they are on, without its campaign parameters if an event was already sent from that URL, as posthog-js reads the parameters of a URL once.
- New `landing` option: pass `{ url, referrer, at }` recorded by your own code at page load, for apps that load the SDK after the page may have navigated. `at` is when you recorded it, in milliseconds since the epoch; `url` and `referrer` left out are read from the document when `init` runs. A setting changed with `set_config` before the first event (`save_campaign_params`, `save_referrer`, `mask_personal_data_properties`, `custom_personal_data_properties`, `custom_campaign_params`) is applied to what was recorded, as if `init` had run with it; one turned off removes what was recorded for it, and while capturing is off nothing new is stored until it is back.
- `$fbc` is derived from the `fbclid` of the landing page when the URL at the first event no longer carries it. A `_fbc` cookie from the Meta pixel takes precedence, as before, when it holds the same click or one recorded after the landing: after `landing.at`, or after `init` when the SDK read the URL itself. With a `landing.url` and no `at`, a cookie for another click takes precedence whatever its time.
- Tedooo tracking-link parameters `tp_link` and `tp_click` are captured as event properties by default, like `utm_*`. Your own `custom_campaign_params` are kept. Tedooo Panel credits a visit to a link from the page URL of the visit's first pageview or event, not from these properties: see "What a tracking link needs" in the README.
- `$initial_ph_keyword` is read from the stored initial referrer; posthog-js reads the keyword from the live referrer, which differs from the initial one on later page loads.
- With the `segment` option the landing page is still recorded at the first event, as in 0.1.0.

Based on posthog-js 1.435.6, as before.

## 0.1.0

First release.

- Events are sent to `https://p.tedooo.com` by default, so no `api_host` and no reverse proxy are needed.
- Ships the core SDK: `init`, `capture`, `identify`, `setPersonProperties`, `group`, `alias`, `reset`, `register`, pageview and pageleave capture, and autocapture.
- No scripts are loaded at runtime. The only requests are to the capture host: the SDK's settings on `init`, then events.
- Based on posthog-js 1.435.6. The API of the main entry is the same, so an existing posthog-js integration only needs its import changed. Subpaths such as `posthog-js/react` have no equivalent yet.
