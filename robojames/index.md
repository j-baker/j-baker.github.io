---
layout: page
title: RoboJames
sitemap: false
---

<img src="logo.svg" alt="RoboJames logo" width="96" height="96">

**A personal Signal assistant for a household of two.**

RoboJames is a private, self-hosted assistant that lives in Signal. It runs on a
home server operated by James Baker and is used only by James and one other
member of his household. It is not offered as a service to the public; there is
no sign-up page, no marketing site, no API, and no plan to add other users.

## What it does

- Replies to Signal messages from its two authorised users.
- Looks up and creates Google Calendar events on request (when the user has
  linked their Google account through the consent flow).
- Plans public-transport journeys around London (TfL) and, when needed, via the
  Google Maps Routes API.
- Schedules reminders that fire as Signal messages at a later time.

## Who runs it

- **Operator and primary user**: James Baker — <j.baker@outlook.com>
- **Other authorised user**: one further member of the operator's household.
- **Total users**: two. The user allowlist is hard-coded in the deployment
  configuration; new users cannot self-register.

## How to control your data

- See the [Privacy Policy](privacy/) for a full description of what is stored.
- To revoke RoboJames' access to your Google account at any time, visit
  <https://myaccount.google.com/permissions> and remove the "RoboJames" app.
- To request deletion of your stored data, email
  [j.baker@outlook.com](mailto:j.baker@outlook.com).

## Legal

- [Privacy Policy](privacy/)
- [Terms of Use](terms/)
