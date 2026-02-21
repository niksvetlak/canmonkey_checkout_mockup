CanMonkey Checkout Flow — Proposed Redesign
Interactive mockup of the redesigned checkout experience for canmonkey.com.
Live Preview
Open index.html in any browser or view the GitHub Pages deployment.
What's New

4-step progress tracker — Contact → Address → Property → Payment
Sticky trust bar — "15-Day Risk-Free Trial", "No Charge Until Trial Ends", "Cancel Anytime" visible throughout
Notification preferences — Email/SMS chips wired to existing RouteTask backend
Business registration — Expandable section with company type dropdown
Can stepper controls — +/− buttons with plan limit enforcement (Starter = 2 cans max)
Upsell cards — Can Cleaning & On-Demand Waste Removal with toggle-to-add
Quarterly/Monthly billing toggle — Dynamic pricing updates across product table and summary
Sticky order summary strip — Fixed bottom bar on payment step showing plan + total
Security badges — SSL, Stripe, Money-Back Guarantee near card entry
Mobile-first design — Optimized for phone screens with compact sticky headers

Implementation Notes
~80% pure CSS/HTML restructuring, ~20% leverages existing backend systems. No new database fields required. See the full change request document for developer specs.
Brand

Font: DM Sans
Primary Navy: #0f172a
Blue Accent: #3182ce
Layout: Single-column, max-width 640px
