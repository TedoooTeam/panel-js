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

| Option              | Default                  | What it does                                                                                                             |
| ------------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| `api_host`          | `'https://p.tedooo.com'` | Where events are sent. Set it only if you route traffic through your own domain.                                         |
| `autocapture`       | `true`                   | Capture clicks, form submissions and other interactions automatically.                                                   |
| `capture_pageview`  | `true`                   | Send a `$pageview` event on load. Pass `'history_change'` for single-page apps.                                          |
| `capture_pageleave` | `'if_capture_pageview'`  | Send a `$pageleave` event when the visitor leaves.                                                                       |
| `person_profiles`   | `'identified_only'`      | Which visitors get a person profile: `'always'`, `'identified_only'` or `'never'`.                                       |
| `persistence`       | `'localStorage+cookie'`  | Where the visitor ID is stored: `'localStorage+cookie'`, `'localStorage'`, `'cookie'`, `'sessionStorage'` or `'memory'`. |
| `loaded`            | none                     | Callback that runs once the SDK is ready.                                                                                |

All options are typed; your editor lists the rest.

## What the SDK requests

On `init` the SDK fetches its settings from `https://p.tedooo.com/array/<project key>/config` and then posts events to `https://p.tedooo.com/e/`. It also posts to `/flags/` on `init` and `identify`; pass `advanced_disable_flags: true` if you want none of that traffic. No scripts are loaded at runtime.

If your site sets a Content Security Policy, allow the capture host in `connect-src`:

```
connect-src https://p.tedooo.com
```

`script-src` needs no entry.

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
- **Feature flags**: `isFeatureEnabled`, `getFeatureFlag` and friends work but every flag is undefined, and each check sends a `$feature_flag_called` event. Leave flag options off.
- **Heatmaps**, **web vitals**, **dead clicks** and **exception autocapture** are off by default. Do not turn them on: the capture service does not store their events.

## License

See [LICENSE](./LICENSE). This package is a modified version of [posthog-js](https://github.com/PostHog/posthog-js) and includes its Apache-2.0 and MIT license texts.
