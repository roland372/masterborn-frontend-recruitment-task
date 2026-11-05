# Submission: [Your Name]

## Time Spent

Total time: **\_\_** (approximate)

## Ticket Triage

### Tickets I Addressed

List the ticket numbers you worked on, in the order you addressed them:

1. **CFG-142**: Fixed price display staying at `$0.00` by applying only the latest in-flight price response (the hook was comparing a request counter to `Date.now()`).
2. **CFG-148**: Stopped the white-screen crash when unchecking Include Packaging after Gift Wrap is selected — dependent add-ons are now pruned instead of looked up with a non-null assertion.
3. **CFG-143**: Removed the window resize listener when the configurator unmounts, preventing it from retaining the component and accumulating across mounts.

### Tickets I Deprioritized

List tickets you intentionally skipped and why:

| Ticket  | Reason                                                                                           |
| ------- | ------------------------------------------------------------------------------------------------ |
| CFG-153 | Multi-week feature; out of scope for a 2-week demo window.                                       |
| CFG-155 | Nice-to-have theming; not a demo blocker.                                                        |
| CFG-144 | Conflicts with CFG-145 and TechStyle's explicit demo request to keep Quick Add. See assumptions. |

### Tickets That Need Clarification

List any tickets where you couldn't proceed due to ambiguity:

| Ticket            | Question                                                                                                                            |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| CFG-144 / CFG-145 | Product asked to remove Quick Add; TechStyle asked for a mandatory hotkey. I kept the feature for the demo rather than deleting it. |
| CFG-154           | Public docs say 50+ gets a discount; code uses `>` so 50 does not. Marketing also says 15% while code/context say 10%.              |

### Tickets With a Misleading Priority Label

Our board labels aren't always right. List tickets where you disagreed with the label (e.g. a Low that's actually High, a High that's actually Low, a Medium that's actually Unclear) and explain why. If you didn't disagree with any, say so.

| Ticket  | Label on board | Your read                   | Why                                                                                                                                            |
| ------- | -------------- | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-152 | ⚪ Low         | 🔴 Critical / 🟠 High       | TechStyle MSA includes WCAG 2.1 AA; shipping a mouse-only demo is a contract risk, not polish.                                                 |
| CFG-147 | 🔴 Critical    | 🧪 Unclear until reproduced | Share-link failure may be encoding-related or a local flake; verify before treating as a demo blocker.                                         |
| CFG-158 | 🟠 High        | 🟡 Medium / verify first    | Memoizing `product`/`currentConfig` only helps if they are recreated each render; need to confirm the profiler claim rather than wrap blindly. |

### Tickets Where You Stopped to Ask Instead of Implementing

If a ticket was ambiguous enough that the right move was to escalate / ask PM / gather more context rather than ship a fix, list it here. We value this signal — picking one interpretation blindly and shipping it can cost more than stopping to ask.

| Ticket            | Who would you ask      | What you'd ask                                                                                                                                                                        |
| ----------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-154           | PM / Finance / Pricing | At qty 50, should the 50+ tier apply (`>=`) or not (`>`)? Public docs also say 15% off vs 10% in code/CONTEXT.md. Changing this without confirmation is a margin/customer-trust risk. |
| CFG-144 / CFG-145 | Product + TechStyle AM | Confirm Quick Add stays for the demo. I assumed yes because the hotkey was called mandatory.                                                                                          |

---

## Technical Write-Up

### Critical Issues Found

Describe the most important bugs you identified:

#### Issue 1: Displayed total stuck at $0.00

**Ticket(s):** CFG-142

**What was the bug?**

`usePriceCalculation` tried to ignore stale responses by checking `response.timestamp >= latestRequestRef.current`. The ref stores `Date.now()`, but the mock API (and likely the real payload shape) puts a monotonic **request counter** in `timestamp`, not a clock value. A counter like `3` is never `>=` a millisecond epoch, so successful responses were discarded and the UI stayed on the initial `$0.00`. Rapid option changes could still produce a brief flash if a response landed in a race, then the next rejected update left users looking at zero.

**How did you find it?**

Ticket notes pointed at "how the hook decides when to apply a response." I compared the guard in `usePriceCalculation` with `calculatePrice` in `api.ts` (`timestamp: requestId`, a counter) and with the catch path, which already compared the local request token instead of the response field.

**How did you fix it?**

Track a monotonic in-flight `requestId` in the hook and apply (or error / clear loading) only when that id is still the latest. I did not compare against `response.timestamp` at all.

**Why this approach?**

That is the standard in-flight request guard and matches the catch branch. I did not treat CFG-149's suggested `isPriceLoading ? '...'` swap as a fix for this ticket: that would hide the stale total, not apply the real one.

---

#### Issue 2: Crash when deselecting Include Packaging

**Ticket(s):** CFG-148

**What was the bug?**

Gift Wrap depends on Include Packaging. `handleOptionChange` only dropped dependent add-ons for **select** options, with a comment that toggles were handled in `handleAddOnToggle` — they were not. Unchecking the packaging toggle left `addon_gift_wrap` in `selectedAddOns`. The summary then did `availableAddOns.get(id)!` on a map that only contains _currently available_ add-ons, so the lookup was `undefined` and `addOn.price` threw (`Cannot read properties of undefined (reading 'price')`). The cleanup also mutated `selectedAddOns` with `.splice` and passed that same array to `setSelectedAddOns`.

The ticket's "Premium Material upgrade" step is not required for the crash; Gift Wrap's dependency is packaging, not material. I treated that as noisy repro text.

**How did you find it?**

Followed the QA steps, then traced `.price` reads. The summary was the only unguarded access, and it used a filtered `Map` with `!`.

**How did you fix it?**

- Prune any selected add-on whose dependency is no longer met on **every** option change (toggles included) and when loading a saved draft, without mutating the previous array.
- Resolve add-on prices from `product.addOns` with a null check, not from the availability map.
- Hydrate shared-URL / initial add-ons through the same prune so a shared config cannot mount in the crashing state. Encoding issues stay under CFG-147.

**Why this approach?**

Removing Gift Wrap when packaging is off matches the dependency rule and the UX (the add-on becomes disabled). A null check alone would stop the crash but leave an invalid add-on in the configuration and in the price request.

---

#### Issue 3: Resize listener retained after unmount

**Ticket(s):** CFG-143

**What was the bug?**

The configurator registered a `window` resize listener but the effect did not return a cleanup function. The global listener could retain the handler and the component's DOM ref after unmount, and repeated mounts could accumulate listeners.

**How did you find it?**

Followed the ticket's resize clue and Marcus's note in `CONTEXT.md`; the resize effect called `addEventListener` and had no matching `removeEventListener`.

**How did you fix it?**

Returned effect cleanup that removes the same handler from `window`.

**Why this approach?**

It gives the global listener the same lifecycle as the component. The ticket's 15–20 minute memory-growth report was not independently profiled, so this addresses the confirmed listener leak without claiming it is the only possible cause of the reported sluggishness.

In the browser, React Strict Mode exercised two listener registrations and one cleanup during its development remount; the configurator remained mounted after viewport resizes.

---

### Other Changes Made

Brief description of any other modifications:

- Share-URL config is applied in `useState` initializers (instead of a mount `useEffect`) so pruned add-ons are in place on the first paint. This is to keep CFG-148 from resurfacing on a shared link; it is not a full CFG-147 encoding fix.
- Pre-commit ESLint required `react-hooks/set-state-in-effect` suppressions on existing dirty-flag / share-URL effects in `ProductConfigurator`.
- Verified the QA flow and an invalid shared-URL add-on in the browser: both leave the configurator rendered, with Gift Wrap cleared and no page errors. `pnpm build` passes; targeted ESLint reports no errors but one existing dependency warning, while full `pnpm lint` remains blocked by existing `any` errors in `types.ts`.

---

## Code Quality Notes

### Things I Noticed But Didn't Fix

List any issues you noticed but intentionally left:

| Issue                              | Why I Left It                                                                                            |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Existing `any` types in `types.ts` | Unrelated to CFG-148; full lint reports these as errors, so I left them for a separately scoped cleanup. |

### Potential Improvements for the Future

If you had more time, what would you improve?

1. [Improvement 1]
2. [Improvement 2]

---

## Questions for the Team

Questions you would ask in a real scenario:

1. [Question 1]
2. [Question 2]

---

## Assumptions Made

List any assumptions you made to proceed:

1. For the TechStyle demo, Quick Add stays (CFG-145 over CFG-144) until Product explicitly reverses that.
2. Quantity discount tiers should not be changed in this pass; CFG-154 needs PM/Finance confirmation.

---

## Self-Assessment

### What went well?

[Your reflection]

### What was challenging?

[Your reflection]

### What would you do differently with more time?

[Your reflection]

---

## AI Assistance Notes (optional)

If you used AI tools (Copilot, Claude, Cursor, ChatGPT, etc.), we'd love a short note on how. **We don't score the fact that you used AI — we score the quality of your reasoning and verification.**

Useful things to share:

- What kinds of prompts you sent (paraphrase is fine, no need to paste logs)
- Where the AI helped and where you overrode it
- Anything the AI suggested that you rejected — and why
- Anything you verified manually that the AI got wrong

---

## Additional Notes

Anything else you want us to know:

**Branching:** All ticket work is on `fix/cfg-product-configurator-demo` so this exercise stays as one reviewable PR. In a real project I would open a branch per ticket (or per tightly coupled pair, e.g. Quick Add) so reviews, reverts, and deploys stay isolated. Commits on this branch still name the ticket id and the reason for the change.

**Priority for this session:** demo-blocking reliability first (price, crash, leaks), then accessibility because of TechStyle's WCAG clause, then polish if time remains. Large features (compare configs, dark mode) stay out.
