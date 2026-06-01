# AI Front-End Integration Guide — GHL API Bridge

> **Audience:** This document is written for an AI website/app builder (AI Agent Studio,
> Simple GoHighLevel AI, or any code-generating assistant). It contains everything the AI
> needs to generate working websites, landing pages, and UI/UX that talk to this API.
>
> **You do not need a GoHighLevel API key.** This bridge already holds the OAuth tokens for
> every installed location. You only need the location's `locationId` and the endpoint URL.

---

## 1. The one thing to remember

Every single feature is reached through **one endpoint**, with **one POST request**, in **one envelope shape**:

```
POST https://estimatr.tools/ghl_api/controller.php
Content-Type: application/json
```

```json
{
  "actionType": "resource.action",
  "customData": {
    "type": "resource.action",
    "locationId": "<LOCATION_ID>",
    "...": "fields, filters, ids, or a payload object"
  }
}
```

**Rules that never change:**

1. `actionType` and `customData.type` must be **identical**.
2. `customData.locationId` is **always required**. In GHL pages/workflows this is `{{location.id}}`.
3. The action format is either `resource.action` (e.g. `contacts.list`) or `resource.subresource.action` (e.g. `payments.coupons.create`).
4. The server replies with `{ "success": true, "action": "...", "data": { ... } }` on success, or an HTTP 4xx/5xx with `{ "error": "message" }` on failure.

---

## 2. Drop-in helper (always generate this once per page)

```js
// Single helper used for every call to the bridge.
async function callGhl(customData) {
  const ENDPOINT = 'https://<your-domain>/ghl_api/controller.php';

  const res = await fetch(ENDPOINT, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      actionType: customData.type,   // must match customData.type
      customData
    })
  });

  const text = await res.text();
  let json;
  try { json = JSON.parse(text); } catch { json = { raw: text }; }

  if (!res.ok) {
    throw new Error(json.error || json.message || `HTTP ${res.status}`);
  }
  return json;          // -> { success, action, data }
}
```

CORS is open for browser calls, so this works directly from any website front-end. Server-to-server
(non-browser) calls instead require the location's webhook secret — see §7.

---

## 3. How to shape `customData` per action verb

| Verb | What to send in `customData` | Example keys |
|------|------------------------------|--------------|
| `list` / `search` | Filters as flat fields (or a `params` object) | `query`, `limit`, `page`, `startAfter` |
| `get` | The record id | `id` (preferred) — legacy `contactId`, `invoiceId`, etc. also work |
| `create` / `post` | The record fields, either flat or inside `payload` | `firstName`, `email`, ... or `payload: {...}` |
| `update` / `put` | The id **plus** the fields to change | `id` + flat fields or `payload: {...}` |
| `delete` | The record id | `id` |

**Flat vs `payload`:** Both are accepted. If both are present, `payload` wins. For AI-generated
front-ends, prefer a `payload: { ... }` object for create/update — it's the clearest and least
error-prone. Flat fields exist mainly for GHL workflow webhook custom values.

**IDs:** Always send `id`. The bridge also accepts resource-specific keys (`contactId`, `userId`,
`customFieldId`, `customValueId`, `conversationId`, `couponId`, `invoiceId`, `estimateId`,
`orderId`, `transactionId`, `calendarId`, `opportunityId`, `paymentId`).

---

## 4. Complete action catalog

> The AI must **not invent actions**. Only the actions below exist. If a user needs something
> outside this list, say so and offer the nearest valid action.

### Contacts — `contacts.*`
- `contacts.list` — filters: `query`, `limit`, `page`
- `contacts.search` — filters: `query`, `limit`
- `contacts.get` — `id`
- `contacts.create` — payload: `firstName`, `lastName`, `email`, `phone`, `tags`, `customFields`
- `contacts.update` — `id` + payload
- `contacts.delete` — `id`
- **Notes** (require `contactId`):
  - `contacts.notes.list` — `contactId`
  - `contacts.notes.get` — `contactId` + `noteId`
  - `contacts.notes.create` — `contactId` + `body` (and optional `userId`)
  - `contacts.notes.update` — `contactId` + `noteId` + `body`
  - `contacts.notes.delete` — `contactId` + `noteId`
- **Tasks** (require `contactId`):
  - `contacts.tasks.list` — `contactId`
  - `contacts.tasks.get` — `contactId` + `taskId`
  - `contacts.tasks.create` — `contactId` + `title`, `body`, `dueDate` (ISO), optional `assignedTo`, `completed`
  - `contacts.tasks.update` — `contactId` + `taskId` + fields
  - `contacts.tasks.complete` — `contactId` + `taskId` + `completed` (true/false)
  - `contacts.tasks.delete` — `contactId` + `taskId`

### Users — `users.*`
- `users.list`, `users.get` (`id`), `users.create`, `users.update` (`id`), `users.delete` (`id`)

### Conversations & Messaging — `conversations.*`
- `conversations.list` / `conversations.search` — filters: `query`, `contactId`, `limit`
- `conversations.get` — `id`
- `conversations.create` — payload: `contactId`, `locationId`
- `conversations.update` — `id` + payload
- `conversations.delete` — `id`
- `conversations.markread` — `id` (the conversationId)
- **Messages:**
  - `conversations.messages.list` — `conversationId` (+ optional `params`)
  - `conversations.messages.get` — `id` (messageId)
  - `conversations.messages.send` — payload: `type` (e.g. `SMS`), `contactId`, `message`
  - `conversations.messages.email` — payload: `contactId`, `subject`, `html`/`message`
  - `conversations.messages.update` — `id` + payload
  - `conversations.messages.delete` — `id`
  - `conversations.messages.cancel` — `id` (cancel a scheduled message)
- **Import (bulk history):** `conversations.import` — send contact fields
  (`contactName` or `contactFirstName`/`contactLastName`, `contactEmail`, `contactPhone`) plus
  `textsBlob` and/or `callsBlob`.

### Calendars & Appointments — `calendars.*`
- `calendars.list`, `calendars.get` (`id`), `calendars.create`, `calendars.update` (`id`), `calendars.delete` (`id`)
- `calendars.availability.get` — `calendarId`, `startDate`, `endDate`, `timezone`, `duration`, optional `userId`
- `calendars.services.list`, `calendars.services.get` (`id`)
- `calendars.events.list`, `calendars.events.get` (`id`), `calendars.events.create`, `calendars.events.update` (`id`), `calendars.events.delete` (`id`)
  - Event payload: `calendarId`, `contactId`, `title`, `startTime`, `endTime`, `appointmentStatus`
  - `calendars.events.list` accepts `calendarId` + `startTime`/`endTime` (or `startDate`/`endDate`) — dates are auto-converted to epoch-millis. A `calendarId`, `userId`, or `groupId` is **required**.
- `calendars.appointmentnotes.list` / `.get` / `.create` / `.update` / `.delete`

### Appointments — `appointments.*`
> Appointments are GHL calendar events. This is the recommended way to **book** an appointment.
- `appointments.create` — payload: `calendarId`, `contactId`, `title`, `startTime`, `endTime`, `appointmentStatus` (e.g. `confirmed`), optional `assignedUserId`, `notes`
- `appointments.get` — `id`
- `appointments.update` — `id` + payload
- `appointments.delete` — `id`
- `appointments.list` — `calendarId`/`userId`/`groupId` + `startTime`/`endTime` (required)

### Opportunities — `opportunities.*`
- `opportunities.list` / `opportunities.search` — filters auto-mapped (uses `location_id` internally); optional `query`, `pipeline_id`, `pipeline_stage_id`, `limit`
- `opportunities.get` (`id`), `opportunities.create`, `opportunities.update` (`id`), `opportunities.delete` (`id`)

### Pipelines — `pipelines.*`
- `pipelines.list`, `pipelines.get` (`id`), `pipelines.create`, `pipelines.update` (`id`), `pipelines.delete` (`id`)

### Forms — `forms.*`
- `forms.list`, `forms.get` (`id`), `forms.create`, `forms.update` (`id`), `forms.delete` (`id`)

### Tags — `tags.*`
- `tags.list`, `tags.get` (`id`), `tags.create`, `tags.update` (`id`), `tags.delete` (`id`)

### Custom Fields — `customfields.*`
- `customfields.list`, `customfields.get` (`id`), `customfields.create`, `customfields.update` (`id`), `customfields.delete` (`id`)
- `customfields.upload` — send `payload` (file upload custom field)

### Custom Values — `customvalues.*`
- `customvalues.list`, `customvalues.get` (`id`), `customvalues.create`, `customvalues.update` (`id`), `customvalues.delete` (`id`)

### Custom Objects — `objects.*` (alias `customobjects.*`)
- **Schemas:** `objects.list`, `objects.get` (`id`), `objects.create`, `objects.update` (`id`), `objects.delete` (`id`)
- **Records (rows inside a schema):** require `schemaKey` on every call.
  - `objects.records.list` — `schemaKey` (+ optional `params`)
  - `objects.records.get` — `schemaKey` + `id`
  - `objects.records.create` — `schemaKey` + payload
  - `objects.records.update` — `schemaKey` + `id` + payload
  - `objects.records.delete` — `schemaKey` + `id`

### Associations — `associations.*`
- **Definitions:** `associations.list`, `associations.get` (`id`), `associations.create`, `associations.update` (`id`), `associations.delete` (`id`)
- **Relations:** require `associationId`.
  - `associations.relations.list` — `associationId`
  - `associations.relations.create` — `associationId` + payload
  - `associations.relations.delete` — `associationId` + `relationId`
- `associations.byentity` — `entityId` (find associations for a record/contact)

### Media Storage — `media.*` (aliases `medias.*`, `mediastorage.*`)
- `media.list` — optional filters (`limit`, `query`, `parentId`, `type`); the bridge auto-adds the required `altId`/`altType`/`type`/`sortBy`/`sortOrder`
- `media.get` (`id`)
- `media.upload` — send `payload` with the file data (`url`, `name`, optional `folderId`)
- `media.delete` (`id`)

### Products — `products.*`
- `products.list` — filters: `limit`, `offset`, `search`
- `products.get` (`id`)
- `products.create` — payload: `name`, `productType` (`DIGITAL`/`PHYSICAL`/`SERVICE`), `description`
- `products.update` (`id`) + payload
- `products.delete` (`id`)
- **Prices** (require `productId`):
  - `products.prices.list` — `productId`
  - `products.prices.get` — `productId` + `priceId`
  - `products.prices.create` — `productId` + `name`, `type` (`one_time`/`recurring`), `currency`, `amount`
  - `products.prices.update` — `productId` + `priceId` + fields
  - `products.prices.delete` — `productId` + `priceId`

### Invoices — `invoices.invoice.*`
- `invoices.invoice.list`, `.get` (`id`), `.create`, `.update` (`id`), `.delete` (`id`)
- Shortcut `invoice.*` also routes here.

### Estimates — `invoices.estimate.*`
- `invoices.estimate.list`, `.get` (`id`), `.create`, `.update` (`id`), `.delete` (`id`)
- Shortcuts `estimate.*` / `estimates.*` also route here.

### Payments — `payments.*`
- **Coupons:** `payments.coupons.list` / `.get` / `.create` / `.update` / `.delete` (shortcut `coupons.*`)
- **Orders:** `payments.orders.list` / `.get` (shortcut `orders.*`)
- **Transactions:** `payments.transactions.list` / `.get` (shortcut `transactions.*`)

### Contracts — `contracts.*`
- `contracts.list`, `contracts.get` (`id`), `contracts.create`, `contracts.update` (`id`), `contracts.delete` (`id`)
- `contracts.getbycontact` — `contactId`

### Voice AI — `voiceai.*` (alias `conversationalai.*`)
- `voiceai.list`, `voiceai.get` (`id`), `voiceai.create`, `voiceai.update` (`id`), `voiceai.delete` (`id`)

### AI content generation — `ai.<mode>.generate`
- `ai.<mode>.generate` / `ai.<mode>.save` — server-side AI helper; pass mode-specific fields in `customData`.

---

## 5. Copy-paste examples for common UI patterns

### Lead capture form (create a contact)
```js
async function submitLead(form) {
  const result = await callGhl({
    type: 'contacts.create',
    locationId: '{{location.id}}',
    payload: {
      firstName: form.firstName.value,
      lastName:  form.lastName.value,
      email:     form.email.value,
      phone:     form.phone.value
    }
  });
  return result.data; // created contact
}
```

### Booking widget (list calendars → fetch availability)
```js
const calendars = (await callGhl({ type: 'calendars.list', locationId })).data;

const slots = (await callGhl({
  type: 'calendars.availability.get',
  locationId,
  calendarId: chosenCalendarId,
  startDate: '2026-06-01',
  endDate:   '2026-06-07',
  timezone:  'America/New_York',
  duration:  30
})).data;
```

### Book the appointment (use `appointments.create`)
```js
await callGhl({
  type: 'appointments.create',
  locationId,
  payload: {
    calendarId, contactId,
    title: 'Discovery Call',
    startTime: '2026-06-02T14:00:00-04:00',
    endTime:   '2026-06-02T14:30:00-04:00',
    appointmentStatus: 'confirmed'
  }
});
```

### Send an SMS from a chat UI
```js
await callGhl({
  type: 'conversations.messages.send',
  locationId,
  payload: { type: 'SMS', contactId, message: 'Thanks for reaching out!' }
});
```

### Log a note on a contact
```js
await callGhl({
  type: 'contacts.notes.create',
  locationId,
  contactId,
  body: 'Spoke with the client — ready to move forward.'
});
```

### Create a follow-up task
```js
await callGhl({
  type: 'contacts.tasks.create',
  locationId,
  contactId,
  title:   'Send proposal',
  body:    'Email the signed proposal PDF',
  dueDate: '2026-06-10T15:00:00Z'
});
```

### Product picker for an invoice (list products → list prices)
```js
const products = (await callGhl({ type: 'products.list', locationId, limit: 50 })).data;

const prices = (await callGhl({
  type: 'products.prices.list',
  locationId,
  productId: chosenProductId
})).data;
```

### Custom object record (e.g. a "Properties" object)
```js
await callGhl({
  type: 'objects.records.create',
  locationId,
  schemaKey: 'custom_objects.properties',
  payload: { name: '123 Main St', price: 450000, beds: 3 }
});
```

---

## 6. Response handling pattern (always generate this)

```js
try {
  const { data } = await callGhl({ type: 'contacts.list', locationId, limit: 20 });
  render(data);
} catch (err) {
  showError(err.message); // bridge returns { error } -> err.message
}
```

The payload of interest is **always** in `response.data`. The exact shape of `data` mirrors the
underlying GoHighLevel API response for that resource (e.g. `data.contacts`, `data.contact`,
`data.calendars`, `data.events`).

---

## 7. Authentication models (important for the AI to pick correctly)

- **Browser / website front-end (recommended path):** No secret needed. CORS is open, so a
  `fetch` from the page just works. Generate exactly the helper in §2.
- **Server-to-server / GHL workflow webhook (no `Origin` header):** A secret **is** required.
  Provide the location's webhook secret in any one of:
  - `customData.secretKey`
  - `customData.webhookSecret`
  - top-level `secretKey` / `webhookSecret`
  - header `X-Webhook-Secret: <secret>`
  - header `Authorization: Bearer <secret>`

If the AI is generating front-end JS that runs in a browser, **do not** embed the secret — it's
not needed and would be exposed. Only use the secret for backend/workflow calls.

---

## 8. Troubleshooting (diagnose in this order)

| Error / symptom | Cause | Fix |
|-----------------|-------|-----|
| `Missing locationId in payload` | No `customData.locationId` | Add it (`{{location.id}}` in GHL) |
| `Location not found in database` | Location not installed / wrong id | Reinstall the app on that sub-account |
| `Invalid actionType...` | Bad format | Use `resource.action` / `resource.subresource.action` |
| `Unknown action '...'` | Verb not supported for that resource | Check §4 catalog |
| `Missing id for ...` | get/update/delete without an id | Add `id` |
| `Missing create/post fields` | create/update with empty body | Add fields or `payload` |
| 401 `Unauthorized webhook request` | Server call with wrong/missing secret | Send correct webhook secret (§7) |
| Browser CORS error | Network/URL issue (CORS is otherwise open) | Verify endpoint URL + that the page used POST + JSON |

---

## 9. Style policy for the generating AI

1. Always use the `callGhl` helper — never hand-roll `fetch` per call.
2. Always wrap calls in `try/catch` and surface `err.message` in the UI.
3. Read results from `response.data`.
4. For destructive actions (`*.delete`), add a confirmation step in the UI.
5. Never invent an action or a field — stay inside the §4 catalog.
6. Default to `payload: {}` for create/update for clarity.
7. Use `{{location.id}}` as the `locationId` placeholder when generating for GHL-hosted pages.
