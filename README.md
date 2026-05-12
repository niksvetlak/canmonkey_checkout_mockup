# CanMonkey Checkout Mockup

Proposed 5-step checkout flow for the Can-to-Curb subscription service. Built as a single static HTML file (`index.html`) with vanilla JS. This document outlines every integration point that needs to be wired to internal systems when building the production version.

---

## Flow Overview

| Step | Screen | Key Data Collected |
|------|--------|--------------------|
| 1 | Contact Information | Name, email, phone, notification prefs, business registration |
| 2 | Service Address | Service address, billing address, property type |
| 3 | Property Information | Waste provider, pickup day, can count, recycling frequency |
| 4 | Review & Add-Ons | Upsell selections, billing cadence |
| 5 | Payment | Card/bank details, terms agreement |

---

## Backend Integration Points

### 1. Service Market Check (Step 2 — City/ZIP)
**Current behavior:** Checks the entered city against a hardcoded list of Phoenix metro cities.

**Production:** Replace with a real market lookup by ZIP code (more reliable than city name).

```
GET /api/markets/availability?zip=XXXXX
Response: {
  in_market: true,
  market_id: "phoenix-metro",
  services: {
    can_cleaning: true,   // drives Can Cleaning upsell availability
    same_day_onboarding: true
  }
}
```

**Drives:**
- Can Cleaning card: shows `Add` button if `can_cleaning: true`, `Waitlist` button if false
- Should trigger on ZIP field blur, not city name

---

### 2. Same-Day Onboarding Eligibility (Step 4)

**Current behavior:** Card is shown only if:
1. Pickup day is within 3 days from today
2. Current local time < 3:00 PM

If pickup is tomorrow, shows a live countdown. If 2–3 days out, shows a static message. Card description updates dynamically to reflect the actual selected pickup day. Card auto-hides and removes from cart if conditions are no longer met (checked every 60s).

**Production:** Cutoff time and timezone should come from the route schedule API, not be hardcoded. Do not use browser local time — derive timezone from the property's service address.

```
GET /api/routes/same-day-eligibility?zip=XXXXX
Response: {
  eligible: true,
  cutoff_time: "15:00",
  cutoff_timezone: "America/Phoenix",
  minutes_remaining: 87
}
```

---

### 3. Upsell Products

All four upsells need corresponding Stripe products/prices. One-time items should be added as separate line items; the subscription plan is the primary recurring charge.

| Upsell | Type | Price | Stripe |
|--------|------|-------|--------|
| ⚡ Same-Day Onboarding | One-time | $9.99 | One-time payment intent |
| 🌿 Can Freshener | One-time | $9.99/2-pack (qty selector) | One-time payment intent |
| 🧼 Can Cleaning | Bi-annual, billed per service | $59/cleaning | Triggered separately post-service |
| 🗑️ OnDemand Waste Removal | One-time, billed post-service | $59 first can + $25/extra | Triggered separately post-service |

**Notes:**
- Can Cleaning and OnDemand are not charged at checkout — they are flagged on the customer record and billed separately after the service is completed
- Can Freshener and Same-Day Onboarding are charged at checkout as one-time items
- Can Cleaning should only be offered if `services.can_cleaning: true` from the market check

---

### 4. Subscription Plan & Billing Cadence (Step 4)

**Current behavior:** Toggle between Quarterly ($45/mo) and Monthly ($54/mo). Quarterly is default.

**Production:** Pull plan pricing from Stripe Products API so prices don't need to be hardcoded.

```
GET /api/plans/starter
Response: {
  name: "Can-to-Curb Starter Plan",
  monthly_price_id: "price_xxx",
  quarterly_price_id: "price_xxx",
  monthly_rate: 54,
  quarterly_rate: 45,
  trial_days: 15
}
```

---

### 5. Trial Period
**Current behavior:** 15-day trial displayed throughout. "Total Due Today" is $0.00 for subscription items. Trial end date is hardcoded as "March 7, 2026."

**Production:** Calculate trial end date dynamically as `signup_date + 15 days`. Pass `trial_period_days: 15` to Stripe subscription creation. Display the actual calculated end date on Step 4 billing summary and Step 5 submit button subtext.

---

### 6. Address Validation (Step 2)
**Current behavior:** Free-text input, no validation.

**Production:** Recommend Google Places Autocomplete or USPS address validation to ensure accurate ZIP/city data before the market check runs.

---

### 7. Stripe Checkout / Payment (Step 5)

The payment form is a mockup only — no Stripe.js is wired up.

**Production implementation:**
1. On load of Step 5, create a Stripe `SetupIntent` or `PaymentIntent`
2. Mount Stripe Elements into the card fields
3. On submit: confirm the payment method, create the customer, attach the subscription with trial, and add any one-time charges (Freshener, Same-Day Onboarding) as additional payment intent line items
4. Can Cleaning and OnDemand should be saved as flags on the customer record in your database — not charged at checkout

```
POST /api/checkout
Body: {
  contact: { first_name, last_name, email, phone, notify_email, notify_sms },
  business: { is_business, company_name, company_type },
  service_address: { address1, address2, city, state, zip, country, property_type },
  billing_address: { same_as_service, ...fields },
  property: { waste_provider, pickup_day, trash_cans, recycle_cans, recycle_frequency },
  plan: { billing_cadence: "quarterly" | "monthly" },
  upsells: {
    same_day_onboarding: true,
    can_freshener: { added: true, quantity: 2 },
    can_cleaning: { waitlisted: false, enrolled: true },
    ondemand_waste_removal: { enrolled: true }
  },
  payment_method_id: "pm_xxx"
}
```

---

### 8. Business Registration (Step 1)
If `is_business: true`, save `company_name` and `company_type` to the customer record. This may affect invoicing (B2B invoice format vs. residential receipt).

---

### 9. Notification Preferences (Step 1)
Customer selects Email and/or SMS. At least one must be selected (enforced in UI). Pass to your notification system on customer creation.

---

## Current Hardcoded Values to Replace

| Location | Current Value | Replace With |
|----------|--------------|--------------|
| Phoenix metro city list | Hardcoded array in JS | Market availability API by ZIP |
| Same-day cutoff time | `14:00` hardcoded | Route schedule API |
| Trial end date | "March 7, 2026" | `signup_date + 15 days` |
| Quarterly rate | `$45` | Stripe plan API |
| Monthly rate | `$54` | Stripe plan API |
| Billing start date | "March 7, 2026" | `signup_date + 15 days` |
