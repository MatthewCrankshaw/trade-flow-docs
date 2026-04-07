# Requirements: Trade Flow

**Defined:** 2026-04-01
**Core Value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment in one simple, structured system.

## v1.7 Requirements

Requirements for Onboarding & Landing Page milestone. Each maps to roadmap phases.

### Landing Page

- [x] **LAND-01**: User can view a public landing page at the root URL without authentication
- [x] **LAND-02**: User can see a hero section with trade-specific value proposition and "Start Free Trial" CTA
- [x] **LAND-03**: User can see feature highlights showing core capabilities (jobs, quotes, scheduling)
- [x] **LAND-04**: User can see a pricing section displaying £6/month with no-card trial messaging
- [x] **LAND-05**: User can navigate to sign up or log in from the landing page
- [x] **LAND-06**: Authenticated user visiting the root URL is redirected to the dashboard

### Onboarding Wizard

- [x] **ONBD-01**: New user is required to enter their display name before accessing the app
- [x] **ONBD-02**: New user is required to enter business name and select primary trade before accessing the app
- [x] **ONBD-03**: Country defaults to UK and currency defaults to GBP during business setup (no user input required)
- [x] **ONBD-04**: Tax rates, job types, visit types, items, and quote email template are auto-created when business setup completes
- [x] **ONBD-05**: User can see progress indication during onboarding steps
- [x] **ONBD-06**: Onboarding wizard resumes at the correct step if user refreshes or returns
- [x] **ONBD-07**: Existing users with completed profile and business bypass onboarding automatically

### No-Card Trial

- [x] **TRIAL-01**: User's 30-day free trial starts automatically after business setup without entering payment details
- [x] **TRIAL-02**: User can see trial days remaining in the app header
- [x] **TRIAL-03**: User can add payment method via Stripe Billing Portal at any time during trial
- [x] **TRIAL-04**: Subscription auto-cancels when trial ends without a payment method on file

### Hard Paywall

- [x] **PAYWALL-01**: User with invalid subscription sees a full-screen blocking page instead of the app
- [x] **PAYWALL-02**: User sees differentiated messaging based on subscription state (trial expired, payment failed, canceled)
- [x] **PAYWALL-03**: User can access Stripe Billing Portal from the paywall screen to manage subscription
- [x] **PAYWALL-04**: User sees "your data is safe" messaging on the paywall screen
- [x] **PAYWALL-05**: Support role users bypass the hard paywall entirely
- [x] **PAYWALL-06**: Existing soft paywall modal and write-action gating are removed

### Welcome Dashboard

- [x] **DASH-01**: New user sees a personalised welcome message on first dashboard visit
- [x] **DASH-02**: Dashboard shows getting-started widget with "Create your first job" and "Send your first quote" checklist items
- [x] **DASH-03**: Getting-started checklist items link to relevant pages and show completion state

## Future Requirements

Deferred to future release. Tracked but not in current roadmap.

### Trial Nudges

- **NUDGE-01**: User receives in-app contextual prompts when trial is nearing end (7, 3, 1 day)
- **NUDGE-02**: User receives email notification when trial is about to end
- **NUDGE-03**: Trial chip shows urgency visual states as trial nears end

### Landing Page Enhancements

- **LANDE-01**: Landing page displays customer testimonials and social proof
- **LANDE-02**: Landing page has trade-specific variations per industry

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Multi-page marketing site | No SEO/content strategy yet; single page sufficient for early stage |
| Product demo video | Product needs to stabilise before recording |
| Interactive product tour | Validate getting-started checklist approach first |
| A/B testing on landing page | Needs traffic volume to justify |
| Email verification gate | Adds friction at peak engagement; Firebase handles async |
| Freemium tier | Complicates subscription logic; £6/month is already very low |
| Custom Stripe Elements form | PCI scope increase; Billing Portal handles card collection |
| Onboarding step for first customer/job | Makes mandatory setup too long; handled by getting-started widget |
| Grace period after trial expiry | Complicates state machine; 30 days is generous; user can resubscribe anytime |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| LAND-01 | Phase 36 | Complete |
| LAND-02 | Phase 36 | Complete |
| LAND-03 | Phase 36 | Complete |
| LAND-04 | Phase 36 | Complete |
| LAND-05 | Phase 36 | Complete |
| LAND-06 | Phase 36 | Complete |
| ONBD-01 | Phase 37 | Complete |
| ONBD-02 | Phase 37 | Complete |
| ONBD-03 | Phase 37 | Complete |
| ONBD-04 | Phase 37 | Complete |
| ONBD-05 | Phase 37 | Complete |
| ONBD-06 | Phase 37 | Complete |
| ONBD-07 | Phase 37 | Complete |
| TRIAL-01 | Phase 35 | Complete |
| TRIAL-02 | Phase 37 | Complete |
| TRIAL-03 | Phase 37 | Complete |
| TRIAL-04 | Phase 35 | Complete |
| PAYWALL-01 | Phase 38 | Complete |
| PAYWALL-02 | Phase 38 | Complete |
| PAYWALL-03 | Phase 38 | Complete |
| PAYWALL-04 | Phase 38 | Complete |
| PAYWALL-05 | Phase 38 | Complete |
| PAYWALL-06 | Phase 38 | Complete |
| DASH-01 | Phase 39 | Complete |
| DASH-02 | Phase 39 | Complete |
| DASH-03 | Phase 39 | Complete |

**Coverage:**
- v1.7 requirements: 26 total
- Mapped to phases: 26
- Unmapped: 0

---
*Requirements defined: 2026-04-01*
*Last updated: 2026-04-02 after Phase 37 gap closure*
