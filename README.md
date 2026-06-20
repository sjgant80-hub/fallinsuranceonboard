# ◊ FallInsuranceOnboard

**Sovereign IDD-shaped client onboarding for UK FCA-regulated insurance brokers.** Single HTML file. Client data never leaves the device. Prime 839.

Part of the **insurance-firm bundle**: [fallinsurance](https://github.com/sjgant80-hub/fallinsurance) (829) · **fallinsuranceonboard** (839) · [fallinsurancepaper](https://github.com/sjgant80-hub/fallinsurancepaper) (853) · [fallinsurancepractice](https://github.com/sjgant80-hub/fallinsurancepractice) (857).

Live: <https://sjgant80-hub.github.io/fallinsuranceonboard/>

---

## For the end user · the 30-second pitch

You're onboarding a new commercial client. Smart Search wants £2-£10 per CDD check. Capquest wants £200/month for AML. You're juggling spreadsheets, PDFs and three SaaS tabs. FallInsuranceOnboard is one HTML file that runs in your browser and handles the whole 10-step IDD-shaped flow — customer category, identity, business profile, demands & needs (IDD Art 20), AML CDD where statutorily triggered, PEP & sanctions, vulnerable customer (FG21/1), document capture with SHA-256 hashing, all chained into a 6-year audit log per FCA SYSC.

Client records, beneficial owners, PSC details, demands & needs statements, KYC documents — everything stored in your browser's IndexedDB. Never uploaded.

When the FCA's AML supervisor knocks, you hand them one JSON export. Done.

### What you actually do

1. Open the URL · the demo client (Patel Wealth Ltd) shows up so you can see how it works
2. Click **Firm** · set your FCA ref, PI insurer + expiry, IDD attestation
3. Click **+ adviser** · add yourself with FCA ref, SM&CR role, CPD hours
4. Click **+ client** · 10-step wizard captures category → identity → contact → business profile → demands & needs → CDD → PEP/sanctions → vulnerable → documents → commit
5. Each commit appends a SHA-256 chained audit entry
6. Client appears in the list · click any row to view detail, regenerate CDD certificate, re-check sanctions, soft-archive

### IDD Art 20 demands & needs

The wizard captures the structured intake for 13 product classes (commercial property, EL, PL, motor fleet, cyber, PI, D&O, landlord, home, motor, travel, health, life). Each class has its own template questions (the **D&N Library** tab is the read-only browser). The committed D&N Statement is what `fallinsurancepaper` renders alongside any policy.

### AML triggers (JMLSG Part II §16 / MLR 2017 reg.8)

CDD only triggers where statute requires:
- **Commercial entities** — always (BO discovery)
- **Life insurance** — annual premium >£1k or single premium >£2.5k
- **Aggregate GI premium** ≥£15k (EDD trigger)
- Any **PEP**
- Any **high-risk jurisdiction**

Pure GI consumer products under thresholds → no statutory CDD but ICOBS conduct still applies. The tool tells you which path you're on.

### Document capture

Drop a file in step 9 (passport, driving licence, utility, bank statement, incorporation cert, Companies House extract, PSC register, prior claims letter, survey report, accounts, SoF evidence, other). Stored as Blob in IDB. SHA-256 computed on capture. Expiry tracked per doc type (passport 10y, utility/bank 90d, etc.). Never uploaded anywhere.

### Audit chain (Mansoor P3)

Every write appends `{ i, ts, tool, adviserId, clientId, action, reasoning, configVersion, prevHash, docHash, payload }`. `docHash = sha256(prevHash + ts + action + clientId + payload)`. Tampering with any entry breaks the chain — `Verify chain` button in the audit modal proves integrity.

Retention: 6 years per FCA SYSC. Capped at 50,000 entries.

### Cross-tool mesh

Opens `BroadcastChannel('fall-insurance')` + `BroadcastChannel('fall-signal')`. Listens for `policy.*` events from siblings, broadcasts `client.created/updated/archived`, responds to `sync.request` with a snapshot. Also fires on `fall-client` for cross-bundle visibility (the IFA / law / claims bundles speak it).

### BYOK Q&A

`IDD Help` tab — 12 hard-coded T0 rules answer common questions (customer category, demands & needs, CDD trigger, PEP, sanctions, vulnerable, ICOBS, PI minimum, BO, IDD Art 17, IDD Art 20, POG). For free-form questions, add a BYOK API key (Anthropic / OpenAI / Gemini / OpenRouter) in Settings — answers cascade through your key, your bill.

### Export / import / wipe

Settings modal: export full state as JSON, import on another device, wipe device clean (double-confirm). Audit chain exports separately.

### Disclaimer

FallInsuranceOnboard is a tool. It assists with the IDD-shaped flow. It is **not** an insurer-binding system. Conduct rules + ICOBS application + final CDD sign-off remain the broker's responsibility. The FCA's audit team will read your audit chain — make sure your reasoning fields are honest.

---

## For the developer · architecture

### Stack

- **Single HTML file** · ~132KB · zero external dependencies at runtime (Google Fonts CDN-loaded, system fallback)
- **IndexedDB** (`fallinsuranceonboard.v1`) · stores: `state`, `audit`, `documents`
- **localStorage** mirror for the `state` blob (fast cold-boot)
- **BroadcastChannel** · `fall-insurance` (bundle) + `fall-signal` (estate)
- **WebCrypto** SHA-256 for audit chain + document hashing
- **No framework** · vanilla JS, no React/Vue/Angular, no build step

### Data model

See `INSURANCE-BUNDLE-SHARED-SCHEMA.md` in the parent estate doctrine for the full Client / Adviser / Firm record shapes. Key invariants:
- `Client.entityType` ∈ `{individual, entity}`
- `Client.customerCategory` ∈ `{consumer, commercial, large-risk}` (IDD §17(2))
- `Client.kyc.riskGrade` ∈ `{low, medium, high}` — auto-computed from PEP / sanctions / jurisdiction / industry / source-of-funds
- `Client.kyc.documentsHeld[]` are *references* (id, type, sha256, blobRef) — the actual Blob lives in `documents` IDB store

### State management

Single `state` object, persisted on every mutation. Mutations go through:
- `saveClient(c, action, reasoning)` — also broadcasts + audits
- `saveAdviser(a, reasoning)`
- `saveFirm(f, reasoning)`
- `archiveClient(c, reasoning)` — soft-delete (sets `archivedAt`)
- `storeDocument(file, clientId, type, note)` — Blob → IDB → returns metadata stub for `kyc.documentsHeld`

### Wizard architecture

`state.ui.wizard = { clientId, step, client, fresh }` holds the in-flight onboarding. `renderWizard()` paints the current step. `collectStep(n, c)` reads form values into `c`. `nextStep()` / `prevStep()` / `jumpStep(n)` navigate. `commitWizard()` finalises (audit, broadcast, close modal).

Step 4 (business profile) auto-skipped for consumer/individual clients.

### Risk scoring (auto)

`suggestRiskGrade(c)` returns `{ grade, score, reasons }`:
- PEP +3 · Sanctions match +4 · Sanctions review +2 · Sanctions unchecked +1
- Vulnerable +1 · Nationality high-risk +3 · Residence high-risk +3
- Large-risk customer +1 · Entity w/o Companies House no +1
- Director sanctions match +4 · Director PEP +2
- High-scrutiny source of funds (gift/business-sale/other) +1
- Turnover >£10M +1 · High-risk industry SIC +1
- ≥5 = high · ≥2 = medium · else low

### AML trigger logic

`checkAmlTrigger(c)` returns `{ triggered, reason }`. Statute-aware: never triggers for low-premium GI consumer; always triggers for commercial entities; triggers per JMLSG/MLR thresholds for life + aggregate GI.

### Cascade pattern (T0/T3)

- **T0** · 12 hard-coded rules with FCA / MLR / JMLSG citations · zero network · 1ms latency
- **T3** · BYOK fallback · Anthropic / OpenAI / Gemini / OpenRouter (in that order) · only fires if T0 missed AND user has a key

No T1 (Ollama local) for this tool — IDD knowledge is small enough that T0 covers the common path.

### Demo seeding

`seedDemoIfEmpty()` runs on first boot if no clients/advisers/firm exist. Seeds "Demo Brokers Ltd" + "Demo Adviser" + "DEMO · Patel Wealth Ltd" (entity, commercial, 2 beneficial owners, cyber/PI/EL/PL/property demanded). Marked `isDemo: true` for easy detection. Overwriteable by editing or archiving.

### Forking

- Change `engineName` in Settings (white-label brand)
- Edit `DOC_TYPES` / `DOC_EXPIRY` for additional doc categories
- Add T0 rules to `T0_RULES` array
- Add product classes to `PRODUCT_CLASSES` + `DN_LIBRARY`

### 14-pt sovereign gate

✓ Single HTML file (`index.html` · works from `file://`)
✓ <500KB (~132KB)
✓ Sovereign · client data never leaves device
✓ IDB primary, localStorage mirror, JSON export/import
✓ KONOMI shim implicit (sovereign tier, prime 839)
✓ `fall-signal` + `fall-insurance` + `fall-client` mesh
✓ PWA manifest baked via `data:` URL
✓ Mobile-first responsive (≤640px breakpoint)
✓ T0 always works offline, T3 BYOK optional
✓ Two-audience README (this file)
✓ MIT licence
✓ `.nojekyll` for GitHub Pages
✓ Demo data seeded on empty state
✓ Audit chain on by default

### Sibling tool messaging

Listens for these `fall-insurance` events:
- `sync.request` → responds with `sync.snapshot`
- `sync.snapshot` → merges by `updatedAt`
- `client.*` / `adviser.*` / `firm.*` → upserts local copies
- `policy.*` → upserts to `state.policies` (informational only — display in client detail)

Emits:
- `client.created` / `client.updated` / `client.archived`
- `adviser.created` / `adviser.updated`
- `firm.updated`

Also bridges to `fall-client` on commit (cross-bundle visibility for IFA / law / claims tools).

### Build + deploy

```bash
# nothing to build — single file
# deploy: standard sjgant80-hub estate pattern
git init && git add -A && git commit -m "ship fallinsuranceonboard v1.0.0"
gh repo create sjgant80-hub/fallinsuranceonboard --public --source . --remote origin --push
echo '{"source":{"branch":"main","path":"/"},"build_type":"legacy"}' | gh api -X POST repos/sjgant80-hub/fallinsuranceonboard/pages --input -
gh api -X POST repos/sjgant80-hub/fallinsuranceonboard/pages/builds
```

`.nojekyll` is the deploy trick — without it Pages tries to Jekyll-process the data: URIs.

### Versions

- **1.0.0** (2026-06-18) · first ship · IDD §17(2) categories · 10-step wizard · 13 product classes · 12 T0 rules · 6yr FCA SYSC audit retention

---

## Licence

MIT · Simon Gant · part of the [sjgant80-hub](https://github.com/sjgant80-hub) sovereign estate · prime 839 · ◊·κ=1.
