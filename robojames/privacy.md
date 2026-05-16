---
layout: page
title: RoboJames Privacy Policy
sitemap: false
---

_Last updated: 16 May 2026_

RoboJames is a private, self-hosted personal assistant operated by James Baker
("the operator") for use by **two specified individuals**: the operator and one
other member of his household ("the users"). It is not a public service and is
not made available to anyone outside that two-person household.

This policy describes what data RoboJames handles, how it handles it, and how
the users can control it.

## 1. Who this policy is for

This policy applies only to the two named individuals on the operator's
allowlist. RoboJames does not accept connections from anyone else; there is no
sign-up flow.

## 2. What data is collected

When a user interacts with RoboJames, the following data is processed:

| Source | Data | Reason |
|---|---|---|
| Signal | The text of messages the user sends to the bot, and the bot's replies | To carry on the conversation and to provide history for the assistant |
| Signal | The user's Signal identifier (phone number, username, or UUID) | To route replies back to the correct user |
| Google (after the user links their account) | Email address and basic profile information returned by Google's `userinfo` endpoint | To label which Google account is linked |
| Google (after consent) | Google Calendar events, calendar list, and free/busy data | To answer questions about the user's schedule and to create or update events on their request |
| Anthropic Claude API | The above conversation context, sent on each turn so the model can generate a reply | To produce the assistant's responses |
| TfL / Google Maps | Origin and destination strings the user supplies for journey planning | To return a route |

RoboJames does **not** collect device identifiers, location beyond what the
user supplies in a message, contact lists, photos, files, or any data outside
the categories listed above.

## 3. Google user data: scopes and purposes

When a user voluntarily links a Google account, RoboJames requests the
following OAuth scopes, and uses each only for the purpose listed below:

- **`https://www.googleapis.com/auth/calendar.events`** — to read the user's
  upcoming calendar events when they ask about their schedule, and to create
  or modify events when they explicitly ask for one to be added or changed.
- **`https://www.googleapis.com/auth/calendar.calendarlist.readonly`** — to
  enumerate which calendars the user has, so the assistant can target the
  right one (e.g. "work" vs "family").
- **`https://www.googleapis.com/auth/calendar.freebusy`** — to check the
  user's availability when suggesting meeting times.

Google user data is used **only** to provide the assistant features the user
requests in their Signal conversation. It is not shared with any third party
beyond what is described in section 5, is not used for advertising, is not
sold, and is not used to train or improve any machine-learning model.

## 4. Where data is stored

- Conversation history, the user allowlist, and OAuth refresh tokens are stored
  in a PostgreSQL database running on a single home server controlled by the
  operator. The server is not publicly addressable except through a reverse
  proxy that only exposes the OAuth callback endpoint required for the Google
  link flow.
- Backups, if any, are stored locally on the same household network.
- No data is replicated to any cloud storage provider.

## 5. Third-party processing

RoboJames sends data to two third-party services strictly for the purpose of
providing its functionality:

- **Anthropic** (the operator of the Claude API) receives the conversation
  context on each turn so it can generate the assistant's reply. Anthropic
  acts as a processor; the operator's account is bound by Anthropic's
  commercial terms, which prohibit Anthropic from using API inputs to train
  their models by default.
- **Google** receives requests against the Calendar APIs that the user has
  consented to. These are authorised by the user's own OAuth grant.
- **Transport for London** receives origin and destination strings when the
  user asks for a journey plan.
- **Google Maps Routes API**, when the operator has enabled the optional
  fallback, receives origin and destination strings when TfL cannot plan a
  route.

No data is shared with advertisers, analytics providers, or data brokers.

## 6. Retention and deletion

- Conversation history is retained indefinitely unless the user asks for it
  to be deleted.
- Google OAuth tokens are retained until the user unlinks their account,
  either through the bot or by revoking access at
  <https://myaccount.google.com/permissions>.
- Either user may request deletion of any or all of their data at any time
  by emailing the operator (see section 8). Deletion requests will be
  actioned within seven days.

## 7. Security

- The database is accessible only to the operator's local user account on the
  home server.
- OAuth refresh tokens are protected by filesystem permissions.
- The Signal bot's allowlist is checked on every inbound message; messages
  from anyone who is not one of the two authorised users are discarded
  without being processed.
- RoboJames is open source; the code can be inspected at the operator's
  request.

## 8. Contact

For any question about this policy, or to request deletion of stored data,
contact the operator:

**James Baker** — <j.baker@outlook.com>

## 9. Changes

Because RoboJames is used only by two individuals, changes to this policy
will be communicated directly between them. The version date at the top of
this document will be updated when changes are made.
