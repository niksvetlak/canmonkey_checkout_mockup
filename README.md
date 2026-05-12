# CanMonkey Checkout Mockup

Proposed 5-step checkout flow for the Can-to-Curb subscription service. Built as a single static HTML file (`index.html`) with vanilla JS. This document outlines every integration point that needs to be wired to internal systems when building the production version.

---

## How Spencer's Current Checkout Works

Understanding the existing backend flow is critical for wiring up this mockup correctly. Each step already triggers backend operations:

| Step | Screen | What Gets Created |
|------|--------|-------------------|
| 1 | Contact Information | Customer + User record in CanMonkey · Stripe Customer |
| 2 | Service Address | — (address saved to customer) |
| 3 | Property Information | Property record in CanMonkey (linked to customer) |
| 4 | Review & Add-Ons | — (selections held in session) |
| 5 | Payment | Stripe Subscription (with property metadata) · one-time charges |

**Key implication:** By the end of Step 1, a CanMonkey customer and Stripe customer already exist. This means the service area check at Step 2 can flag the existing customer as `waitlisted` rather than creating a separate lead — no duplicate records.

---

## Step-by-Step Backend Operations

### Step 1 — Contact Information
1. Create **User + Customer** record in CanMonkey database
2. Create **Stripe Customer** (`stripe_customer_id` saved to CanMonkey customer record)
3. Save notification preferences (email / SMS) to customer record
4. If `is_business: true`, save company name and type (affects invoicing format)

---

### Step 2 — Service Address
1. **Validate address** (Google Places Autocomplete or USPS) before allowing Continue
2. **Run service area geo check** against Spencer's triangular service area checker
3. **If in area:** Save address to customer record, proceed to Step 3
4. **If out of area:**
   - Save address to customer record
   - Flag customer as `waitlisted` in CanMonkey
   - POST to n8n webhook → Monday.com Leads board with `Waitlisted` tag
   - Show waitlist confirmation screen — skip Steps 3–5 entirely
   - Do NOT create a property record

```
POST /api/service-area/check
Body: { zip: "XXXXX", address: "123 Main St", city: "...", state: "AZ" }
Response: { in_area: true | false, area_id: "phoenix-north" | null }
```

**Important:** Never show "outside service area" language. Frame it as joining a waitlist — the customer record and address are already saved for when the area launches.

**Existing n8n safety net:** The current flow (signup → team cancels → n8n checks address → Monday waitlist) should remain active as a fallback for any edge cases that bypass the frontend check.

---

### Step 3 — Property Information
1. Create **Property** record in CanMonkey linked to the customer
2. Save: waste provider, pickup day, trash can count, recycle can count, recycling frequency
3. Property ID stored in session for subscription metadata at Step 5

---

### Step 4 — Review & Add-Ons
No backend calls. All selections (plan, billing cadence, upsells) held in session until Step 5 submit.

**Plan tiers:**

| Plan | Cans | Collection Days | Quarterly | Monthly |
|------|------|----------------|-----------|---------|
| Starter | Up to 2 | 1/week | $45/mo | $54/mo |
| Premium | Up to 4 | Up to 2/week | $59/mo | $69/mo |
| Business | Up to 6 | Up to 2/week | $65/mo | $75/mo |

---

### Step 5 — Payment
1. Mount **Stripe Elements** into card fields
2. On submit:
   - Confirm payment method → create `PaymentMethod` in Stripe
   - Attach to Stripe Customer
   - Create **Stripe Subscription** with:
     - `trial_period_days: 15`
     - Property ID in metadata
     - Selected plan price ID (quarterly or monthly)
   - Charge one-time items immediately (Can Freshener, Same-Day Onboarding) as separate PaymentIntents
   - Flag Can Cleaning and OnDemand on the property record — **not charged at checkout**, billed separately after service is completed

```
POST /api/checkout/submit
Body: {
  customer_id: "cm_xxx",
  property_id: "prop_xxx",
  plan: { price_id: "price_xxx", billing_cadence: "quarterly" | "monthly" },
  upsells: {
    same_day_onboarding: { enrolled: true },
    can_freshener: { enrolled: true, quantity: 2 },
    can_cleaning: { enrolled: true },
    ondemand_waste_removal: { enrolled: true }
  },
  payment_method_id: "pm_xxx"
}
```

---

## Additional Integration Points

### Same-Day Onboarding Eligibility (Step 4)
Card is shown only if pickup day is within 3 days **and** current time < 3:00 PM. Cutoff and timezone should come from the route schedule API — do not rely on browser local time.

```
GET /api/routes/same-day-eligibility?zip=XXXXX
Response: {
  eligible: true,
  cutoff_time: "15:00",
  cutoff_timezone: "America/Phoenix",
  minutes_remaining: 87
}
```

Card description updates dynamically based on days until pickup. Card auto-removes from cart if conditions are no longer met (checked every 60s).

---

### Can Cleaning Availability (Step 4)
Currently limited to Phoenix metro. Card shows `Add` for in-market customers and `Waitlist` for others. Availability should be returned from the service area check at Step 2 so it doesn't require a separate call.

Add `can_cleaning_available: true | false` to the service area check response.

---

### Plan Pricing
Pull from Stripe Products API — do not hardcode rates. Store price IDs per plan per cadence.

---

### Trial Period
Calculate trial end date as `signup_date + 15 days`. Pass `trial_period_days: 15` to Stripe subscription. Display calculated end date on Step 4 billing summary and Step 5 submit button.

---

## Current Hardcoded Values to Replace

| Location | Current Value | Replace With |
|----------|--------------|--------------|
| Phoenix metro city list | Array in JS | Service area geo check API by ZIP |
| Same-day cutoff | `15:00` hardcoded | Route schedule API |
| Trial end date | "March 7, 2026" | `signup_date + 15 days` |
| Quarterly rate | `$45` | Stripe plan API |
| Monthly rate | `$54` | Stripe plan API |
| Billing start date | "March 7, 2026" | `signup_date + 15 days` |
