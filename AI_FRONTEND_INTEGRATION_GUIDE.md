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
4. The server replies with `{ "success": true, "action": "...", "data": { ... } }`. Always check `data.httpCode` — a `2xx` means success; `4xx`/`5xx` means failure even if `success: true` appears in the wrapper.

---

## 2. Drop-in helper (always generate this once per page)

```js
async function callGhl(customData) {
  const ENDPOINT = 'https://estimatr.tools/ghl_api/controller.php';

  const res = await fetch(ENDPOINT, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      actionType: customData.type,
      customData
    })
  });

  const text = await res.text();
  let json;
  try { json = JSON.parse(text); } catch { json = { raw: text }; }

  if (!res.ok) throw new Error(json.error || json.message || `HTTP ${res.status}`);

  // Treat GHL-level 4xx/5xx as errors too
  const httpCode = json?.data?.httpCode ?? 200;
  if (httpCode >= 400) {
    const msg = Array.isArray(json?.data?.message)
      ? json.data.message.join(', ')
      : (json?.data?.message || `API error ${httpCode}`);
    throw new Error(msg);
  }

  return json; // -> { success, action, data }
}
```

CORS is open for browser calls. Server-to-server calls require the webhook secret — see §7.

---

## 3. How to shape `customData` per action verb

| Verb | What to send in `customData` | Example keys |
|------|------------------------------|--------------|
| `list` / `search` | Filters as flat fields (or a `params` object) | `query`, `limit`, `page` |
| `get` | The record id | `id` (preferred) |
| `create` / `post` | Fields flat or inside `payload: {}` | `payload: { firstName, email }` |
| `update` / `put` | The `id` **plus** fields flat or inside `payload: {}` | `id` + `payload: {...}` |
| `delete` | The record id | `id` |

**Always prefer `payload: { ... }` for create/update** — it is the clearest form and least error-prone.

**IDs:** Always send `id`. The bridge also accepts resource-specific aliases: `contactId`, `noteId`, `taskId`, `userId`, `customFieldId`, `conversationId`, `couponId`, `invoiceId`, `estimateId`, `orderId`, `transactionId`, `calendarId`, `opportunityId`, `paymentId`, `schemaKey`, `productId`, `priceId`.

---

## 4. Complete action catalog

> **Never invent an action or field name.** Only the actions listed here exist.

### Contacts — `contacts.*`

- `contacts.list` — filters: `query`, `limit`, `page`
- `contacts.search` — filters: `query`, `limit`
- `contacts.get` — `id`
- `contacts.upsert` — **use this instead of `contacts.create` when the contact may already exist** (matches on email/phone, returns existing contact instead of erroring). Payload: `firstName`, `lastName`, `email`, `phone`, `tags`, `customFields`
- `contacts.create` — same payload as upsert; use only when you are certain the contact is new
- `contacts.update` — `id` + payload
- `contacts.delete` — `id`

**Notes** (require `contactId`):
- `contacts.notes.list` — `contactId`
- `contacts.notes.get` — `contactId` + `noteId`
- `contacts.notes.create` — `contactId` + `body` (optional `userId`)
- `contacts.notes.update` — `contactId` + `noteId` + `body`
- `contacts.notes.delete` — `contactId` + `noteId`

**Tasks** (require `contactId`):
- `contacts.tasks.list` — `contactId`
- `contacts.tasks.get` — `contactId` + `taskId`
- `contacts.tasks.create` — `contactId` + `title`, `body`, `dueDate` (ISO 8601), optional `assignedTo`, `completed`
- `contacts.tasks.update` — `contactId` + `taskId` + fields
- `contacts.tasks.complete` — `contactId` + `taskId` + `completed` (boolean)
- `contacts.tasks.delete` — `contactId` + `taskId`

### Users — `users.*`
- `users.list`, `users.get` (`id`), `users.create`, `users.update` (`id`), `users.delete` (`id`)

### Conversations & Messaging — `conversations.*`
- `conversations.list` / `conversations.search` — filters: `query`, `contactId`, `limit`
- `conversations.get` — `id`
- `conversations.create` — payload: `contactId`, `locationId`
- `conversations.update` — `id` + payload
- `conversations.delete` — `id`
- `conversations.markread` — `id`
- **Messages:**
  - `conversations.messages.list` — `conversationId`
  - `conversations.messages.get` — `id`
  - `conversations.messages.send` — payload: `type` (`SMS`), `contactId`, `message`
  - `conversations.messages.email` — payload: `contactId`, `subject`, `html`/`message`
  - `conversations.messages.update` — `id` + payload
  - `conversations.messages.delete` — `id`
  - `conversations.messages.cancel` — `id`
- **Import:** `conversations.import` — `contactName`/`contactEmail`/`contactPhone` + `textsBlob`/`callsBlob`

### Calendars & Appointments — `calendars.*`
- `calendars.list`, `calendars.get` (`id`), `calendars.create`, `calendars.update` (`id`), `calendars.delete` (`id`)
- `calendars.availability.get` — `calendarId`, `startDate`, `endDate`, `timezone`, `duration`
- `calendars.services.list`, `calendars.services.get` (`id`)
- `calendars.events.list` — requires `calendarId`, `userId`, or `groupId` + `startTime`/`endTime`; dates auto-converted to epoch-millis
- `calendars.events.get` / `.create` / `.update` / `.delete`
- `calendars.appointmentnotes.list` / `.get` / `.create` / `.update` / `.delete`

### Appointments — `appointments.*`
> Preferred way to book appointments.
- `appointments.create` — payload: `calendarId`, `contactId`, `title`, `startTime`, `endTime`, `appointmentStatus` (`confirmed`/`pending`/`cancelled`), optional `notes`
- `appointments.get` — `id`
- `appointments.update` — `id` + payload
- `appointments.delete` — `id`
- `appointments.list` — `calendarId`/`userId`/`groupId` + `startTime`/`endTime` (required)

### Opportunities — `opportunities.*`
- `opportunities.search` / `opportunities.list` — optional: `query`, `pipeline_id`, `pipeline_stage_id`, `limit`
- `opportunities.get` (`id`), `opportunities.create`, `opportunities.update` (`id`), `opportunities.delete` (`id`)

### Pipelines — `pipelines.*`
- `pipelines.list`, `pipelines.get` (`id`), `pipelines.create`, `pipelines.update` (`id`), `pipelines.delete` (`id`)

### Forms — `forms.*`
- `forms.list`, `forms.get` (`id`), `forms.create`, `forms.update` (`id`), `forms.delete` (`id`)

### Tags — `tags.*`
- `tags.list`, `tags.get` (`id`), `tags.create`, `tags.update` (`id`), `tags.delete` (`id`)

### Custom Fields — `customfields.*`
- `customfields.list`, `customfields.get` (`id`), `customfields.create`, `customfields.update` (`id`), `customfields.delete` (`id`)

### Custom Values — `customvalues.*`
- `customvalues.list`, `customvalues.get` (`id`), `customvalues.create`, `customvalues.update` (`id`), `customvalues.delete` (`id`)

### Custom Objects — `objects.*` (alias `customobjects.*`)

**Schemas (object type definitions):**
- `objects.list` — lists all object schemas; each has a `key` field you need for records
- `objects.get` (`id`)
- `objects.create` — payload: `name`, `labels: { singular, plural }`, optional `primaryDisplayProperty` (string field key), `searchableProperties` (array). Do NOT include a `properties` array — GHL does not accept fields at creation time.
- `objects.update` (`id`) + payload
- `objects.delete` (`id`)

**Records** — require `schemaKey` on every call. `schemaKey` is the full dotted key from `objects.list` (e.g. `custom_objects.events`):
- `objects.records.list` — `schemaKey`
- `objects.records.get` — `schemaKey` + `id`
- `objects.records.create` — `schemaKey` + `payload: { properties: { fieldKey: value } }`
- `objects.records.update` — `schemaKey` + `id` + `payload: { properties: { fieldKey: value } }`
- `objects.records.delete` — `schemaKey` + `id`

> **CRITICAL for records:** All custom field values MUST go inside `properties: {}`. Use the **short field key** (e.g. `events`), never the fully-qualified key (`custom_objects.events.events`). Field keys come from the schema returned by `objects.list`.

### Associations — `associations.*`

**Definitions** (what object types can be linked):
- `associations.list` — call this once to find your `associationId` for a given pair of object types
- `associations.get` (`id`), `associations.create`, `associations.update` (`id`), `associations.delete` (`id`)

**Relations** (actual links between two records):
- `associations.relations.list` — `recordId` (the contact or object record id whose links you want)
- `associations.relations.create` — payload: `associationId`, `firstRecordId`, `secondRecordId`
- `associations.relations.delete` — `relationId`
- `associations.byentity` — `entityId`

> **How to link a contact to an object record:**
> 1. Call `associations.list` once. Find the entry where the two sides match your object types. Save its `id` as your `associationId`.
> 2. Call `associations.relations.create` with that `associationId` + `firstRecordId` (contact id) + `secondRecordId` (object record id).

### Media Storage — `media.*` (aliases `medias.*`, `mediastorage.*`)

- `media.list` — optional: `limit`, `query`, `parentId`
- `media.get` (`id`)
- `media.delete` (`id`)
- `media.upload` — send inside `payload`:

| Method | When to use | How |
|--------|-------------|-----|
| Hosted URL | File is already publicly accessible online — **always prefer this** | `payload: { name: "photo.jpg", file: "https://..." }` |
| Base64 / data URI | Local file upload — **max 4 MB only** | `payload: { name: "photo.jpg", file: "data:image/jpeg;base64,..." }` |

> For base64 uploads, enforce on the frontend before sending: max ~4 MB original file size, accepted types: jpeg, png, gif, webp, pdf. For larger files, upload to storage first then send the URL.

### Products — `products.*`
- `products.list` — filters: `limit`, `offset`, `search`
- `products.get` (`id`)
- `products.create` — payload: `name`, `productType` (`DIGITAL`/`PHYSICAL`/`SERVICE`), `description`
- `products.update` (`id`) + payload
- `products.delete` (`id`)
- **Prices** (require `productId`):
  - `products.prices.list` — `productId`
  - `products.prices.get` — `productId` + `priceId`
  - `products.prices.create` — `productId` + payload: `name`, `type` (`one_time`/`recurring`), `currency`, `amount`
  - `products.prices.update` — `productId` + `priceId` + fields
  - `products.prices.delete` — `productId` + `priceId`

### Invoices — `invoices.invoice.*`
- `invoices.invoice.list`, `.get` (`id`), `.create`, `.update` (`id`), `.delete` (`id`)
- Shortcut `invoice.*` also routes here.
- Each line item must have `type: "service"` or `type: "product"`. No other values.

### Estimates — `invoices.estimate.*`
- `invoices.estimate.list`, `.get` (`id`), `.create`, `.update` (`id`), `.delete` (`id`)
- Shortcuts `estimate.*` / `estimates.*` also route here.

> **Estimate-specific rules — read carefully, these differ from invoices:**
>
> **Items:** Do NOT send a `type` field on estimate line items. The server forces the correct internal value automatically. Only send `name`, `quantity`, `amount`, `currency` per item.
>
> **frequencySettings:** This object is **required** on every estimate create/update. Always include it exactly as shown:
> ```json
> "frequencySettings": { "enabled": false, "type": "one_time" }
> ```
> `enabled` must be an explicit boolean (`true`/`false`), never a string or omitted. Valid `type` values: `one_time`, `weekly`, `bi_weekly`, `monthly`, `quarterly`, `semi_annual`, `annual`.
>
> **name:** Max 40 characters. Truncate before sending.
>
> **Minimum required fields for a working estimate:**
> ```json
> {
>   "name": "Roof Repair Estimate",
>   "contactId": "REAL_CONTACT_ID",
>   "issueDate": "2026-06-05",
>   "currency": "USD",
>   "items": [
>     { "name": "Labor", "quantity": 2, "amount": 150.00, "currency": "USD" }
>   ],
>   "frequencySettings": { "enabled": false, "type": "one_time" }
> }
> ```

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
- `ai.<mode>.generate` / `ai.<mode>.save` — pass mode-specific fields in `customData`

---

## 5. Copy-paste examples for common UI patterns

### Upsert a contact (safe for forms — won't duplicate)
```js
const result = await callGhl({
  type: 'contacts.upsert',
  locationId,
  payload: {
    firstName: 'Nick',
    lastName:  'Valencia',
    email:     'nick@example.com',
    phone:     '3174355613'
  }
});
const contactId = result.data.contact.id; // capture this for downstream calls
```

### Book an appointment (contact id required — upsert first if needed)
```js
await callGhl({
  type: 'appointments.create',
  locationId,
  payload: {
    calendarId,
    contactId,              // must be a real id, not a placeholder
    title: 'Discovery Call',
    startTime: '2026-06-10T14:00:00-04:00',
    endTime:   '2026-06-10T14:30:00-04:00',
    appointmentStatus: 'confirmed'
  }
});
```

### Create an estimate
```js
await callGhl({
  type: 'invoices.estimate.create',
  locationId,
  payload: {
    name:       'Roof Repair Estimate',  // max 40 chars
    contactId,
    issueDate:  '2026-06-05',            // YYYY-MM-DD
    expiryDate: '2026-07-05',
    currency:   'USD',
    items: [
      { name: 'Labor',    quantity: 4,  amount: 150.00, currency: 'USD' },
      { name: 'Shingles', quantity: 10, amount:  45.00, currency: 'USD' }
      // Do NOT include 'type' on estimate items
    ],
    discount: { type: 'percentage', value: 0 },
    frequencySettings: { enabled: false, type: 'one_time' }  // always required
  }
});
```

### Create a custom object schema
```js
await callGhl({
  type: 'objects.create',
  locationId,
  payload: {
    name: 'Repairs',
    labels: { singular: 'Repair', plural: 'Repairs' },
    primaryDisplayProperty: 'name',     // short field key
    searchableProperties: ['name']
    // Do NOT include a 'properties' array — add fields separately after creation
  }
});
```

### Create a custom object record
```js
await callGhl({
  type: 'objects.records.create',
  locationId,
  schemaKey: 'custom_objects.events',   // full dotted key from objects.list
  payload: {
    properties: {
      events: 'Spring Gala',            // SHORT field key, not 'custom_objects.events.events'
      venue:  'Grand Ballroom'
    }
  }
});
```

### Link a contact to an object record (association)
```js
// Step 1 (one-time setup): find the associationId
const defs = await callGhl({ type: 'associations.list', locationId });
// Find entry where firstObjectKey === 'contact' and secondObjectKey === 'custom_objects.events'
const associationId = defs.data.associations[0].id; // save this in your config

// Step 2: create the relation
await callGhl({
  type: 'associations.relations.create',
  locationId,
  payload: { associationId, firstRecordId: contactId, secondRecordId: eventRecordId }
});
```

### Upload a media file (prefer hosted URL)
```js
// Mode A — hosted URL (recommended, no size limit)
await callGhl({
  type: 'media.upload',
  locationId,
  payload: { name: 'photo.jpg', file: 'https://example.com/photo.jpg' }
});

// Mode B — base64 (local files only, enforce ≤ 4 MB on the frontend before sending)
await callGhl({
  type: 'media.upload',
  locationId,
  payload: { name: 'photo.jpg', file: 'data:image/jpeg;base64,...' }
});
```

### Send an SMS
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

### Create a task on a contact
```js
await callGhl({
  type: 'contacts.tasks.create',
  locationId,
  contactId,
  title:   'Send proposal',
  dueDate: '2026-06-10T15:00:00Z'
});
```

---

## 6. Response handling (always generate this pattern)

```js
try {
  const { data } = await callGhl({ type: 'contacts.list', locationId, limit: 20 });
  render(data);
} catch (err) {
  showError(err.message);
}
```

The payload of interest is always in `response.data`. Shape mirrors the GHL API response for that resource (`data.contacts`, `data.contact`, `data.calendars`, `data.events`, `data.records`, etc.).

---

## 7. Authentication models

- **Browser / website front-end:** No secret needed. CORS is open. Use the helper in §2 as-is.
- **Server-to-server / GHL workflow webhook (no `Origin` header):** A secret is required. Send it in any one of:
  - `customData.secretKey` or `customData.webhookSecret`
  - top-level `secretKey` / `webhookSecret`
  - header `X-Webhook-Secret: <secret>`
  - header `Authorization: Bearer <secret>`

Never embed the secret in browser-side code — it is not needed there and would be exposed.

---

## 8. Hard rules — things that will always cause errors if violated

| Rule | Wrong | Right |
|------|-------|-------|
| Object record fields | Sent flat or with long key `custom_objects.events.fieldName` | Always inside `properties: { shortKey: value }` |
| Estimate item type | Sending `type: "product"`, `"service"`, `"custom"` on estimate items | Do not send `type` on estimate items at all |
| Invoice item type | Sending `type: "custom"` | Only `"service"` or `"product"` |
| frequencySettings on estimates | Omitting it, or `enabled` as string/missing | Always `{ "enabled": false, "type": "one_time" }` |
| Estimate name length | More than 40 characters | Truncate to 40 before sending |
| contacts.create on duplicate | Fails with 400 if location blocks duplicates | Use `contacts.upsert` — returns existing contact |
| Placeholder ids | `"YOUR_CONTACT_ID_HERE"` | Only send real ids captured from previous API calls |
| media.upload base64 size | File > ~4 MB | Upload to a CDN first, send URL instead |
| objects.create with properties | Sending `properties: [...]` array | Omit it — add fields separately after creation |
| Dates | `"June 4, 2026"` | Always `"YYYY-MM-DD"` or ISO 8601 (`2026-06-04T14:00:00Z`) |

---

## 9. Troubleshooting reference

| Error / symptom | Cause | Fix |
|-----------------|-------|-----|
| `Missing locationId in payload` | No `customData.locationId` | Add it (`{{location.id}}` in GHL) |
| `Location not found in database` | Wrong id or location not installed | Reinstall the app on that sub-account |
| `Unknown action '...'` | Verb not in catalog | Check §4 |
| `Missing id for ...` | get/update/delete without an id | Add `id` |
| `properties must be an object` | Object record fields sent flat | Wrap in `properties: {}` |
| `property X should not exist` | Unknown field key on object record | Use real keys from `objects.list` |
| `items.0.type must be a valid enum value` | Sent a type on estimate items | Remove `type` from estimate items |
| `frequencySettings.enabled must be a boolean` | `enabled` missing or wrong type | Send `"enabled": false` explicitly |
| `This location does not allow duplicated contacts` | Used `contacts.create` on existing contact | Use `contacts.upsert` |
| `Contact with id ... not found` | Sent a placeholder contactId | Use real id from upsert/create response |
| 401 `Unauthorized webhook request` | Server call without secret | Send webhook secret (§7) |

---

## 10. Style policy for the generating AI

1. Always use the `callGhl` helper — never hand-roll `fetch` per call.
2. Always wrap calls in `try/catch` and surface `err.message` in the UI.
3. Read results from `response.data`.
4. For destructive actions (`*.delete`), add a confirmation step in the UI.
5. Never invent an action or field name — stay inside the §4 catalog.
6. Always use `payload: {}` for create/update.
7. Use `{{location.id}}` as the `locationId` placeholder for GHL-hosted pages.
8. Use `contacts.upsert` by default — only use `contacts.create` when you are certain the contact cannot exist yet.
9. Never send placeholder strings (`YOUR_..._HERE`) — always use real ids captured from earlier calls in the same flow.
10. For multi-step flows (upsert contact → create appointment → create object record → link association), execute steps in order and pass ids forward from each response.
