# Exit Planner — Project Spec

> **Single source of truth for this product.** When starting a new Cursor/Claude chat, first instruction is always: "Read PROJECT.md before doing anything." Update this file whenever scope changes.

---

## 1. What this is

A web app that helps small landlords in Washington State model the after-tax outcome of selling a single-family rental property across multiple exit strategies. Inputs are the property basics and the landlord's tax context. Output is a side-by-side comparison of 5 strategies with year-1 net cash, year-1 tax owed, and a 10-year wealth projection.

The product is positioned as **"the document you bring to your CPA"** — not a CPA replacement. This framing is both the go-to-market wedge and the liability shield.

## 2. Who it's for

Primary persona: **Small landlord in Washington State, 1–10 SFR rental properties, 10+ year hold, considering selling in the next 12 months.** Typically 45–65 years old. Uses a CPA for taxes but doesn't get proactive planning. Has heard of 1031 exchanges but doesn't know the full menu of exit options. Comfortable with a web form, not with a spreadsheet.

Secondary (v1.1+): CPAs and fee-only financial planners who want a quick scenario tool to use with clients.

## 3. Why this exists (the gap)

- Deal-analysis tools (BiggerPockets, DealCheck, Roofstock) optimize for **acquisition**, not exit. Their "exit" logic is tweaking sale price.
- Portfolio/bookkeeping tools (Stessa, Baselane, Azibo, Hemlane) handle **operations**, not sale planning.
- Single-strategy calculators (1031 accommodators, DST brokers, cost-seg firms) are **lead magnets biased toward their own product**. None of them will tell a user "for you, don't do the 1031."
- CPA-facing tax-planning software (Holistiplan, Corvee) is $1–3K/month and advisor-workflow, not landlord-facing.

**Whitespace:** an unbiased, landlord-facing, multi-strategy exit comparison with WA-specific tax rules, passive-loss handling, and conversational what-if exploration.

## 4. Scope — MVP v0

**In scope:**
- Washington State only
- Single-family rentals only
- One property at a time (no portfolio view)
- 5 strategies: outright sale, traditional 1031 exchange, DST 1031, installment sale, §121 primary-residence conversion
- Side-by-side results grid with drill-down per strategy
- Natural-language "what-if" chat (Claude API) that patches inputs and re-runs deterministic engine
- PDF export styled as "CPA handoff document"
- localStorage persistence only — no user accounts, no backend storage of tax data

**Explicitly out of scope for v0:**
- User accounts / auth
- Multi-property portfolios
- States other than WA
- Property types other than SFR rentals (no multifamily, STR, commercial, land, MHP)
- Lazy Man's 1031 strategy (requires cost seg modeling — save for v1.1)
- Hold-to-step-up estate planning strategy
- Zillow / MLS / ATTOM data integration (user enters current value manually)
- Advisor/CPA accounts or white-label
- Payments (free during MVP validation; add Stripe after ~50 users)
- Email capture / marketing automation
- SEO content / blog

## 5. The 5 strategies modeled

1. **Outright sale** — baseline. Federal LTCG + §1250 depreciation recapture (capped 25%) + NIIT (3.8% if applicable) + WA 7% capital gains tax (on LTCG above current threshold) + selling costs. Suspended passive losses released on full disposition.
2. **Traditional 1031 exchange** — defers all federal gain + recapture + NIIT + WA if fully reinvested into like-kind with equal or greater value and debt replaced. Model boot scenarios when target replacement value < relinquished value.
3. **DST 1031** — same deferral mechanics as trad 1031 but passive. Flag trade-offs: illiquidity, sponsor fees, no control.
4. **Installment sale (seller financing)** — spreads LTCG across years per IRC §453, but **depreciation recapture is fully taxed in year of sale** per §453(i). This is the common misconception to surface.
5. **§121 primary-residence conversion** — move in for 2 of last 5 years, exclude up to $250K single / $500K MFJ of gain. **Depreciation recapture is still taxed** (another common misconception). Model 2-year and 3-year conversion timelines.

Each strategy produces a `ScenarioResult` (see data model) with year-1 numbers, 10-year projection, assumptions, and caveats.

## 6. User journey

1. Land on `/` → click "Start my analysis"
2. `/analyze` — 3-step form (property, tax context, intent), ~7 minutes to complete
3. `/results` — side-by-side grid of 5 strategies, sortable
4. Click any card → expanded tax-breakdown view
5. Click "Ask a what-if" → chat modal, e.g. "what if my spouse qualifies as a real estate professional next year?"
6. Click "Export PDF" → downloadable CPA-handoff document

## 7. Screens (v0)

- `/` — landing page
- `/analyze` — 3-step input form with progress indicator
- `/results` — 5-card comparison grid, sortable, with expandable drill-down and what-if chat
- PDF export (not a screen, a download action from `/results`)

No other screens in v0. No dashboards, no saved scenarios, no account pages.

## 8. Data model

```ts
type PropertyInput = {
  purchasePrice: number;
  purchaseDate: string;          // ISO
  currentValue: number;          // user-entered for v0
  loanBalance: number;
  capitalImprovements: number;
  depreciationTaken?: number;    // helper auto-computes if blank
  state: 'WA';                   // hardcoded v0
  propertyType: 'SFR_RENTAL';    // hardcoded v0
};

type TaxContext = {
  filingStatus: 'single' | 'mfj' | 'mfs' | 'hoh';
  estimatedAGI: number;
  suspendedPAL: number;          // Form 8582 carryforward
  capitalLossCarryover: number;
  ownerAge: number;
};

type Intent = {
  wantsToStayInvested: boolean;
  needsCashNow: boolean;
  willingToMoveIn: boolean;
  timelineMonths: number;
};

type StrategyKey =
  | 'outright_sale'
  | 'trad_1031'
  | 'dst_1031'
  | 'installment_sale'
  | 'section_121';

type ScenarioResult = {
  strategy: StrategyKey;
  year1: {
    grossProceeds: number;
    federalCapGains: number;
    depreciationRecapture: number;   // §1250, max 25% federal
    niit: number;                    // 3.8%
    waCapGainsTax: number;           // 7% over current threshold
    passiveLossOffset: number;
    sellingCosts: number;
    netCashInHand: number;
  };
  projection: { year: number; wealth: number }[];
  assumptions: string[];
  caveats: string[];
};
```

## 9. Tech stack

- **Next.js 15** (App Router) + **TypeScript**
- **Tailwind CSS** + **shadcn/ui** (Button, Card, Input, Select, Tooltip, Dialog, Progress at minimum)
- **Zod** for input validation
- **Recharts** for the 10-year wealth projection chart
- **Vercel AI SDK** + **Claude API** (`claude-sonnet-4-5`) for what-if chat
- **@react-pdf/renderer** for client-side PDF generation
- **localStorage** for v0 persistence — no backend storage of user data
- **Vercel** hosting (free tier)
- **Vitest** for tax-engine unit tests

No database, no auth provider, no analytics, no error monitoring in v0. Add when needed.

## 10. Architecture

/app
/page.tsx                       landing
/analyze/page.tsx               input form (3 steps)
/results/[id]/page.tsx          comparison grid
/api/chat/route.ts              Claude streaming endpoint for what-if
/lib
/tax-engine
constants.ts                  brackets, thresholds, rates (current tax year)
basis.ts                      adjusted basis, depreciation helpers
outright-sale.ts
trad-1031.ts
dst-1031.ts
installment-sale.ts
section-121.ts
index.ts                      runScenario(inputs, strategy)
/schemas                        zod schemas
/storage                        localStorage read/write helpers
/components
/ui                             shadcn primitives
/landing                        landing-page sections
/form                           form steps
/results                        scenario cards, drill-downs, what-if chat
/test
tax-engine.test.ts              worked examples, CPA-reviewed

## 11. Tax-engine correctness rules

This is core IP and core liability. Every rule must have a unit test with a worked example. A CPA will review test cases before launch.

- Adjusted basis = purchase + capital improvements − depreciation taken
- §1250 depreciation recapture = lesser of (total depreciation) or (total gain), taxed at up to 25% federal
- Remaining LTCG taxed at 0/15/20% by AGI bracket
- NIIT 3.8% if AGI > $200K single / $250K MFJ, applies to cap gains + recapture
- WA capital gains tax 7% on LTCG above current-year threshold (verify annually; constant in `constants.ts`)
- Suspended passive losses release on full disposition — can offset gain. This is the rule most accommodator calculators ignore.
- §121: $250K / $500K exclusion requires 2 of 5 years primary use. Depreciation recapture is still taxed.
- Installment sale: recapture fully taxed in year of sale per §453(i). Only capital-gain portion spreads.

Every strategy result must include a caveats array listing what the model does NOT account for (AMT, QBI interaction, prior 1031 chains, state NIIT in non-WA states, etc.).

## 12. Working rules for AI-assisted development

- Prefer clarity over cleverness. Comment the "why," not the "what."
- Ask before assuming. If a spec is ambiguous, surface the question; don't silently pick.
- Change only what's necessary. No drive-by refactors.
- After any code change, list the files touched and one sentence per change.
- Never introduce a new dependency without flagging it first.
- Never add backend storage, auth, or analytics without explicit approval — v0 is localStorage-only.
- Keep the tax engine pure: no I/O, no side effects, fully unit-testable.
- Match existing code style. Don't rewrite working code to a different pattern.

## 13. Roadmap

**Week 1:** ✅ Next.js deployed to Vercel.
**Week 2:** Landing page + 3-step form + results grid with stubbed tax engine returning hardcoded numbers.
**Week 3:** Real tax engine for all 5 strategies with unit tests. CPA review of test cases.
**Week 4:** What-if LLM chat + PDF export + first 10 real users from r/realestateinvesting and PNW landlord groups.
**Week 5+:** Iterate based on user feedback. Consider Stripe, domain purchase, and v1.1 scope only after reaching ~50 real users.

## 14. Success criteria for v0

- 50 completed analyses from real WA landlords
- At least 10 users who say the output influenced a real decision
- At least 3 CPAs who say they'd use or recommend the tool
- Zero tax-math errors flagged by users or reviewing CPA

If those are hit, expand to CA + OR next. If not, talk to users and fix before adding scope.

## 15. Disclaimer (must appear on every screen)

"For informational purposes only. Not tax, legal, or investment advice. Verify all results with a licensed CPA or tax attorney before acting."
