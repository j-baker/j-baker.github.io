---
layout: bare
title: RoboJames Privacy Policy
sitemap: false
---

_Last updated: 16 May 2026_

RoboJames is a private, self-hosted personal assistant operated by James Baker
("the operator") for use by two specified individuals. It is not a public
service.

## Data collected

| Data | Purpose |
|---|---|
| Signal messages and replies | Conversation history |
| Signal identifier | Routing replies |
| Google profile (after linking) | Labelling the linked account |
| Google Calendar data (after consent) | Answering schedule queries and creating events |
| Conversation context sent to Anthropic Claude API | Generating responses |
| Journey origin/destination sent to TfL / Google Maps | Route planning |

## Google user data

When a user links a Google account, RoboJames requests the following scopes:

- **`calendar.events`** — read, create, and modify calendar events.
- **`calendar.calendarlist.readonly`** — list available calendars.
- **`calendar.freebusy`** — check availability.

Google user data is used only to provide assistant features requested in
conversation. It is not shared with third parties beyond what is listed below,
not used for advertising, not sold, and not used to train any model.

## Third-party processing

- **Anthropic** receives conversation context to generate replies.
- **Google** receives Calendar API requests authorised by the user's OAuth grant.
- **TfL / Google Maps** receives origin and destination for journey planning.

No data is shared with advertisers, analytics providers, or data brokers.

## Retention and deletion

- Conversation history is retained until the user requests deletion.
- Google OAuth tokens are retained until the user unlinks their account or
  revokes access at <https://myaccount.google.com/permissions>.
- Deletion requests will be actioned within seven days.

## Contact

**James Baker** — <j.baker@outlook.com>
