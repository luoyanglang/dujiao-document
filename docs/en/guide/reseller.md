---
outline: deep
---

# Reseller System

> Updated: 2026-06-23

The Reseller system is Dujiao-Next's built-in **white-label, multi-tenant distribution model**. A reseller runs their own standalone storefront (a dedicated domain with custom branding) and sells your products at a self-set markup on top of the base price — **the difference between the reseller price and the base price is the reseller's profit**. Buyers place and pay for orders on the reseller's storefront, while the main site fulfills them centrally. Reseller profit is settled to the reseller's account after a confirmation period, and the reseller can request withdrawals.

::: tip Difference from the Affiliate Program
This system and the [Affiliate Program](./affiliate) are two independent features — do not confuse them:

| Dimension | Affiliate | Reseller (this doc) |
|-----------|-----------|---------------------|
| Model | Referral links + commission rebate | White-label storefront + self-set pricing margin |
| Storefront | Shares the main site | Dedicated domain, independent branding |
| Buyer perception | Still shopping on the main site | Shopping on the reseller's branded site |
| Revenue source | Order amount × commission rate | Reseller price − base price (margin profit) |
| Best suited for | Individual promotion / traffic | Channel agents / second-tier merchants |
:::

---

## 1. Feature Overview

| Module | Description |
|--------|-------------|
| Application & Review | Users apply to become resellers; admins review and set markup limits |
| Domain Binding | System subdomain (assigned by admin) or custom domain (reviewed by admin) |
| White-Label Site | Custom site name, logo, favicon, announcement, SEO, navigation, footer, theme |
| Products & Pricing | List/unlist products, 4 pricing modes for self-set markup |
| Order Fulfillment | Buyers order on the reseller site; the main site fulfills centrally |
| Profit Settlement | Profit posting → confirmation period → withdrawable, ledgered per currency |
| Withdrawal | Resellers request withdrawals; admins review and pay out offline |
| Risk Control | Self-dealing / related-account profit blocking; pricing floor prevents platform loss |

---

## 2. Enabling Reseller Mode

Reseller mode is disabled by default. Enable it in the `reseller` section of the backend `config.yml`:

```yaml
reseller:
  enabled: false
  # List of main-site hosts. After enabling reseller mode, put the production main-site domains here.
  # Requests matching main_hosts are handled as the main-site context; otherwise reseller domains are resolved.
  main_hosts:
    - localhost
    - 127.0.0.1
    - "::1"
  # Enable only when the reverse proxy is trusted and forwards Host correctly.
  # false uses the raw Host; true prefers X-Forwarded-Host.
  trusted_forwarded_host: false
  # Base domain used when an admin assigns a system subdomain, e.g. hello.{subdomain_base}.
  # Before enabling in production, set it to a domain with wildcard DNS/TLS ready, e.g. shop.example.com.
  subdomain_base: ""
  # Whether ordinary users may self-submit a reseller application from the personal center.
  self_apply_enabled: true
  # Confirmation days before posted reseller profit becomes "withdrawable" (0 = instant, max 3650).
  settlement_confirm_days: 7
```

| Option | Description |
|--------|-------------|
| `enabled` | Master toggle for reseller mode |
| `main_hosts` | List of main-site hosts; matches are handled as the main-site context |
| `trusted_forwarded_host` | When the reverse proxy is trusted, prefer `X-Forwarded-Host` for domain detection |
| `subdomain_base` | Base domain for system subdomains (requires wildcard DNS/TLS set up in advance) |
| `self_apply_enabled` | Whether users may self-apply from the personal center |
| `settlement_confirm_days` | Profit confirmation days; `0` for instant, max `3650` |

::: warning Production prerequisites
- Before enabling, add your real main-site domains to `main_hosts`, otherwise main-site requests may be misidentified as a reseller site.
- To use system subdomains, `subdomain_base` must point to a domain with **wildcard DNS (`*.shop.example.com`) and a wildcard TLS certificate** configured.
- The reverse proxy (Nginx, etc.) must pass the real `Host`/`X-Forwarded-Host` to the backend — reseller domain resolution relies on that header.
:::

---

## 3. End-to-End Flow

```
User applies  →  Admin reviews (sets markup limit)  →  Assign/bind domain
   →  Reseller configures site + product pricing  →  Go live
   →  Buyer orders & pays on reseller site  →  Profit posted (confirmation period)
   →  Becomes withdrawable on maturity  →  Reseller requests withdrawal  →  Admin reviews & pays
```

---

## 4. Becoming a Reseller (User Side)

### 4.1 Submitting an Application

1. Open the "Reseller Center" in the storefront personal center
2. Fill in the application reason under "Reseller Application" and submit
3. The application enters the "Pending Review" state, awaiting admin processing

> Users can self-apply only after the admin enables `enabled` and `self_apply_enabled: true` in `config.yml`.

### 4.2 Application Status

| Status | Identifier | Description |
|--------|-----------|-------------|
| Pending Review | `pending_review` | Submitted, awaiting admin review |
| Active | `active` | Approved; reseller identity is in effect |
| Rejected | `rejected` | Rejected; the user may edit the reason and **reapply** |
| Disabled | `disabled` | Disabled by admin; the reseller site goes offline immediately and cannot reapply |

> A rejected (`rejected`) user can submit again; a disabled (`disabled`) user cannot self-reapply and must contact the admin for restoration.

---

## 5. Admin Review (Admin Panel)

Manage the reseller lifecycle under "Reseller Management → Reseller Review" in the admin panel.

### 5.1 Review Actions

| Action | Description |
|--------|-------------|
| Approve | Set the "default markup percent" and "max markup limit"; reseller identity takes effect |
| Reject | Provide a rejection reason; the user may reapply |
| Disable | Immediately take all of the reseller's domains offline (reseller site returns unavailable) |
| Restore | Restore a disabled reseller back to the active state |

### 5.2 Operational Parameters

When approving or editing later, the following reseller-level parameters can be adjusted:

| Parameter | Description |
|-----------|-------------|
| Default markup percent | The uniform markup percentage applied when a product/SKU has no individual pricing |
| Max markup limit | Hard cap on a single item's implied markup; exceeding it fails order/display validation (`0` = no limit) |
| Settlement status | `normal` / `frozen`; once frozen, the reseller is barred from withdrawing |

### 5.3 State Machine

```
              ┌──────────────┐
Application ─▶ │ pending_review│
              └──────┬───────┘
        approve │        │ reject
                ▼        ▼
            ┌────────┐ ┌──────────┐  reapply
            │ active │ │ rejected │ ─────────▶ pending_review
            └───┬────┘ └────┬─────┘
        disable │           │ disable
                ▼           ▼
            ┌──────────────────┐
            │     disabled     │ ──restore──▶ active
            └──────────────────┘
```

---

## 6. Domain Binding

The reseller site identifies the tenant context by domain, and supports two domain types.

### 6.1 System Subdomain

Assigned by the admin in the panel, based on `subdomain_base` in `config.yml`:

- e.g. with `subdomain_base: shop.example.com`, assigning the subdomain `hello` yields `hello.shop.example.com`
- A system subdomain is **automatically marked verified and active** upon assignment, with no extra review
- Subdomain labels allow only lowercase letters, digits, and hyphens, with length ≤ 63

### 6.2 Custom Domain

The reseller submits their own domain (e.g. `shop.mybrand.com`) under "Domain Management" in the storefront:

1. The reseller points the domain's DNS (CNAME/A record) to the platform
2. They submit the domain in the storefront; it enters the "Pending Review" state
3. The domain takes effect after the admin approves it under "Reseller Management"

> In the current version, custom domains go through **manual admin review**; automatic DNS verification is not yet enabled. The submitted domain cannot be a main-site domain or `subdomain_base` and its subdomains.

### 6.3 Domain Status and Primary Domain

| Dimension | Values |
|-----------|--------|
| Binding status | `pending_review` / `active` / `disabled` |
| Verification status | `pending` / `verified` / `failed` |
| Primary domain | `is_primary`; each reseller has exactly one primary domain |

- The primary domain is used to generate order snapshots and the canonical public-facing domain
- When the primary domain is disabled, the system **automatically re-selects** one of the other "active + verified" domains as primary
- After domain approval/disabling, the reseller domain resolution cache is invalidated immediately, so changes take effect right away

---

## 7. White-Label Site Configuration

Resellers customize their storefront appearance under "Site Configuration" in the storefront. All text fields support multiple languages (matching the main site):

| Option | Description |
|--------|-------------|
| Site name | Reseller site title |
| Logo / Favicon | Site logo and browser icon |
| Announcement | Homepage announcement content |
| Support info | Customer support contact / entry |
| SEO | Title, description, keywords, etc. |
| Navigation | Top navigation menu |
| Footer links | Bottom link area |
| Theme | Site theme / color scheme |

> The reseller identity must be "active" to edit the site configuration. Images are submitted via the reseller console's upload endpoint.

---

## 8. Products and Pricing

### 8.1 Listing / Unlisting

- Resellers can **list or unlist** at the product / SKU level (`is_listed`) under "Product Management"
- Unlisted products / SKUs are not shown on the reseller site and cannot be ordered

### 8.2 Pricing Modes

Each product / SKU can independently choose a pricing mode:

| Mode | Identifier | Calculation |
|------|-----------|-------------|
| Inherit | `inherit` | Use the base price, no markup (or fall back to the upper-level rule) |
| Percentage markup | `markup_percent` | `base price × (100 + markup%) / 100` |
| Fixed markup | `fixed_markup` | `base price + fixed markup amount` |
| Fixed price | `fixed_price` | Directly specify the reseller price |

### 8.3 Pricing Hierarchy and Inheritance

Pricing is resolved **top-down** in the following priority, adopting the first non-"inherit" rule encountered:

```
SKU-level setting  ▶  Product-level setting  ▶  Reseller default markup  ▶  Base price (inherit)
```

- If the SKU setting's mode is not `inherit`, the SKU rule is used first
- Otherwise, if the product setting's mode is not `inherit`, the product rule is used
- Otherwise, if the reseller's "default markup percent" > 0, that percentage is applied
- If none match, the price equals the base price

### 8.4 Pricing Floor (Validation)

To prevent platform loss, the reseller price must satisfy all of the following, otherwise order and display validation fails:

- Reseller price **> 0**
- Reseller price **≥ base price** (cannot be lower than the main-site price)
- If the SKU has a cost price (> 0), reseller price **≥ cost price**
- If the reseller has a "max markup limit" (> 0), the implied markup **≤ the limit**

> The display page tolerates stale/dirty configs: if a SKU fails pricing validation due to changes in base price / cost price / limit, it **only hides that SKU and logs a warning** instead of breaking the whole product page. Operators should fix the config based on the warning.

---

## 9. Orders and Fulfillment

- When a buyer orders on the reseller site, the system computes the reseller price using the rules above
- **Reseller orders do not participate in any main-site discounts**: coupons, member pricing, promotional pricing, and wholesale pricing do not apply and these discounts are zeroed out at checkout
- At checkout the system generates an **order pricing snapshot** (`reseller_order_snapshot`) recording the base price, reseller price, profit, risk-control verdict, etc., which serves as the basis for later profit settlement
- Order fulfillment (delivery / card-secret issuance) is **handled centrally by the main site** — resellers do not need their own inventory

---

## 10. Profit Settlement

### 10.1 Profit Posting

After an order is **paid successfully**, the system generates an "order profit" ledger entry from the order snapshot:

```
profit = Σ (reseller price − base price)   // accumulated across items in the order
```

- Ledger type `order_profit`, initial status `pending_confirm`
- Available time = payment time + `settlement_confirm_days`
- Not posted when profit is 0 or negative, or the snapshot marks it as non-profit-eligible (see the "Risk Control" section below)
- Posting is idempotent per order; no duplicates are generated

### 10.2 Confirmation Period

A scheduled backend worker batch-converts **matured** `pending_confirm` entries to `available` (withdrawable) and refreshes the balance cache. With `settlement_confirm_days: 0`, profit is withdrawable instantly.

### 10.3 Refund Deduction

On order refunds, the system proportionally claws back posted profit and creates a `refund_deduct` (negative) ledger entry:

- The deduction ratio uses the **order total as a fixed denominator**, so cumulative deductions across multiple partial refunds equal exactly the original profit — **never over-deducting**
- If the corresponding profit is still within the confirmation period (`pending_confirm`), the refund deduction entry also stays pending and uses the same available time, maturing together with the profit — preventing un-matured profit from being deducted into an "available negative balance" that wrongly freezes the account
- Refund details are allocated by each item's share of the profit

### 10.4 Balance Accounts

Each reseller is ledgered independently **per currency** (`reseller_balance_account`):

| Field | Description |
|-------|-------------|
| Available balance | Confirmed net amount eligible for withdrawal (including negative entries such as refunds) |
| Locked balance | Amount already in a withdrawal request, awaiting review |
| Negative balance | The owed amount when the available net is negative |

Account status: `normal` / `negative_balance` / `frozen_review` / `disabled`. Withdrawals are forbidden when in a negative-balance, frozen, or disabled state.

### 10.5 Ledger Types and Statuses

**Ledger types:**

| Type | Description |
|------|-------------|
| `order_profit` | Order profit posting (positive) |
| `refund_deduct` | Refund deduction (negative) |
| `manual_adjust` | Manual admin adjustment |

**Ledger statuses:**

| Status | Description |
|--------|-------------|
| `pending_confirm` | Pending confirmation (within the confirmation period) |
| `available` | Withdrawable |
| `locked` | Locked (in a withdrawal request) |
| `withdrawn` | Withdrawn |
| `canceled` | Voided |

---

## 11. Withdrawal

### 11.1 Reseller Request

1. Check the available balance per currency under "Finance / Withdrawal" in the storefront
2. Fill in the withdrawal amount, currency, withdrawal channel, and payee account, then submit
3. The system validates the amount against the **net available balance** (including negative entries such as refunds), rejecting if insufficient
4. On success, the system locks the corresponding withdrawable entries; if the last entry's amount exceeds what is needed, it is automatically **split**

> The reseller identity must be "active" with settlement status `normal` to withdraw; withdrawals are not allowed when the account is in a negative-balance / frozen / disabled state.

### 11.2 Admin Review

Handled under "Reseller Management → Withdrawal Management" in the admin panel (protected by finance permissions):

- **Approve (pay)**: locked entries become "withdrawn"; the admin completes payout offline
- **Reject**: provide a rejection reason; the locked entries are released back to "available"

---

## 12. Risk Control

During the checkout pricing phase, the system performs self-dealing risk checks. When triggered, that order's **profit is not credited to the reseller's account** (the buyer is still charged as usual; it simply isn't settled to the reseller):

| Block type | Identifier | Description |
|------------|-----------|-------------|
| Self-purchase | `self_dealing_owner` | The buyer is the reseller themselves |
| Related account | `self_dealing_related_account` | The buyer is flagged as a related account of the reseller |

In addition, the "Pricing Floor" above guarantees at the source that the reseller price is never below the base/cost price, so the platform never loses money on any reseller order.

---

## 13. Notes

- Reseller mode takes effect only after `enabled` is turned on in `config.yml`.
- System subdomains rely on wildcard DNS/TLS for `subdomain_base`; custom domains currently go through manual review.
- Reseller orders do not stack any main-site discounts (coupons / member pricing / promotional pricing / wholesale pricing).
- The reseller price cannot be below the base price and cost price, and is bound by the max markup limit.
- Profit confirmation days and the in-period refund "mature-together" logic are handled automatically by the backend; in-period refunds do not wrongly freeze the account.
- Withdrawals are based on the net available balance; negative-balance / frozen / disabled accounts cannot withdraw.
- Admin reseller "Withdrawal / Finance" endpoints are protected by finance permissions and require the corresponding RBAC role to operate.
