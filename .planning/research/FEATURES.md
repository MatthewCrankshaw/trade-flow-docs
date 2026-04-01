# Feature Research

**Domain:** SaaS onboarding overhaul, public landing page, no-card Stripe trial, hard paywall for trade management software
**Researched:** 2026-03-31
**Confidence:** HIGH (competitor patterns well-documented, Stripe APIs verified against official docs)

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete or untrustworthy.

#### Landing Page

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Hero with clear value proposition | Every competitor (ServiceM8, Fergus, Jobber, Tradify) leads with "what it does + who it's for" headline. Tradies scanning the page need to know in 5 seconds. | LOW | Headline + subheadline + single CTA. Fergus: "Job management software built for tradies". Jobber: "Run a stronger service business". Keep it trades-specific. |
| Primary CTA to start free trial | All four competitors have prominent "Start Free Trial" / "Try Free" button above the fold. No-card messaging displayed inline. | LOW | Single CTA, not two competing buttons. "Start Free Trial - No card required" is the standard phrasing. |
| Feature highlights section | ServiceM8 shows bulleted features, Fergus uses interactive tabs (Enquiries, Quotes, Scheduling, Payments, Reporting), Jobber uses 4 benefit sections. Users need to see what they get. | LOW | 3-5 feature cards showing core capabilities (Jobs, Quotes, Scheduling, Invoicing). Screenshots or illustrations increase trust. |
| Social proof / trust signals | Fergus shows Xero/Google/G2 logos. Jobber shows "29M+ jobs completed, 50+ industries". ServiceM8 shows "$35B in jobs managed" and Xero award badge. Without social proof, trades don't trust you. | LOW | For early-stage: use specific metrics ("Built for plumbers, electricians, and builders"), ratings if available, or a single testimonial. Logos come later. |
| Mobile-responsive design | Tradies browse on phones between jobs. Fergus and Tradify emphasise mobile-first. A landing page that breaks on mobile loses the primary audience. | LOW | Already using Tailwind with mobile-first approach. Standard responsive implementation. |
| Login link in nav | Every competitor has "Log In" visible in the header. Returning users need quick access without hunting. | LOW | Existing auth system. Just needs a nav link. |

#### Onboarding Wizard

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Profile name collection | Every SaaS asks for name at signup or immediately after. Needed for personalised greetings ("Welcome, Matt") and quote/invoice sender identity. | LOW | Single form step: first name + last name. Firebase auth only captures email -- name must be stored in Trade Flow user record. |
| Business name + trade selection | ServiceM8, Tradify, Fergus, and Jobber all collect business type during setup. Trade Flow already generates defaults per trade -- this step feeds that system. | LOW | Single form step. Existing business creation endpoint handles the rest (defaults for job types, visit types, items, tax rates). |
| Auto-provisioned defaults | Trade Flow already does this (v1.0+). Trade-specific job types, visit types, items, and tax rates created on business creation. Table stakes because competitors all provide starter templates. | ALREADY DONE | No new work. Existing business creation service handles this. |
| Progress indication | SaaS onboarding best practice: users need to know how many steps remain. Reduces abandonment. Airtable's wizard approach showed 20% lift in activation. | LOW | Step indicator (1/3, 2/3, 3/3) or simple progress bar. |
| Mandatory completion before app access | This is the shift from current dismissible flow. If onboarding is skippable, users enter the app with no business configured, hitting errors everywhere. Fergus and Tradify both require setup before app use. | MEDIUM | Requires route guards and state checks. New users without completed profile+business must be redirected to onboarding. Depends on: new `onboardingComplete` flag or checking user+business existence. |

#### No-Card Free Trial

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Trial starts without payment method | ALL four competitors (ServiceM8, Fergus, Tradify, Jobber) offer 14-day no-card trials. Industry standard. Card-required trials have ~49% conversion but filter out most prospects. No-card gets 18-25% conversion but 3-5x more signups. For early-stage product, volume matters more. | MEDIUM | Stripe supports this natively. Create subscription via API with `trial_period_days=30`, no payment method attached. Key param: `trial_settings.end_behavior.missing_payment_method`. Replaces current Stripe Checkout flow for trial start. |
| Trial period visibility | Current trial chip exists (v1.6). Users need to know how many days remain. | ALREADY DONE | Existing trial chip component. May need copy tweak ("14 days left -- add payment method to continue"). |
| Add payment method later via Billing Portal | Stripe Billing Portal already integrated (v1.6). Users add card when ready, not at signup. | ALREADY DONE | Existing Settings > Billing tab with portal link. Needs prominence during trial nearing end. |
| End-of-trial handling | When trial ends with no card: subscription must cancel or pause. User must be blocked from write actions (hard paywall kicks in). | MEDIUM | Stripe `trial_settings.end_behavior.missing_payment_method=cancel` is simplest. Webhook `customer.subscription.deleted` already handled (v1.6). Existing subscription sync should handle this. |

#### Hard Paywall

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Full-screen blocking when subscription invalid | Current soft paywall only blocks write actions with a modal. A hard paywall replaces the app content entirely when subscription is expired/canceled/missing. This is standard for paid SaaS -- Jobber, Tradify, Fergus all lock you out fully when subscription lapses. | MEDIUM | Full-screen overlay or redirect to a "reactivate" page. Must show: what happened (trial ended / payment failed / canceled), what to do (add card / update payment / resubscribe), and a way to access billing portal. Depends on: existing subscription state from Redux store. |
| Billing Portal access from paywall | User locked behind paywall must be able to fix their subscription without support tickets. | LOW | Button linking to Stripe Billing Portal. Existing portal session creation endpoint works. |
| Support bypass preserved | Current support role bypass (v1.6) must continue working. | LOW | Existing `SubscriptionGuard` already handles this. Just extend to new paywall component. |
| Data preservation messaging | Users with expired trials need assurance their data isn't deleted. Reduces panic and support requests. | LOW | Copy: "Your data is safe. Add a payment method to continue where you left off." |

### Differentiators (Competitive Advantage)

Features that set Trade Flow apart. Not required, but valuable.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Welcome dashboard with personalised greeting | "Good morning, Matt" with first-steps guidance creates warmth competitors lack. Jobber and ServiceM8 dump you into a feature-heavy dashboard. A simple welcome state for new users reduces overwhelm. | LOW | Conditional rendering on dashboard: if new user with no jobs, show welcome card with greeting + getting-started checklist instead of empty charts. |
| Getting-started checklist widget | Notion-style checklists (create job, create quote) raised onboarding completion to 60% with 40% retention bump at 30 days. Existing onboarding widget pattern can be repurposed. | LOW | Reuse existing widget pattern from current dashboard. Two items: "Create your first job", "Send your first quote". Checkmarks persist via existing mechanisms. |
| Trade-specific landing page copy | Fergus has industry-specific cards (Plumbers, Electricians, HVAC, Builders). Jobber serves 50+ industries. Trade Flow can speak directly to specific trades rather than generic "field service" language. | LOW | Copy variation, not code. Hero section mentions "plumbers, electricians, builders" specifically. Resonates with target audience who distrust generic software. |
| Single pricing displayed on landing page | Fergus shows 2 plans, Jobber shows 3. Trade Flow has one plan at GBP 6/month -- radically simple. No decision paralysis. This simplicity is a selling point vs competitors charging GBP 30-75/month. | LOW | Single pricing card on landing page. "GBP 6/month after your free trial. That's it." Undercuts every competitor by 5-12x. |
| Instant time-to-value (under 2 minutes to app) | Best practice: under 2 minutes to first perceived value. With only 2-3 mandatory onboarding steps (profile + business + trial activation), user lands in the app in under 60 seconds. Competitors require more setup. | LOW | Minimal mandatory steps. No email verification gate, no feature tour, no tutorial before access. Get them into the app fast. |
| Contextual trial-ending nudges | Instead of just a chip, show contextual prompts when trial is nearing end (7 days, 3 days, 1 day). "Add a payment method to keep your data" messaging in-context drives conversion better than email alone. | LOW | Enhanced trial chip with urgency states. Could link directly to billing portal. Low effort, high conversion impact. |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem good but create problems for this specific milestone.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Interactive product tour / guided walkthrough | SaaS onboarding guides recommend it. Airtable, Notion do it. | Requires tooltip/spotlight library, per-feature tour scripts, maintenance as UI changes. Overkill when the product has 3 core screens. Trades people learn by doing, not by watching. | Getting-started checklist that links to relevant pages. Let users explore naturally. |
| Email verification gate before app access | "Best practice" for reducing fake signups. | Adds friction at the exact moment user is most engaged. Firebase handles email verification asynchronously. Trade Flow is B2B SaaS, not a social network -- fake signups are not a pressing problem at this stage. | Let Firebase send verification email in background. Do not gate access on it. |
| Multi-page marketing site | Competitors have 20+ pages (features, industries, resources, blog). Looks more "professional". | Massive content effort. Single-page landing with clear sections converts just as well for early-stage products. Trade Flow does not need SEO-driven content marketing yet. | Single-page landing with hero, features, pricing, CTA sections. Add pages later when content exists. |
| Animated demos / product videos on landing page | ServiceM8 and Jobber use video. Looks polished. | Requires video production, hosting, and slows page load. Screenshots or static illustrations are sufficient and faster to produce. Can add video later. | Static screenshots or device mockups showing the actual product UI. |
| Freemium tier / feature-limited free plan | "Let users try forever for free." Sounds generous. | Complicates subscription logic. Creates support burden from non-paying users. Trade Flow's GBP 6/month is already extremely low -- freemium adds complexity for minimal conversion benefit. | 30-day full-access free trial with hard paywall. Clean, simple, industry standard. |
| Custom Stripe Elements payment form | More "integrated" feel than redirecting to Stripe. | PCI compliance scope increases. SCA/3DS handling becomes your problem. Current Stripe Checkout/Portal already handles all of this. No benefit for the complexity cost. | Continue using Stripe Billing Portal for payment method collection. Already integrated (v1.6). |
| Onboarding step for adding first customer/job | "Guide them through the whole workflow." | Makes mandatory onboarding too long. Best practice says under 2 minutes. Customer/job creation is what the getting-started checklist handles -- optional, in-app, at user's pace. | Getting-started widget on dashboard (create job, send quote). |
| Grace period after trial expiry | "Give them a few extra days to decide." | Complicates subscription state machine. Users who have not converted in 30 days rarely convert in 33. Stripe's cancel-at-trial-end is clean. | Hard paywall immediately at trial end. User can resubscribe at any time -- data preserved. |

## Feature Dependencies

```
[Public Landing Page]
    (independent -- no auth, no app state)

[Profile Setup Step]
    └──requires──> [User record in MongoDB (existing)]

[Business Setup Step]
    └──requires──> [Profile Setup Step]
    └──triggers──> [Auto-create defaults (existing)]

[No-Card Trial Activation]
    └──requires──> [Business Setup Step (Stripe customer needs business context)]
    └──requires──> [Stripe Customer creation (existing)]
    └──modifies──> [Subscription creation flow (replaces Checkout)]

[Mandatory Onboarding Guards]
    └──requires──> [Profile Setup Step]
    └──requires──> [Business Setup Step]
    └──requires──> [Onboarding completion state tracking]

[Hard Paywall]
    └──requires──> [Subscription state in Redux store (existing)]
    └──replaces──> [Soft paywall modal (existing)]
    └──requires──> [Billing Portal access (existing)]

[Welcome Dashboard]
    └──requires──> [Profile name (from Profile Setup)]
    └──requires──> [Business exists (from Business Setup)]
    └──enhances──> [Existing dashboard page]

[Getting-Started Widget]
    └──requires──> [Welcome Dashboard]
    └──reuses──> [Existing onboarding widget pattern]
```

### Dependency Notes

- **Business Setup requires Profile Setup:** Business must be associated with a named user. Profile name feeds into quote sender identity and dashboard greeting.
- **No-Card Trial requires Business:** Stripe subscription needs a customer ID. Current flow creates Stripe customer during business creation. Trial activation should happen automatically after business setup completes.
- **Hard Paywall replaces Soft Paywall:** Not additive -- the soft paywall modal (v1.6) should be removed and replaced with full-screen blocking. Both cannot coexist without confusion.
- **Landing Page is independent:** No dependencies on auth or app state. Can be built and deployed independently of other features.
- **Welcome Dashboard enhances existing:** Conditional rendering on the existing dashboard, not a new page. Shows welcome state for users with no jobs.

## MVP Definition

### Launch With (v1.7)

Minimum viable set -- what is needed to replace current onboarding and enforce hard paywall.

- [ ] Public landing page (hero, features, pricing, CTA) -- first touchpoint for new users
- [ ] Profile setup step (first name, last name) -- mandatory, feeds greeting and identity
- [ ] Business setup step (business name, primary trade) -- mandatory, triggers default creation
- [ ] No-card Stripe trial activation -- automatic after business setup, 30-day trial, no payment method
- [ ] Mandatory onboarding route guards -- redirect incomplete users to onboarding flow
- [ ] Hard paywall screen -- full-screen block when subscription invalid, with billing portal access
- [ ] Welcome dashboard with greeting -- personalised greeting with getting-started checklist
- [ ] Getting-started widget -- create job + send quote checklist items
- [ ] Remove existing dismissible onboarding flow -- clean up old code

### Add After Validation (v1.x)

Features to add once core onboarding and paywall are working.

- [ ] Trial-ending email nudges (7 day, 3 day, 1 day) -- trigger when `customer.subscription.trial_will_end` webhook fires
- [ ] Enhanced trial chip with urgency states -- visual escalation as trial nears end
- [ ] Landing page testimonials / social proof -- add once real user quotes are available
- [ ] Trade-specific landing page variations -- separate sections or routes per trade

### Future Consideration (v2+)

Features to defer until product-market fit is established.

- [ ] Multi-page marketing site -- only when SEO/content strategy exists
- [ ] Product demo video -- only when product is stable enough to record
- [ ] Interactive product tour -- only if checklist approach shows low activation
- [ ] A/B testing on landing page -- only when traffic volume justifies it

## Feature Prioritisation Matrix

| Feature | User Value | Implementation Cost | Priority | Depends On Existing |
|---------|------------|---------------------|----------|---------------------|
| Public landing page | HIGH | LOW | P1 | None (new route, no auth) |
| Mandatory profile setup | HIGH | LOW | P1 | User MongoDB record |
| Mandatory business setup | HIGH | LOW | P1 | Profile setup, existing business creation |
| No-card trial activation | HIGH | MEDIUM | P1 | Business setup, Stripe customer, new API flow |
| Mandatory onboarding guards | HIGH | MEDIUM | P1 | Profile + business completion state |
| Hard paywall screen | HIGH | MEDIUM | P1 | Subscription state (existing Redux store) |
| Welcome dashboard greeting | MEDIUM | LOW | P1 | Profile name, business exists |
| Getting-started widget | MEDIUM | LOW | P1 | Existing widget pattern |
| Remove old onboarding flow | MEDIUM | LOW | P1 | New onboarding complete |
| Contextual trial-ending nudges | MEDIUM | LOW | P2 | Trial chip (existing) |
| Trial-ending emails | MEDIUM | MEDIUM | P2 | Email infrastructure (existing), webhook event |
| Landing page social proof | LOW | LOW | P3 | Real user testimonials |

**Priority key:**
- P1: Must have for v1.7 launch
- P2: Should have, add in subsequent iteration
- P3: Nice to have, future consideration

## Competitor Feature Analysis

### Landing Pages

| Feature | ServiceM8 | Tradify | Fergus | Jobber | Trade Flow Approach |
|---------|-----------|---------|--------|--------|---------------------|
| Hero headline | "Smart software for contractors & services" | "Job Management Software for Trades" | "Job management software built for tradies" | "Run a stronger service business" | Trade-specific: "Run your trade business from first call to final payment" |
| Primary CTA | "Free Trial" button | "Try Free" | Multi-step trial form | "Start Free Trial" | "Start Free Trial - No card required" |
| Social proof | "$35B in jobs managed", Xero award | Customer logos | Xero/Google/G2 logos, testimonials | "29M+ jobs", app store ratings, testimonials | Metrics when available, trade-specific trust signals |
| Pricing display | 5 tiers (Free to Premium Plus) | Single page link | 2 plans (GBP 53-75/month) | 3 plans ($39-599/month) | Single plan: GBP 6/month. Radical simplicity. |
| Feature sections | Bulleted list | Feature pages | Interactive tabs (5 categories) | 4 benefit sections with testimonials | 3-5 feature cards with screenshots |
| Industry targeting | Generic "contractors & services" | Trade-specific pages per industry | Industry cards (Plumbers, Electricians, etc.) | 50+ industry pages | Mention 3-4 core trades in hero copy. Industry pages deferred. |

### Onboarding

| Feature | ServiceM8 | Tradify | Fergus | Jobber | Trade Flow Approach |
|---------|-----------|---------|--------|--------|---------------------|
| Trial length | 14 days, no card | 14 days, no card | 14 days, no card | 14 days, no card | 30 days, no card. Longer trial = more time to build habit. |
| Signup friction | Email only | Email only | Email + reCAPTCHA | Email + business type | Email (Firebase) then profile + business in-app |
| Setup guidance | Videos, articles, partner support | Free 1-on-1 onboarding session | Guided Job Card Tour with example data | Onboarding specialists, intuitive setup | Mandatory 2-step wizard + getting-started checklist |
| Time to app access | Minutes (self-serve) | Minutes (self-serve) | Minutes (self-serve) | Minutes (self-serve) | Under 60 seconds (2 form steps only) |
| Post-setup guidance | Help centre, videos | Phone support, training | Product walkthroughs | Support team | Getting-started checklist on dashboard |

### Paywall / Subscription

| Feature | ServiceM8 | Tradify | Fergus | Jobber | Trade Flow Approach |
|---------|-----------|---------|--------|--------|---------------------|
| Post-trial behaviour | Locks out | Locks out | Locks out | Locks out | Hard paywall (full-screen block) |
| Price point | Free tier + paid from ~$9/month | ~GBP 29/user/month | GBP 53-75/month | $39-599/month | GBP 6/month. Significant undercut. |
| Payment method collection | During/after trial | After trial | After trial | After trial | Stripe Billing Portal (add when ready) |
| Data on expiry | Preserved | Preserved | Preserved | Preserved | Preserved. Messaging: "Your data is safe." |

## Key Implementation Notes

### Stripe No-Card Trial API (verified against official docs)

The existing Stripe Checkout flow (v1.6) creates trials with card required. The new flow replaces this:

1. **Create subscription directly via API** (not Checkout) with `trial_period_days: 30` and no payment method
2. Set `trial_settings.end_behavior.missing_payment_method: 'cancel'` -- subscription auto-cancels if no card added by trial end
3. Set `payment_settings.save_default_payment_method: 'on_subscription'` -- when user adds card via Billing Portal, it attaches to subscription
4. Webhook `customer.subscription.trial_will_end` fires 3 days before trial ends -- use for nudge notifications
5. Webhook `customer.subscription.deleted` fires when trial ends without card -- triggers hard paywall

**Existing webhook handlers (v1.6) already process `customer.subscription.created`, `customer.subscription.updated`, and `customer.subscription.deleted`** -- the no-card flow generates the same events. Main change is subscription creation moves from Checkout redirect to server-side API call during onboarding.

### Landing Page Routing

The landing page must be the root route (`/`) for unauthenticated users. Authenticated users hitting `/` should redirect to `/dashboard`. This is a routing concern, not a new app -- the landing page lives in the same React app but renders outside the authenticated layout.

### Onboarding State Machine

The onboarding flow is a linear progression: Profile -> Business -> Trial -> Dashboard. State can be derived from data existence (has user name? has business? has subscription?) rather than a separate flag. This avoids a new onboarding state field and leverages existing data.

## Sources

- [ServiceM8 Landing Page](https://www.servicem8.com) -- live site analysis, 2026-03-31
- [ServiceM8 Free Trial](https://www.servicem8.com/register) -- registration page
- [Tradify Pricing](https://www.tradifyhq.com/pricing) -- 14-day no-card trial
- [Tradify Signup](https://go.tradifyhq.com/signup/) -- registration flow
- [Fergus Landing Page](https://fergus.com/) -- live site analysis, 2026-03-31
- [Fergus Signup](https://fergus.com/signup/) -- registration page
- [Jobber Landing Page](https://www.getjobber.com) -- live site analysis, 2026-03-31
- [Jobber Pricing](https://www.getjobber.com/pricing/) -- plan comparison
- [Stripe Free Trials Documentation](https://docs.stripe.com/billing/subscriptions/trials/free-trials) -- official API docs for no-card trials (HIGH confidence)
- [Stripe Trial Configuration](https://docs.stripe.com/billing/subscriptions/trials) -- trial_settings.end_behavior parameter
- [SaaS Free Trial Conversion Statistics 2025](https://www.amraandelma.com/free-trial-conversion-statistics/) -- 18-25% opt-in vs ~49% opt-out conversion rates
- [SaaS Onboarding Best Practices 2025](https://productled.com/blog/5-best-practices-for-better-saas-user-onboarding) -- checklist patterns, time-to-value benchmarks
- [Hard Paywall vs Soft Paywall](https://www.revenuecat.com/blog/growth/hard-paywall-vs-soft-paywall/) -- RevenueCat analysis, conversion trade-offs
- [Airtable Onboarding Wizard](https://www.candu.ai/blog/airtables-best-wizard-onboarding-flow) -- 20% activation lift from wizard approach

---
*Feature research for: SaaS onboarding, landing page, no-card trial, hard paywall (trade management domain)*
*Researched: 2026-03-31*
