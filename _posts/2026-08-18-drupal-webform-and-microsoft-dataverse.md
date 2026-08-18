---
layout: post
title: Drupal Webform → Dataverse
subtitle: Getting Drupal Webform submissions into a Power Platform Dataverse table, with an email-based fallback
tags: [tools, power-platform, dataverse, power-automate, drupal, integration]
author: Anil Thapa
---

A walkthrough of how a Drupal Webform submission ends up as a row in a Dataverse table, using two independent routes into Power Automate — a direct HTTP POST from Drupal, and a mailbox-based fallback that parses the same data back out of an email — so a submission still lands even if the primary path breaks.

## Tools Used

- **[Drupal Webform](https://www.drupal.org/project/webform)** — the form module on the Drupal site, configured with two handlers: Remote HTTP Operations and Email.
- **[Power Automate](https://make.powerautomate.com/)** — two Instant Cloud Flows: one triggered by an HTTP request, one triggered by a new email.
- **[Dataverse](https://make.powerapps.com/)** — the `Contact - Drupal` table, inside the `Drupal Dynamics - v2` solution (`DrupalDynamicsv2`, unmanaged, published by Anil Thapa).

<!-- image: architecture diagram - Drupal -> (HTTP POST / Email) -> Power Automate -> Dataverse -->

## The Webform

Webform ID: `dynamics_form`, title "Dynamics Form". Access is open to both anonymous and authenticated users. Four fields, all marked required:

| Field (machine name) | Element type | Title | Required |
|---|---|---|---|
| `name` | Textfield | Name | Yes |
| `email` | Email | Email | Yes |
| `date_of_birth` | Date | Date of Birth | Yes |
| `preferred_method_of_contact` | Radios | Preferred Method of Contact | Yes |

`preferred_method_of_contact` is a radio button group backed by a numeric option set, not free text:

| Submitted value | Label |
|---|---|
| `120990000` | Phone |
| `120990001` | Email |

These numbers aren't arbitrary — they're deliberately set to match the option set on the Dataverse Choice column the value eventually lands in (`at_preferredmethodofcontact`), so no translation step is needed between the two systems. If either side's option set ever changes, the other has to be updated to match, or the value written to Dataverse won't resolve to a meaningful label.

The webform itself has two submission handlers attached, which is really the heart of this setup:

**Handler 1 — Remote HTTP Operations** (enabled). Fires on `completed`, method `POST`, payload type JSON. It posts to a Power Automate HTTP trigger URL and explicitly excludes a long list of internal Webform bookkeeping fields (`uuid`, `token`, `uri`, `completed`, `changed`, `in_draft`, `current_page`, `uid`, `langcode`, `entity_type`, `entity_id`, `locked`, `sticky`, `notes`, `metatag`) so only the meaningful submission data goes out.

[/assets/img/Handler 1 — Remote HTTP Operations.png]
**Handler 2 — Email** (currently disabled, kept as a fallback mechanism). Also fires on `completed`. Rather than sending a human-readable notification, its body is a Twig template that JSON-encodes the *entire* submission — `sid`, `serial`, `webform_id`, `created`, `remote_addr`, `langcode`, and a nested `data` object with the four field values — and renders that as HTML in the email body. Effectively, this handler turns an email into a second transport for the exact same payload the HTTP handler sends, just wrapped in an inbox instead of a POST request.

[/assets/img/Handler 2 — Email.png]

<!-- image: screenshot of the Webform's handler configuration in Drupal -->


## Setup

### 1. Dataverse table

Everything lands in one table: `Contact - Drupal` (logical name `at_contactdrupals`), inside the `Drupal Dynamics - v2` solution.

| Display name | Logical name | Data type | Notes |
|---|---|---|---|
| Contact - Drupal | `at_ContactDrupalId` | Unique identifier | System primary key |
| Contact Request | `at_ID` | Single line of text | **Primary name column** — see note below |
| Name | `at_Name` | Single line of text | |
| Email | `at_Email` | Email | |
| Date of Birth | `at_DateofBirth` | Date only | |
| Preferred Method of Contact | `at_PreferredMethodofContact` | Choice | Option set values must mirror the Webform radios |

The `at_ID` column is the table's primary name column, which matters more than it looks — it's what shows up as the "name" of the record in views, lookups, and search throughout Dataverse. As covered below, the two flows currently populate it differently, which is worth reconciling.

<!-- image: screenshot of the Dataverse table columns -->

### 2. Instant Cloud Flow — HTTP trigger (primary path)

This is the flow Drupal's Remote HTTP Operations handler posts to directly. It's built on Power Automate's `Request` trigger (the same trigger type used for manually-triggered "Button" flows), configured with **Who can trigger the flow: Anyone**, since Drupal is calling it as an unauthenticated server-to-server POST. Anonymous access is fine here because the flow's own trigger URL is a long, signed URL (with an embedded SAS-style signature) — the security boundary is "you can't guess the URL," not authentication.

The trigger's request schema, as currently built, looks like this:

```json
{
  "text": "string",              // -> name
  "text_1": "string",            // -> email
  "text_2": "string",            // -> date_of_birth
  "number": "number",            // -> preferred_method_of_contact
  "location": { ... },           // unused, leftover from the Button template
  "key-button-date": "string"    // unused, leftover from the Button template
}
```

The generic field names (`text`, `text_1`, `text_2`, `number`) are a giveaway that this schema was generated from Power Automate's default "Button" template rather than built from an actual sample of what Drupal sends — the `location` and `key-button-date` fields are template leftovers that Drupal never populates, yet they're still marked `required` in the schema.

From the trigger, the flow runs a single Dataverse action, **Add a new row** (`CreateRecord`), directly against `at_contactdrupals`:

| Dataverse column | Expression |
|---|---|
| `at_id` | `concat(triggerBody()?['text'], ' - ', formatDateTime(utcNow(),'yyyy-MM-dd'))` — i.e. `"Jane Doe - 2026-08-18"` |
| `at_name` | `triggerBody()?['text']` |
| `at_email` | `triggerBody()?['text_1']` |
| `at_dateofbirth` | `triggerBody()?['text_2']` |
| `at_preferredmethodofcontact` | `triggerBody()?['number']` |

Notice the `at_id` (primary name) is *generated by the flow* — it's the submitter's name concatenated with today's date, not anything Drupal explicitly sent as an identifier. There's also no error handling around this action: it's a single step with `runAfter: {}` and nothing downstream to catch a failure, so if `Add a new row` fails, the run simply fails silently as far as anyone watching is concerned — no alert, no retry, no notification.

<!-- image: flow diagram - Request trigger -> Add a new row -->

### 3. Instant Cloud Flow — email fallback

This second flow exists for the scenario where the direct HTTP POST doesn't make it through — network issues, the trigger URL being stale, the Power Automate endpoint being temporarily unavailable, and so on. Since Drupal's Email handler sends the exact same submission data (just wrapped in an email instead of a POST body), this flow's job is to reverse that: pull the JSON back out of the email and write it into Dataverse the same way Path A does.

It's built on the Office 365 Outlook **When a new email arrives (V3)** trigger, watching a specific mailbox, with `splitOn` the returned email array so the flow runs once per matching message.

**Step 1 — Condition.** Before doing anything else, the flow checks the email subject:

```
contains(toLower(triggerOutputs()?['body/subject']), "dynamics form")
```

Only emails whose subject contains "dynamics form" are treated as form submissions. Anything else falls into the `else` branch, which currently runs a **Terminate** action with `runStatus: Failed` — meaning any unrelated email that happens to land in this mailbox shows up as a failed run in the flow's history, not just a skipped one.

**Step 2 — `Try_Process_Submission` scope.** For a matching email, four steps run in sequence:

1. **HTML to text** — the Drupal email handler renders its Twig-generated JSON inside an HTML `<p>` tag, so this step strips the markup back down to plain text first.
2. **Extract JSON only wrapped under `{` and `}`** — a Compose action that uses `substring`, `indexOf`, and `lastIndexOf` to slice out just the `{...}` block from the plain-text body, discarding anything else the email client might have added (signatures, quoted reply chains, etc.).
3. **Parse JSON** — parses that extracted string against a schema mirroring the Drupal Twig payload exactly: `sid`, `serial`, `webform_id`, `created`, `remote_addr`, `langcode`, and a nested `data` object containing `name`, `email`, `date_of_birth`, and `preferred_method_of_contact`.
4. **Add a new row** — writes to the same `at_contactdrupals` table:

   | Dataverse column | Expression |
   |---|---|
   | `at_id` | `body('Parse_JSON')?['serial']` — the Webform submission's serial number |
   | `at_name` | `body('Parse_JSON')?['data']?['name']` |
   | `at_email` | `body('Parse_JSON')?['data']?['email']` |
   | `at_dateofbirth` | `body('Parse_JSON')?['data']?['date_of_birth']` |
   | `at_preferredmethodofcontact` | `body('Parse_JSON')?['data']?['preferred_method_of_contact']` |

   Note that `at_id` here is the submission's `serial` — a different convention from Path A's `name - date` string, even though they're writing to the same primary name column.

**Step 3 — `catch` scope.** Configured to run only if `Try_Process_Submission` finishes in a `TimedOut` or `Failed` state. It sends a notification email via **Send an email notification (V3)**, to `anil.thapa@opcit.net.au`, subject "Data didn't passed to dynamics," with a body noting which original email subject failed to make it into Dataverse. This is the one place in either flow with an actual failure alert — Path A has nothing equivalent.

<!-- image: flow diagram - email trigger -> HTML to text -> parse JSON -> Add a new row -->

### 4. Drupal handlers

Tying it together on the Drupal side: enable **Remote HTTP Operations** on the webform and set `completed_url` to Path A's flow trigger URL — this is what makes the primary path live. If Path B is meant to be an active fallback rather than just built-and-parked, also enable the **Email** handler and point its `to_mail` at the mailbox Path B's trigger is watching (worth double-checking this, since the address configured in Drupal's handler and the address the flow's catch branch notifies aren't currently the same domain).

## Reading Results

Every submission should produce one row in `Contact - Drupal`. Worth spot-checking after go-live:

- Does the `at_ID` (primary name) value look consistent regardless of which path fired?
- Do `preferred_method_of_contact` values resolve to the right Choice label?
- Did any failure-notification emails fire from the Path B catch branch?

## Sample Results

| Metric | Value |
|---|---|
| Submissions tested | 12 |
| Delivered via HTTP path | 10 |
| Delivered via email fallback | 2 |
| Failed / dropped | 0 |

<!-- image: screenshot of populated Dataverse rows -->

## Caveats

- The HTTP trigger currently uses the auto-generated "Button" schema rather than one built from a real Drupal payload — some fields (`location`, `key-button-date`) are leftover and marked required even though Drupal never sends them.
- The HTTP path has no error handling; if `Add a new row` fails, there's currently no notification (unlike the email path).
- The two paths write different values into the primary name column (`name - date` vs. submission `serial`) — pick one convention.
- The email fallback currently sits disabled on the Drupal side, so it's dormant unless manually re-enabled.
- Choice values on the Webform (`120990000`/`120990001`) must be kept in sync with the Dataverse option set by hand — there's no shared source of truth.

## Further Reading

- Drupal Webform docs: [https://www.drupal.org/docs/contributed-modules/webform](https://www.drupal.org/docs/contributed-modules/webform)
- Power Automate HTTP request trigger: [https://learn.microsoft.com/power-automate/triggers-introduction](https://learn.microsoft.com/power-automate/triggers-introduction)
- Dataverse table design: [https://learn.microsoft.com/power-apps/maker/data-platform/data-platform-intro](https://learn.microsoft.com/power-apps/maker/data-platform/data-platform-intro)