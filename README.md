# Stripe billing — Edge Functions setup

Three Supabase Edge Functions make `billing.html` work:

- **create-checkout-session** — starts a Stripe subscription checkout for the plan the user picked
- **create-portal-session** — opens Stripe's hosted Billing Portal so a subscriber can update their card, cancel, or see invoices
- **stripe-webhook** — the only thing that ever writes to the `subscriptions` table or `profiles.plan`. Stripe calls this directly; nothing on the frontend does.

## 1. Stripe dashboard setup

1. **Create two Products**, each with one recurring Price:
   - "Professional" — $149/month
   - "Team" — $499/month

   (Enterprise stays a "Contact sales" mailto link, not self-serve Stripe checkout — same as the marketing pricing page today.)

2. Copy each **Price ID** (starts with `price_`, not the Product ID which starts with `prod_`).

3. **Developers → API keys** — copy your **Secret key** (starts with `sk_`). Use a *test mode* key while building this out; switch to live mode only once you've tested the full flow end to end.

## 2. Install the Supabase CLI and link your project

```bash
npm install -g supabase
supabase login
supabase link --project-ref YOUR-PROJECT-REF
```

## 3. Set secrets (these become `Deno.env.get(...)` inside the functions)

```bash
supabase secrets set STRIPE_SECRET_KEY=sk_test_...
supabase secrets set STRIPE_PRICE_PROFESSIONAL=price_...
supabase secrets set STRIPE_PRICE_TEAM=price_...
supabase secrets set SITE_URL=https://your-username.github.io/your-repo
```

`SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are injected automatically by the Edge Functions runtime — you don't set those yourself. `STRIPE_WEBHOOK_SECRET` gets set in step 5, after Stripe gives you one.

## 4. Deploy the functions

```bash
supabase functions deploy create-checkout-session
supabase functions deploy create-portal-session
supabase functions deploy stripe-webhook --no-verify-jwt
```

The `--no-verify-jwt` flag on the webhook is required: Stripe can't send a Supabase auth header, so this function has to skip Supabase's normal JWT check. The webhook is still secure — Stripe's signature is verified inside the function itself, using `STRIPE_WEBHOOK_SECRET`.

## 5. Point Stripe at your webhook

1. Copy the deployed URL, something like:
   `https://YOUR-PROJECT-REF.supabase.co/functions/v1/stripe-webhook`
2. Stripe Dashboard → **Developers → Webhooks → Add endpoint**, paste that URL
3. Subscribe to at least these events:
   - `checkout.session.completed`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
   - `invoice.payment_failed`
4. Copy the **Signing secret** Stripe gives you (starts with `whsec_`) and set it:
   ```bash
   supabase secrets set STRIPE_WEBHOOK_SECRET=whsec_...
   ```

## 6. Test it

Stripe's test mode gives you fake card numbers — `4242 4242 4242 4242`, any future expiry, any CVC — that trigger a real (test) checkout without charging anything. Use the [Stripe CLI](https://stripe.com/docs/stripe-cli) to forward webhook events to your deployed function while testing:

```bash
stripe listen --forward-to https://YOUR-PROJECT-REF.supabase.co/functions/v1/stripe-webhook
```

Then: sign up for an account through `setup.html`, open `billing.html`, click "Upgrade to Professional," complete checkout with the test card, and confirm you land back on `billing.html?checkout=success` with your plan updated. Check `supabase functions logs stripe-webhook` if it doesn't update — that's where errors from the webhook handler show up.

## Known limitation

`invoice.payment_failed` marks the subscription `past_due` in the database but doesn't currently notify the user (no email, no in-app alert). Wiring that into the existing `alert_prefs` system would be a reasonable next step — flagged here rather than built, since it overlaps with how vendor-change alerts already work and is worth doing consistently with that system rather than as a one-off.
