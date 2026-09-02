# AJSU-Tasker

A single-file biweekly work log for tracking daily activities across pay periods. It was built for El Paso County Budget Department and is shared here for anyone who wants to adapt it.

**Try it live:** https://ajharrisgit.github.io/tasker/

## What it does

- Logs daily activity blocks across two work weeks per pay period
- Generates a printable Tasker Memo for supervisor sign-off
- Tracks productivity metrics with at-a-glance summary views
- Builds cross-period trend charts automatically
- Exports/imports your data as a JSON backup file
- Exports activity logs as CSV

## Getting your own copy

This is a single HTML file. There is no install, no build step, no server, and no account.

1. Download `AJSU-Tasker-v6.html` from this repo (or clone the whole repo).
2. Open it directly in **Chrome or Edge**. Firefox is not supported. See below.
3. Start typing. Your data saves automatically to that browser on that device.

Your data lives entirely in your own browser's local storage. Nothing is sent
to a server. Nothing you enter here is visible to anyone else, including
other people who open this same app from the hosted link above in their own
browser.

## Adapting it for your own pay calendar

The pay-period dates are a plain array near the top of the script. See
[`PayPeriodCalendar-Template.json`](PayPeriodCalendar-Template.json) for a
sanitized, standalone copy of that structure with notes on how to swap in
your own calendar safely. There is a rollover trap around year boundaries, so
read the notes before you edit.

## Browser support

**Chrome or Edge only.** Firefox's handling of locally-opened (`file://`)
files does not reliably persist data between sessions. This is a known
limitation of Firefox itself, not a bug in this app, and it caused a real
data-loss incident during development. Use Chrome or Edge.

## License

Creative Commons Attribution-NonCommercial 4.0 (CC BY-NC 4.0). See
[`LICENSE`](../LICENSE). Free to use, share, and adapt with credit. Not for
commercial use.

## More detail

`AJSU-Tasker_UserGuide.html` is the end-user guide. It opens in a browser like the app itself.
