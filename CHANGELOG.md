# Changelog

## 0.1.0

First release.

- Events are sent to `https://p.tedooo.com` by default, so no `api_host` and no reverse proxy are needed.
- Ships the core SDK: `init`, `capture`, `identify`, `setPersonProperties`, `group`, `alias`, `reset`, `register`, pageview and pageleave capture, and autocapture.
- No scripts are loaded at runtime. The only requests are to the capture host: the SDK's settings on `init`, then events.
- Based on posthog-js 1.435.6. The API of the main entry is the same, so an existing posthog-js integration only needs its import changed. Subpaths such as `posthog-js/react` have no equivalent yet.
