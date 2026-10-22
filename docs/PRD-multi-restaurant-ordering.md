# PRD: Multi-Restaurant Ordering (Single Cart, Single Checkout)

**Product:** Zomato Food Ordering
**Feature:** Add items from 2–3 restaurants in one cart and checkout together
**Status:** Draft
**Owner:** Product — Food Ordering Experience
**Branch:** `arena/f7a22f21-zomato-new-feature`
**Target release:** vNext (food ordering app: Android / iOS / Web)

---

## 1. Summary

Today, a Zomato cart can hold items from only one restaurant. Users who want food from multiple restaurants (e.g., burgers from one place and dessert from another) must place two separate orders, pay twice, pay delivery fees twice, and track two deliveries.

This PRD defines **Multi-Restaurant Ordering**: users can add items from **2 to 3 restaurants into a single cart** and complete **one combined checkout** with a single payment, a single delivery address, and unified order tracking.

---

## 2. Problem Statement

| User pain point | Business impact |
|-----------------|-----------------|
| Group orders require consensus on one restaurant; someone always compromises | Lower average order value per session; abandoned carts when cravings are split |
| Multiple restaurants = multiple checkouts, multiple payments, multiple delivery fees | Higher checkout friction; lower conversion on mixed-craving occasions |
| No way to combine a main meal from one restaurant with dessert/snacks from another | Missed cross-category attach (desserts, beverages, snacks) |

---

## 3. Goals & Success Metrics

### Goals
1. Enable users to build a single cart with items from 2–3 restaurants.
2. Deliver one combined checkout experience (single payment, single address, one order summary).
3. Maintain per-restaurant operational integrity (separate kitchen tickets, payouts, and SLAs).
4. Keep delivery promise times accurate per restaurant.

### Success Metrics (post-launch, 90 days)
- **Attach rate:** % of multi-restaurant carts that complete checkout ≥ 25%.
- **AOV lift:** Multi-restaurant order AOV ≥ 1.4× single-restaurant order AOV.
- **Conversion:** Checkout conversion for multi-restaurant carts within 5% of single-restaurant baseline.
- **CSAT:** Post-delivery CSAT for multi-restaurant orders ≥ 4.2/5.
- **Ops error rate:** Wrong/missing items per restaurant < 2%.

---

## 4. User Stories

1. As a hungry user, I can add items from Restaurant A to my cart, then add items from Restaurant B (and C) without clearing my cart.
2. As a user, when I add items from a second restaurant, I am clearly told my cart now contains multiple restaurants and delivery may come in separate bags/legs.
3. As a user, I see one cart grouped by restaurant, with per-restaurant subtotals, taxes, and packaging charges, plus one combined total.
4. As a user, I pay once with a single payment method and receive one order confirmation.
5. As a user, I can track each restaurant’s order status separately (Accepted → Preparing → Out for delivery → Delivered).
6. As a restaurant partner, I receive a separate kitchen ticket/OTP for only the items meant for my restaurant.
7. As a delivery partner, I receive clear pickup instructions (single pickup or sequential pickups) for the combined order.

---

## 5. Functional Requirements

### FR-1: Cart composition
- A cart may contain items from **minimum 1, maximum 3** restaurants.
- Items from different restaurants cannot coexist unless the user explicitly confirms the multi-restaurant prompt.
- Cart is keyed by `user_id` + `session_id`; persisted across app restarts.

### FR-2: Adding items from a second restaurant
- When the cart is non-empty and the user taps "Add to cart" at a different restaurant, show a modal:
  - "Your cart has items from [Restaurant A]. Adding [Restaurant B] will create a multi-restaurant order. Delivery may arrive in separate bags. Continue?"
  - Options: "Continue" | "Replace cart" (clear A and start fresh with B) | "Cancel".
- If the user already has 2 restaurants and tries to add a 3rd: allow it (max 3).
- If the user tries to add a 4th restaurant: block with a toast: "Maximum 3 restaurants per order. Remove a restaurant to add more."

### FR-3: Cart UI
- Cart screen groups line items by restaurant (restaurant name, rating, estimated prep time per section).
- Per-restaurant section shows: itemized list, subtotal, taxes, packaging/delivery charges applicable to that restaurant.
- Order summary at bottom: combined total, single delivery fee logic (see FR-8), applied promo, final payable amount.
- "Remove restaurant" action removes all items from that restaurant and re-calculates totals.

### FR-4: Checkout
- Single delivery address selector (one address for the whole order).
- Single payment flow: one payment intent for the combined total (wallet, UPI, cards, COD where allowed).
- One order confirmation screen with a combined order ID and per-restaurant sub-order IDs.
- Promo codes: applicable either order-wide or per-restaurant (see FR-9).

### FR-5: Order splitting (backend)
- On checkout, the combined order is split into **N sub-orders** (one per restaurant).
- Each sub-order references the parent multi-restaurant order.
- Kitchen notification/OTP goes only to the relevant restaurant (restaurant A never sees restaurant B's items).

### FR-6: Order tracking
- User sees one consolidated tracking screen with per-restaurant status chips:
  - Restaurant A: Preparing → Out for delivery
  - Restaurant B: Accepted
- Each restaurant's order can move through statuses independently.
- Delivery tracking reflects the combined delivery leg(s).

### FR-7: Delivery model options (launch with Option A)
| Option | Description |
|--------|-------------|
| **A (Launch)** | One delivery partner picks up sequentially from Restaurant A then Restaurant B (cluster pickup), delivers in one trip. Estimated time = max(prep times) + travel between restaurants + travel to user. |
| B (Future) | Separate delivery partners per restaurant; items may arrive at different times. |

### FR-8: Pricing & fees
- One combined checkout total.
- Delivery fee: single fee for the combined order (not additive per restaurant), capped at a configurable maximum.
- Minimum order value: applies per restaurant (each sub-order must meet that restaurant's cart minimum; if not, prompt user to add more from that restaurant or remove it).
- Packaging charges and taxes are computed per restaurant and summed.
- Surge/dynamic pricing applies per restaurant based on that restaurant's zone/demand.

### FR-9: Promotions & discounts
- Order-wide promo codes (e.g., "SAVE50") apply to the combined total.
- Restaurant-specific promos apply only to that restaurant's sub-order subtotal.
- Zomato Gold / loyalty benefits apply per restaurant per that restaurant's program rules.

### FR-10: Cancellations & refunds
- User can cancel the entire multi-restaurant order before any restaurant accepts.
- After acceptance, user can cancel individual sub-orders (per restaurant) with pro-rated refund for that sub-order only.
- Refunds split back to the original payment instrument proportionally.

---

## 6. Non-Functional Requirements

| Category | Requirement |
|----------|-------------|
| Performance | Cart add/update < 300ms p95; checkout < 2s p95 |
| Availability | Cart service 99.9% uptime |
| Consistency | Cart state consistent across app/web within 1 second |
| Security | Payment tokenization; no card numbers stored; sub-order data isolated per restaurant |
| Accessibility | WCAG 2.1 AA compatible cart and checkout screens |

---

## 7. Out of Scope (v1)

- Scheduling multi-restaurant orders for future delivery (ASAP only at launch).
- Group carts / shared carts among multiple Zomato accounts (single-user cart only).
- Multi-restaurant ordering for "dinner party" large catering (commercial > 10 people).
- Combining grocery/instamart items with restaurant food in one cart.
- Re-ordering a past multi-restaurant order with one tap (re-order per restaurant only).
- Option B (separate delivery partners / staggered delivery times).

---

## 8. UX / UI Design Notes

1. **Cart header:** "Multi-restaurant order · 2 restaurants" chip when applicable.
2. **Restaurant cards in cart:** collapsible sections per restaurant with item count and sub-total.
3. **Empty state for second restaurant:** When adding from a new restaurant, show the confirmation modal before merging.
4. **Checkout CTA:** "Pay ₹{total}" — single button.
5. **Tracking:** Timeline per restaurant; delivery partner ETA shown for the final combined leg.
6. **Restaurant detail page:** "Add to cart" works normally; if cart has another restaurant, intercept with FR-2 modal.

---

## 9. Data Model Changes (indicative)

```sql
-- orders (existing) gains:
orders.is_multi_restaurant BOOLEAN DEFAULT FALSE
orders.parent_order_id UUID        -- NULL for single-restaurant orders; links sub-orders
orders.delivery_fee_cents INTEGER  -- single fee for combined order

-- order_items (existing) gains no change (restaurant_id already implied via menu_item)

-- cart / cart_items (existing) gains:
cart_items.restaurant_id FK        -- enforce all items in a cart share <= 3 distinct restaurant_ids

-- order_payments (existing): one payment intent per parent order; splits recorded per sub-order
```

**API changes (indicative):**
- `POST /cart/add` — validate restaurant count ≤ 3; return `cart.is_multi_restaurant`.
- `POST /cart/merge` — server-side merge with confirmation token (client must have accepted FR-2 modal).
- `POST /checkout/create` — creates parent order + N sub-orders; returns one payment intent.
- `GET /orders/{id}/tracking` — returns consolidated status with per-sub-order statuses.

---

## 10. Rollout Plan

| Phase | Scope | Audience |
|-------|-------|----------|
| Phase 1 | Internal dogfood (Zomato employees) | Internal |
| Phase 2 | Beta: 5% of users in 2 metros (Delhi-NCR, Mumbai) | Opt-in beta |
| Phase 3 | GA: all users in top 10 metros, 2–3 restaurants max | All |
| Phase 4 | Expand to all cities; consider 4+ restaurants post-metrics review | All |

Feature flag: `multi_restaurant_ordering` (default OFF).

---

## 11. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Food arrives cold because one restaurant preps slower | Show per-restaurant prep times; cluster-pickup sequencing; delay second pickup until first is ready. |
| Restaurant B rejects order after A is cooking | Allow per-sub-order cancellation/refund; notify user which items are affected. |
| Delivery partner confusion at two pickups | Clear pickup sequence in partner app; order items packed and labeled per restaurant. |
| Promo abuse (stacking restaurant coupons order-wide) | Validate promo applicability per sub-order; reject invalid combos at checkout. |
| Higher ops complexity / support tickets | In-app FAQ "Why did my food arrive in two bags?"; support tool to view parent + sub-orders together. |

---

## 12. Open Questions (to resolve in design review)

1. Should delivery fee be free above a combined threshold (e.g., ₹499) even if individual restaurants are below their free-delivery threshold?
2. Can users tip per restaurant or one tip split across restaurants?
3. Does Zomato Gold free delivery apply if at least one restaurant is Gold-eligible, or must all be?
4. Max restaurants: launch with 3, or start with 2 for simplicity?

---

## 13. Appendix: Example Flow

1. User browses "Burger King," adds 2 burgers (₹350). Cart: 1 restaurant.
2. User searches "Häagen-Dazs," adds 1 ice cream (₹220). Modal: "Your cart has Burger King. Adding Häagen-Dazs makes this a multi-restaurant order." User taps Continue.
3. Cart screen: two sections — Burger King ₹350, Häagen-Dazs ₹220; delivery fee ₹40 (single); total ₹610.
4. User selects address, taps "Pay ₹610." Single payment success.
5. Backend creates Order #P (parent) + Sub-order #A (Burger King) + Sub-order #H (Häagen-Dazs).
6. Burger King gets ticket for burgers only; Häagen-Dazs gets ticket for ice cream only.
7. User tracking: Burger King "Out for delivery," Häagen-Dazs "Preparing."
8. Delivery partner picks up burgers, then ice cream, delivers both in one trip.

---

*End of PRD. Feedback welcome from Product, Engineering, Operations, and Restaurant Partnerships.*
