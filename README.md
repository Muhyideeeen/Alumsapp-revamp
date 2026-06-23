# alumsapp Custom Domains — Developer Handoff

**Feature:** Let each organisation use their own domain for their **public website** (e.g. `opus.com` replacing `opus.website.alumsapp.com`).
**Audience:** Backend + frontend developer(s).
**Status legend:** `[DONE]` already implemented · `[TODO]` to build.

---

## 1. Scope

In scope:
- The **public website** moves to the customer's own domain (`opus.com`).
- Onboarding stays automated: the customer adds DNS records, the system verifies ownership, and the site goes live with HTTPS automatically.
- A **single shared frontend** keeps rendering each org's branding dynamically from the incoming web address.

Out of scope (do **not** change):
- The **member portal** stays on `opus.alumsapp.com` exactly as today. Customer logins are unaffected.
- Existing `*.website.alumsapp.com` / `*.alumsapp.com` addresses keep working throughout.

You are building the **application** half of this feature. The certificate/edge infrastructure (Caddy, the load balancer, DynamoDB, deployment) is handled separately by DevOps — see section 7.

---

## 2. Decisions already made (context you can rely on)

- **Certificates** are issued and renewed automatically by the edge (Caddy + Let's Encrypt). Your code never touches certificates directly — it only tells the edge *which domains are allowed* (Ticket 4).
- **One active custom domain per organisation** (replaceable). Enforced in the data model.
- **Certificate storage** is AWS DynamoDB, owned by DevOps. Not your concern.
- A domain becomes usable only after the customer proves they own it via a DNS **TXT** record.

---

## 3. How it fits together (so you can see where your code sits)

1. A visitor opens `https://opus.com`.
2. The request reaches the edge (Caddy). If Caddy has never served this domain, it first calls **your internal TLS-check endpoint** (Ticket 4) to confirm the domain is allowed, then obtains a certificate automatically.
3. Caddy forwards the request to the **existing single frontend**, passing the original hostname.
4. The frontend/back end **resolves the organisation from the hostname** (Ticket 2) and renders that org's branding — exactly as it does today for platform subdomains.

Your code provides: the data model (1), hostname→org resolution (2), the onboarding + verification endpoints (3), the internal TLS-check endpoint (4), the onboarding UI (5), a link fix (6), and a background safety job (7).

---

## 4. Security principles (apply to EVERY ticket)

These are not optional polish — they are the security boundaries of the feature:

1. **Treat the hostname as untrusted input.** It comes from the visitor's browser. Always normalise and look it up; never trust it directly or interpolate it into queries unescaped.
2. **Only `active` domains resolve or get certificates.** `pending` / `verifying` / `disabled` domains must never return an organisation (Ticket 2) and must never be authorised for a certificate (Ticket 4).
3. **Fail closed.** On any error (DB down, lookup failure), the safe answer is "no" — do not resolve an org, do not authorise a certificate.
4. **No cross-tenant leakage.** An unknown or unverified host shows a neutral default page, never another organisation's branding, and never a raw error/stack trace.
5. **Verification tokens are secrets.** Generate them with a cryptographically secure random source; never log them or expose them to other organisations.
6. **The TLS-check endpoint is internal-only.** It is reachable only by the edge, protected by both network isolation (DevOps) and a shared secret (your code).
7. **A customer can never claim a platform domain.** Reject any hostname ending in `alumsapp.com`.

---

## 5. Configuration the team must provide

These values are supplied by the coordinator / DevOps, not invented by the developer. Read them from environment variables:

| Variable | Provided by | Meaning |
|---|---|---|
| `PLATFORM_EDGE_IPS` | DevOps | Comma-separated list of the two permanent public IPs customers point their domain at (already allocated). |
| `PLATFORM_EDGE_HOSTNAME` | DevOps | A platform hostname (e.g. `ingress.alumsapp.com`) that `www.<customer-domain>` will CNAME to. |
| `TLS_CHECK_SHARED_SECRET` | DevOps | A long random secret the edge sends with every TLS-check call; your endpoint rejects calls without it. |
| `VERIFY_TXT_PREFIX` | Default `_alumsapp-verify` | The label prepended to the customer's domain for the ownership TXT record. |

---

## 6. The tickets

### Ticket 1 — `custom_domains` data model `[DONE]`

A `custom_domains` collection mapping each customer website domain to an organisation and its verification state.

```js
// models/CustomDomain.js
const { Schema, model } = require('mongoose');

const CustomDomainSchema = new Schema({
  organizationId: { type: Schema.Types.ObjectId, ref: 'Organization', required: true, index: true },
  hostname:       { type: String, required: true, unique: true, lowercase: true, trim: true },
  status:         { type: String, enum: ['pending', 'verifying', 'active', 'disabled'], default: 'pending', index: true },
  verificationToken:  { type: String, required: true },
  verificationMethod: { type: String, default: 'dns-txt' },
  verifiedAt:    { type: Date, default: null },
  lastCheckedAt: { type: Date, default: null },
  lastError:     { type: String, default: null },
  failureCount:  { type: Number, default: 0 },   // used by the re-check job (Ticket 7)
}, { timestamps: true });

// One active custom domain per organisation
CustomDomainSchema.index(
  { organizationId: 1 },
  { unique: true, partialFilterExpression: { status: 'active' } }
);

module.exports = model('CustomDomain', CustomDomainSchema);
```

*(`failureCount` was added for Ticket 7; include it if not already present.)*

**Status lifecycle:** `pending` → `verifying` → `active` → `disabled`.

---

### Ticket 2 — Resolve the organisation from any hostname `[DONE]`

A single function all tenant detection flows through. Returns an org for platform subdomains (as today) and for **active** custom domains; returns nothing for unknown/unverified hosts.

```js
// services/resolveOrganization.js
const CustomDomain = require('../models/CustomDomain');
const Organization = require('../models/Organization');

const PLATFORM_SUFFIXES = ['.website.alumsapp.com', '.alumsapp.com'];

function normalizeHost(raw) {
  if (!raw) return null;
  let host = String(raw).toLowerCase().trim();
  host = host.replace(/^https?:\/\//, '').split('/')[0].split(':')[0].replace(/\.$/, '');
  return host || null;
}

async function resolveOrganization(rawHost) {
  const host = normalizeHost(rawHost);
  if (!host) return { org: null, reason: 'no-host' };

  for (const suffix of PLATFORM_SUFFIXES) {
    if (host.endsWith(suffix)) {
      const slug = host.slice(0, -suffix.length);
      if (!slug || slug.includes('.')) continue;
      const org = await Organization.findOne({ slug });
      return org ? { org, via: 'platform-subdomain' } : { org: null, reason: 'unknown-slug' };
    }
  }

  const domain = await CustomDomain.findOne({ hostname: host, status: 'active' });
  if (domain) {
    const org = await Organization.findById(domain.organizationId);
    return org ? { org, via: 'custom-domain' } : { org: null, reason: 'org-missing' };
  }

  return { org: null, reason: 'unknown-host' };
}

module.exports = { resolveOrganization, normalizeHost };
```

**Key rule:** only `status: 'active'` resolves. Unknown hosts must render a safe default.

---

### Ticket 3 — Domain onboarding + verification endpoints `[TODO]`

**Goal:** let an organisation add their domain, see the DNS records to create, and have the system verify ownership and activate the domain.

**Endpoints (all authenticated as the organisation; an org may only see/modify its own domains):**

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/custom-domains` | Add a domain; returns the DNS records the customer must create. |
| `GET` | `/api/custom-domains` | List this org's domains with status. |
| `POST` | `/api/custom-domains/:id/verify` | Run a verification check now. |
| `DELETE` | `/api/custom-domains/:id` | Remove/disable a domain. |

```js
// routes/customDomains.js
const crypto = require('crypto');
const dns = require('dns').promises;
const CustomDomain = require('../models/CustomDomain');
const { normalizeHost } = require('../services/resolveOrganization');

const VERIFY_PREFIX = process.env.VERIFY_TXT_PREFIX || '_alumsapp-verify';
const EDGE_IPS = (process.env.PLATFORM_EDGE_IPS || '').split(',').map(s => s.trim()).filter(Boolean);
const EDGE_HOSTNAME = process.env.PLATFORM_EDGE_HOSTNAME;

function dnsInstructions(hostname, token) {
  return {
    records: [
      { type: 'A',     name: hostname,                       value: EDGE_IPS,      note: 'Makes your domain reachable' },
      { type: 'TXT',   name: `${VERIFY_PREFIX}.${hostname}`, value: token,         note: 'Proves you own the domain' },
      { type: 'CNAME', name: `www.${hostname}`,              value: EDGE_HOSTNAME, note: 'Optional: makes www work too' },
    ],
  };
}

// POST /api/custom-domains   body: { hostname }
async function addDomain(req, res) {
  const orgId = req.organization.id;                 // from your auth middleware
  const hostname = normalizeHost(req.body.hostname);

  if (!hostname || !/^[a-z0-9-]+(\.[a-z0-9-]+)+$/.test(hostname) || hostname.endsWith('alumsapp.com')) {
    return res.status(400).json({ error: 'invalid_domain' });
  }
  if (await CustomDomain.findOne({ hostname })) {
    return res.status(409).json({ error: 'domain_taken' });   // unique index also enforces this
  }

  const token = crypto.randomBytes(16).toString('hex');
  const doc = await CustomDomain.create({
    organizationId: orgId, hostname, status: 'pending', verificationToken: token,
  });
  return res.status(201).json({
    id: doc.id, hostname, status: doc.status, dns: dnsInstructions(hostname, token),
  });
}

// POST /api/custom-domains/:id/verify
async function verifyDomain(req, res) {
  const doc = await CustomDomain.findOne({ _id: req.params.id, organizationId: req.organization.id });
  if (!doc) return res.status(404).json({ error: 'not_found' });

  doc.status = 'verifying';
  doc.lastCheckedAt = new Date();
  try {
    const txt = await dns.resolveTxt(`${VERIFY_PREFIX}.${doc.hostname}`).catch(() => []);
    const ownershipOk = txt.map(chunks => chunks.join('')).includes(doc.verificationToken);

    const aRecords = await dns.resolve4(doc.hostname).catch(() => []);
    const routingOk = EDGE_IPS.length === 0 || aRecords.some(ip => EDGE_IPS.includes(ip));

    if (ownershipOk && routingOk) {
      doc.status = 'active'; doc.verifiedAt = new Date(); doc.lastError = null; doc.failureCount = 0;
    } else {
      doc.status = 'pending';
      doc.lastError = !ownershipOk ? 'txt_not_found' : 'a_record_not_pointing_to_platform';
    }
  } catch (e) {
    doc.status = 'pending'; doc.lastError = 'dns_lookup_failed';
  }
  await doc.save();
  return res.json({ id: doc.id, hostname: doc.hostname, status: doc.status, lastError: doc.lastError });
}

// GET /api/custom-domains
async function listDomains(req, res) {
  const docs = await CustomDomain.find({ organizationId: req.organization.id });
  return res.json(docs.map(d => ({
    id: d.id, hostname: d.hostname, status: d.status, lastError: d.lastError,
    dns: dnsInstructions(d.hostname, d.verificationToken),
  })));
}

// DELETE /api/custom-domains/:id  -> set disabled (preferred over hard delete for audit)
async function removeDomain(req, res) {
  const doc = await CustomDomain.findOneAndUpdate(
    { _id: req.params.id, organizationId: req.organization.id },
    { status: 'disabled' }, { new: true },
  );
  if (!doc) return res.status(404).json({ error: 'not_found' });
  return res.json({ id: doc.id, status: doc.status });
}

module.exports = { addDomain, verifyDomain, listDomains, removeDomain };
```

**Rules:**
- A domain only becomes `active` when **both** the TXT ownership record matches **and** the apex `A` records point to our IPs.
- Verification must be idempotent and safe to call repeatedly (e.g. a "Check now" button).
- Never expose another org's domains or tokens.

**Acceptance criteria:**
1. Adding a domain returns the three DNS records, with the real platform IPs and the org's unique token.
2. Adding a hostname already in use returns `409`; adding a `*.alumsapp.com` or malformed host returns `400`.
3. With the correct TXT + A records in place, `verify` moves the domain to `active`; otherwise it stays `pending` with a clear `lastError`.
4. All endpoints scope strictly to the authenticated organisation.

---

### Ticket 4 — Internal TLS-check ("ask") endpoint `[TODO]`

**Goal:** the edge (Caddy) calls this **before** issuing a certificate for a domain. It must answer "yes" **only** for domains that are `active`. This is the single most security-sensitive endpoint in the feature.

```js
// routes/internalTlsCheck.js  — INTERNAL ONLY
const CustomDomain = require('../models/CustomDomain');
const { normalizeHost } = require('../services/resolveOrganization');

const SHARED_SECRET = process.env.TLS_CHECK_SHARED_SECRET;

// GET /internal/tls-check?domain=<host>     (Caddy issues a cert only on HTTP 200)
async function tlsCheck(req, res) {
  if (!SHARED_SECRET || req.get('X-TLS-Check-Secret') !== SHARED_SECRET) {
    return res.status(403).end();                 // reject anything without the shared secret
  }
  const host = normalizeHost(req.query.domain);
  if (!host) return res.status(400).end();

  try {
    const domain = await CustomDomain.findOne({ hostname: host, status: 'active' }).lean();
    return domain ? res.status(200).end() : res.status(404).end();
  } catch (e) {
    return res.status(503).end();                 // FAIL CLOSED: error => do NOT authorise
  }
}

module.exports = { tlsCheck };
```

**Rules:**
- Mount this on an internal route/port that is **not** exposed to the public internet (DevOps will also restrict it at the network level).
- It must be fast — it sits inside the visitor's first connection. Index on `hostname`+`status` already supports this.
- Any non-200 response means "do not issue." On uncertainty, return non-200.

**Acceptance criteria:**
1. A request without the correct `X-TLS-Check-Secret` header returns `403`.
2. `?domain=` of an `active` domain returns `200`; a `pending`/`disabled`/unknown domain returns non-200.
3. A simulated DB error returns a 5xx (never `200`).
4. The route is not reachable from the public internet.

---

### Ticket 5 — Onboarding UI (frontend) `[TODO]`

**Goal:** a screen in the organisation's settings where they add and verify their domain. It consumes the Ticket 3 endpoints.

**Requirements:**
- An input to enter a domain → calls `POST /api/custom-domains`.
- Display the returned DNS records in a clear table, **each with a copy button** (record type, name/host, value). Show the A records and the TXT record prominently; mark the `www` CNAME as optional.
- Show the current **status** with plain wording: *Pending DNS* → *Checking* → *Live*. Show `lastError` as friendly guidance (e.g. "We couldn't find the verification record yet — DNS changes can take up to an hour.").
- A **"Check now"** button → calls the `verify` endpoint and refreshes status. Optionally poll every ~30s while `pending`/`verifying`.
- A way to remove a domain → `DELETE`.

**Acceptance criteria:**
1. A non-technical customer can add a domain, copy the records, and reach a "Live" state without help.
2. Status and errors are always shown in plain language, never raw codes.
3. The screen only ever shows the current organisation's domain.

---

### Ticket 6 — Fix the website → portal link `[TODO]`

**Goal:** because the website (`opus.com`) and the member portal (`opus.alumsapp.com`) are now on different addresses, every "Login" / "Member area" link on the public website must point to the **portal address for that organisation**, derived from the org — not hardcoded and not assumed to be same-origin.

**Requirements:**
- Build the portal URL from the organisation's platform portal host (e.g. `https://<slug>.alumsapp.com`).
- Audit the website templates/components for any hardcoded `*.alumsapp.com` or same-origin assumptions in auth/portal links and replace them with the derived URL.

**Acceptance criteria:**
1. From a website served on `opus.com`, the login/member link goes to that org's portal on `*.alumsapp.com`.
2. No hardcoded platform hostnames remain in the website's auth/portal links.

---

### Ticket 7 — Background re-check & deactivation job `[TODO]`

**Goal:** detect when a customer later removes or re-points their DNS, and deactivate the domain so we stop serving and renewing it (prevents "dangling domain" risk).

**Requirements:**
- A scheduled job (e.g. every 6 hours) that, for each `active` custom domain, re-checks that the apex `A` records still include one of our IPs.
- Use a **grace period**: only after several consecutive failures (e.g. `failureCount >= 3`) move the domain to `disabled`. Reset `failureCount` to 0 on a successful check. This avoids disabling on transient DNS hiccups.
- Update `lastCheckedAt` / `lastError` each run.

```js
// jobs/recheckDomains.js  (run on a schedule)
const dns = require('dns').promises;
const CustomDomain = require('../models/CustomDomain');
const EDGE_IPS = (process.env.PLATFORM_EDGE_IPS || '').split(',').map(s => s.trim()).filter(Boolean);
const MAX_FAILURES = 3;

async function recheckDomains() {
  const domains = await CustomDomain.find({ status: 'active' });
  for (const d of domains) {
    const aRecords = await dns.resolve4(d.hostname).catch(() => []);
    const ok = aRecords.some(ip => EDGE_IPS.includes(ip));
    d.lastCheckedAt = new Date();
    if (ok) {
      d.failureCount = 0; d.lastError = null;
    } else {
      d.failureCount = (d.failureCount || 0) + 1;
      d.lastError = 'a_record_no_longer_points_to_platform';
      if (d.failureCount >= MAX_FAILURES) d.status = 'disabled';
    }
    await d.save();
  }
}

module.exports = { recheckDomains };
```

**Acceptance criteria:**
1. An `active` domain whose A records still point to us stays `active` and its `failureCount` stays 0.
2. A domain that stops pointing to us is set to `disabled` only after the configured number of consecutive failures.

---

## 7. What is NOT the developer's responsibility

These are handled by the coordinator / DevOps. The developer should be aware of them but does **not** build them:

- Building and deploying the **Caddy** edge (custom build with the DynamoDB storage feature), including the on-demand TLS configuration and the `www` → apex redirect.
- The AWS **load balancer**, the permanent **IP addresses**, and creating the **DynamoDB** table.
- Network isolation of the internal TLS-check endpoint.
- Supplying the configuration values in section 5.
- Switching the feature on in production and the closed-beta rollout.

The contract between the two halves is small and explicit: **Caddy calls the developer's TLS-check endpoint (Ticket 4) and only issues a certificate on a 200.** Everything else is independent.

---

## 8. Build order & dependencies

1. **Ticket 1** (data model) — done. Everything depends on it.
2. **Ticket 2** (resolution) — done. Required by request rendering and Ticket 4.
3. **Ticket 3** (onboarding + verification) — needed before anything can become `active`.
4. **Ticket 4** (TLS-check endpoint) — can be built in parallel with Ticket 3; both depend only on Tickets 1–2.
5. **Ticket 5** (UI) — depends on Ticket 3's endpoints.
6. **Ticket 6** (link fix) — independent; can be done any time.
7. **Ticket 7** (re-check job) — depends on Tickets 1–3; can be built last.

Nothing in this handoff affects live traffic until DevOps deploys the edge and the feature is switched on, because no customer domain can become `active` and resolve until the full path exists.
