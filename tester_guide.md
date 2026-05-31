# What This Tool Does — A Plain-English Guide for Testers

> **Who this is for:** You already know GoHighLevel (sub-accounts, contacts, calendars,
> pipelines, workflows). You do **not** need to know any code. This guide explains, in plain
> language, what this app lets you and your AI do — and how to get going.

---

## The 30-second version

Normally, getting a website or an AI agent to "talk to" your GoHighLevel account is a technical
nightmare — API keys, tokens, permissions, developer help. **This app removes all of that.**

Once it's installed on a sub-account, your AI agent (in AI Agent Studio / Simple GHL AI) or any
website can instantly **read and write everything in that GHL account** — create contacts, book
appointments, send messages, manage pipelines, invoices, and more — just by asking.

**You install it once. After that, it just works.**

---

## What you can actually do with it

Think of it as a universal remote control for a GoHighLevel sub-account. Here's what's on it:

| You want to… | The app can… |
|--------------|--------------|
| **Capture leads** | Create and update contacts from any website form |
| **Look people up** | Search and pull contact details |
| **Book appointments** | Show available time slots and book calendar events |
| **Message people** | Send SMS and emails, read conversation history |
| **Manage deals** | Create and move opportunities through pipelines |
| **Tag & organize** | Add and manage tags and custom fields |
| **Handle money** | Create invoices and estimates, manage coupons, view orders & transactions |
| **Custom data** | Store and retrieve your own custom objects (e.g. properties, vehicles, policies) |
| **Files** | Upload and manage media/files |
| **Contracts** | Create and look up contracts |
| **AI voice & content** | Work with Voice AI and AI-generated content |

In short: **almost anything you do by clicking around inside GoHighLevel, your AI or website can now do automatically.**

---

## How it works (no jargon)

1. You **install the app** on a GoHighLevel sub-account (one time — see below).
2. The app quietly stores a secure connection to that account. **You never see or handle any keys.**
3. From then on, your AI agent or website sends a simple "request" to one web address, saying
   things like *"create this contact"* or *"book this appointment"*, and the app does it inside
   GoHighLevel for you.

That's the whole thing. The complexity is hidden on purpose.

---

## Getting started — step by step

### Step 1 — Install the app on a sub-account
- Open the app's install link (provided to you).
- Choose the GoHighLevel location/sub-account you want to connect.
- Approve the permissions.
- Done. You'll see a success message. The connection is now live for that account.

> If you're an agency, installing at the agency level connects your accessible sub-accounts
> automatically. Any account it couldn't connect is simply skipped — no errors to worry about.

### Step 2 — Tell your AI agent what you want
This is the fun part. You don't write code — you describe what you want in plain language to your
AI agent, and it builds it. For example:

- *"Build me a landing page with a form that creates a new contact in GoHighLevel when someone signs up."*
- *"Make a booking page that shows my calendar's open slots and lets people book a call."*
- *"Create a page where I can search my contacts by name."*
- *"Add a button that sends a welcome text message to a new lead."*

The AI already has a full instruction sheet for this app (the technical companion to this guide),
so it knows exactly how to connect everything.

### Step 3 — Use the one piece of info the AI needs
When the AI asks, the only thing you need to give it is the **Location ID** of the sub-account.
In GoHighLevel pages and workflows this is written as `{{location.id}}` — the AI knows how to use that.

### Step 4 — Test it
Try your new page or AI flow. Submit a test form, book a test appointment, send a test message.
Then check inside GoHighLevel — the contact, appointment, or message will be right there.

---

## What to expect when testing

- **It's instant.** Actions happen in real time inside the connected GoHighLevel account.
- **It's the real account.** Anything you create (contacts, appointments, messages) is real — use
  test data while you're experimenting.
- **Deleting is permanent.** If you delete something, it's gone from GoHighLevel too. Your AI is
  set up to ask for confirmation before deleting — but be careful with test vs. live data.

---

## If something doesn't work

Don't worry about the technical details — but these are the usual culprits, in plain terms:

| What you see | What it usually means | What to do |
|--------------|-----------------------|------------|
| "Missing locationId" | The page didn't know which account to use | Make sure the Location ID / `{{location.id}}` is filled in |
| "Location not found" | The app isn't installed on that account (yet) | Re-run the install on that sub-account |
| "Unknown action" | The AI asked for something the app doesn't do | Ask the AI to use a supported feature (see the list above) |
| Nothing happens / error on a form | A required field was empty | Make sure required fields (like email/phone for a contact) are filled |

If you're stuck, send whoever set this up: (1) what you were trying to do, and (2) the exact error
message you saw. That's enough for them to fix it fast.

---

## The big picture

You're testing something that turns "I wish my website could just do that in GoHighLevel" into
"done." You know GoHighLevel better than anyone — your job is to dream up the useful flows
(lead forms, booking pages, follow-up messages, deal tracking) and let the AI build them on top of
this connection.

**Install once, then just describe what you want. That's it.**
