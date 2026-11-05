# Submission: [Your Name]

## Time Spent

Total time: **\_\_** (approximate)

## Ticket Triage

### Tickets I Addressed

List the ticket numbers you worked on, in the order you addressed them:

1. **CFG-142**: Fixed price display staying at `$0.00` by applying only the latest in-flight price response (the hook was comparing a request counter to `Date.now()`).
2. **CFG-148**: Stopped the white-screen crash when unchecking Include Packaging after Gift Wrap is selected — dependent add-ons are now pruned instead of looked up with a non-null assertion.
3. **CFG-143**: Added cleanup for the window resize listener on unmount. This fixes a confirmed leak, though the reported long-session memory growth still needs profiling.
4. **CFG-152**: Added keyboard-operable color swatches, visible keyboard focus for quantity controls, and focus management for the dialogs.
5. **CFG-145**: Added Ctrl+Enter / Command+Enter for Quick Add, with the shortcut shown on the button and suppressed while editing, in a modal, or when Quick Add is unavailable.

### Tickets I Deprioritized

List tickets you intentionally skipped, already found to be covered, or left for confirmation:

| Ticket  | Reason                                                                                                                                                                                                            |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-143 | The resize-listener leak is fixed, but I did not run a 15–20 minute memory profile, so I can't confirm whether that explains all of the reported slowdown.                                                        |
| CFG-144 | Conflicts with CFG-145 and TechStyle's explicit demo request to keep Quick Add. I kept it for the demo and documented the decision; Product should confirm the post-demo direction.                               |
| CFG-146 | Low-impact timestamp issue; the UI already formats saved dates in the user's local timezone. I didn't reproduce the reported offset and left it unchanged pending a concrete example.                             |
| CFG-147 | The supplied special characters point to a likely encoding issue, but the ticket also raises a possible production-versus-development difference. I left the share format unchanged pending compatibility checks. |
| CFG-149 | The price display already dims during calculation via its `price-loading` class, so I found no additional change needed for the stated acceptance criteria.                                                       |
| CFG-150 | The referenced mobile screenshot wasn't included, and the reported JavaScript cause isn't demonstrated. I left the grid calculation unchanged until QA can provide a reproducible viewport/example.               |
| CFG-151 | User-friendly errors are worthwhile, but were lower priority than reliability and the demo's explicit accessibility/hotkey needs.                                                                                 |
| CFG-153 | A two-to-three-week feature is out of scope for this time-boxed pass.                                                                                                                                             |
| CFG-154 | Conflicting discount rules could affect customer pricing and margin; no change without PM/Finance confirmation.                                                                                                   |
| CFG-155 | Dark mode is a nice-to-have for this demo; the theming approach also needs a product decision (system preference versus an explicit prop).                                                                        |
| CFG-156 | Non-blocking warnings; deferred in favor of demo-facing reliability and keyboard support.                                                                                                                         |
| CFG-157 | The widget has no defined close/navigation contract with its embedding page. The expected behavior and integration API need clarification before adding a confirmation flow.                                      |
| CFG-158 | The suggested memoization is not supported by the current code: `currentConfig` is already memoized and the app passes a stable `mockProduct`. I didn't add more memoization without profiling evidence.          |

### Tickets That Need Clarification

List any tickets where you couldn't proceed due to ambiguity:

| Ticket            | Question                                                                                                                                                                                                                                          |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-144 / CFG-145 | Product asked to sunset Quick Add, while TechStyle requires its hotkey for the demo. I kept it for the demo; can Product confirm whether Quick Add remains after the demo?                                                                        |
| CFG-147           | The example includes Polish diacritics and emoji, while the current encoder uses `btoa` on JSON text. Can we verify the customer link against production and confirm the backwards-compatible encoding expected before changing the share format? |
| CFG-150           | Can QA provide the missing screenshot and exact mobile viewport so we can reproduce the layout issue and distinguish a grid-calculation problem from a CSS/layout issue?                                                                          |
| CFG-154           | At quantity 50, should the 50+ tier apply (`>=`) or not (`>`)? Public docs also say 15% off versus 10% in code/CONTEXT.md. Which pricing rule is approved?                                                                                        |
| CFG-157           | Which host-page navigation/close action should the embedded widget intercept, and should the host provide a callback or another integration hook for "Save & Close" and "Discard"?                                                                |
| CFG-158           | Can Engineering share the profiler capture and clarify which rerender is costly? `currentConfig` is already memoized and the app's product object is stable, so I don't want to add memoization without confirming the actual source of work.     |

### Tickets With a Misleading Priority Label

Our board labels aren't always right. List tickets where you disagreed with the label (e.g. a Low that's actually High, a High that's actually Low, a Medium that's actually Unclear) and explain why. If you didn't disagree with any, say so.

| Ticket  | Label on board | Your read                       | Why                                                                                                                                                                                                |
| ------- | -------------- | ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-152 | ⚪ Low         | 🔴 Critical / 🟠 High           | TechStyle MSA includes WCAG 2.1 AA; keyboard access is a contract and demo risk, not just polish.                                                                                                  |
| CFG-147 | 🔴 Critical    | 🧪 Unclear pending verification | The example suggests a Unicode encoding problem, but the ticket also says production worked recently. Verify the customer URL and production behavior before assigning full demo-blocker priority. |
| CFG-158 | 🟠 High        | 🟡 Medium / verify first        | `currentConfig` is already memoized and the app passes a stable product object. The claimed performance win needs profiler evidence before changing code.                                          |

### Tickets Where You Stopped to Ask Instead of Implementing

If a ticket was ambiguous enough that the right move was to escalate / ask PM / gather more context rather than ship a fix, list it here. We value this signal — picking one interpretation blindly and shipping it can cost more than stopping to ask.

| Ticket            | Who would you ask      | What you'd ask                                                                                                                                                                        |
| ----------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-154           | PM / Finance / Pricing | At qty 50, should the 50+ tier apply (`>=`) or not (`>`)? Public docs also say 15% off vs 10% in code/CONTEXT.md. Changing this without confirmation is a margin/customer-trust risk. |
| CFG-144 / CFG-145 | Product + TechStyle AM | Product asked to sunset Quick Add, while TechStyle requires its hotkey for the demo. I kept it for the demo and would confirm the post-demo feature decision with Product.            |

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

#### Issue 4: Keyboard navigation gaps

**Ticket(s):** CFG-152

**What was the bug?**

Color swatches were mouse-only `div` elements without a tab stop or keyboard handling. Quantity buttons had no visible keyboard focus style. Modal focus was recorded and restored synchronously on close, but the dialogs did not receive focus when opened and did not keep tab navigation within the dialog.

**How did you find it?**

Compared the affected controls in `ProductConfigurator` and `styles.css` with the four areas listed in CFG-152. The option dropdowns are native HTML `<select>` elements, so their arrow-key behavior is provided by the browser; this ticket's "custom dropdown" note does not match the implementation.

**How did you fix it?**

- Added radio-group semantics, an accessible name, a roving tab stop, and arrow-key selection/navigation to color swatches.
- Added a high-contrast `:focus-visible` outline to color swatches and quantity buttons.
- Focus the first dialog control on open, keep Tab/Shift+Tab within the dialog, expose the modal dialog semantics, and restore focus to the trigger after close.

**Why this approach?**

It addresses the reproducible keyboard gaps with native browser primitives where available and completes the dialogs' focus lifecycle without replacing native selects or introducing a custom dropdown.

---

#### Issue 5: Quick Add keyboard shortcut

**Ticket(s):** CFG-145 (conflicts with CFG-144)

**What was the bug?**

Quick Add had no keyboard shortcut despite TechStyle's explicit requirement for the demo. Product's separate request to remove Quick Add conflicts with this account requirement.

**How did you find it?**

The Quick Add button already had a click handler, but no keyboard shortcut. CFG-144 and CFG-145 give conflicting product direction; CFG-145 states that TechStyle's PM made the hotkey mandatory for the upcoming demo.

**How did you fix it?**

Added Ctrl+Enter and Command+Enter to invoke the same Quick Add action. The button advertises the shortcut. It does not fire while the user is typing in an input, textarea, select, or editable region, while a modal is open, or when Quick Add is unavailable. The button is disabled during price recalculation.

**Why this approach?**

It meets the explicit demo requirement without deleting a feature TechStyle asked to use. The CFG-144 conflict remains documented as a decision for Product after the demo rather than treating the two tickets as simultaneously satisfiable.

---

### Other Changes Made

Brief description of any other modifications:

- Share-URL config is applied in `useState` initializers (instead of a mount `useEffect`) so pruned add-ons are in place on the first paint. This is to keep CFG-148 from resurfacing on a shared link; it is not a full CFG-147 encoding fix.
- Pre-commit ESLint required `react-hooks/set-state-in-effect` suppressions on existing dirty-flag / share-URL effects in `ProductConfigurator`.
- Verified the QA flow and an invalid shared-URL add-on in the browser: both leave the configurator rendered, with Gift Wrap cleared and no page errors. `pnpm build` passes; targeted ESLint reports no errors but one existing dependency warning, while full `pnpm lint` remains blocked by existing `any` errors in `types.ts`.
- Verified the CFG-152 keyboard interactions in the browser: color selection by arrow keys, dialog focus restoration on Escape, and native select arrow-key operation.
- Verified Ctrl+Enter and Command+Enter invoke Quick Add once when enabled; the shortcut is ignored while editing, while a modal is open, and during price recalculation. CFG-144 remains documented as a conflicting product request.

---

## Code Quality Notes

### Things I Noticed But Didn't Fix

List any issues you noticed but intentionally left:

| Issue                                            | Why I Left It                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `any` values in `ConfigChangeEvent` (`types.ts`) | `previousValue` and `newValue` are still typed as `any`, and ESLint reports both. `pnpm build` and TypeScript compilation pass today, so these did not block the fixes; I would replace them with a suitable shared value type when time allows, and prioritize that cleanup immediately if a required build/type-check starts failing. Since the repository's full `pnpm lint` currently fails on these errors, resolve them before treating that lint gate as green. |
| Full memory-growth diagnosis (CFG-143)           | I fixed the missing resize-listener cleanup, but did not run the ticket's 15–20 minute profiling session. Other sources of retained memory or lag may remain.                                                                                                                                                                                                                                                                                                          |
| Unicode share-link behavior (CFG-147)            | `btoa` is not UTF-8-safe for the supplied emoji/diacritic example. I left the encoding unchanged until compatibility with existing and production share links can be checked.                                                                                                                                                                                                                                                                                          |
| Full WCAG 2.1 AA audit (CFG-152)                 | This pass fixes the listed keyboard issues, but is not a complete accessibility audit or certification.                                                                                                                                                                                                                                                                                                                                                                |
| `product` memoization request (CFG-158)          | The app passes a stable product object and `currentConfig` is already memoized. I left the remaining performance claim for profiling rather than adding memoization without evidence.                                                                                                                                                                                                                                                                                  |

### Potential Improvements for the Future

If you had more time, what would you improve?

1. Add focused automated tests for out-of-order price responses, dependency pruning from toggles/drafts/share URLs, and the Quick Add shortcut's disabled/editing/modal cases.
2. Profile a sustained configuration session and complete a keyboard/screen-reader accessibility pass against TechStyle's agreed WCAG criteria.
3. Verify and correct UTF-8 share-link encoding with legacy-link compatibility tests, and resolve the remaining lint errors in `types.ts`.

---

## Questions for the Team

Questions you would ask in a real scenario:

1. For CFG-144/CFG-145, should Quick Add remain after the TechStyle demo, or is the sunset decision still active?
2. For CFG-154, what are the approved quantity threshold and discount percentage? I would want the pricing source of truth confirmed before changing checkout totals.
3. For CFG-147, can we test the customer's exact link against production and confirm whether old share links must remain readable?
4. For CFG-157, what host-page event should trigger the unsaved-changes prompt for an embedded widget, and what API should the host use to close it?

---

## Assumptions Made

List any assumptions you made to proceed:

1. For the TechStyle demo, Quick Add stays (CFG-145 over CFG-144) until Product explicitly reverses that.
2. Quantity discount tiers should not be changed in this pass; CFG-154 needs PM/Finance confirmation.
3. CFG-149's requested loading feedback is already present as a dimmed price while the `price-loading` class is active; I did not add a duplicate indicator.
4. The CFG-152 fixes address the keyboard gaps listed in the ticket, but do not by themselves establish WCAG 2.1 AA compliance.
5. The resize-listener cleanup addresses a confirmed leak, but I have not established that it is the only cause of CFG-143's long-session slowdown.

---

## Self-Assessment

### What went well?

I tackled the demo-facing reliability issues first and kept each ticket in a separate commit. For the crash and keyboard issues, I traced the existing component behavior and verified the reported interactions in the browser rather than relying on a code-only review. I also kept the unresolved product and pricing decisions visible in the submission notes instead of quietly choosing a permanent interpretation.

### What was challenging?

Several ticket descriptions mixed confirmed symptoms with assumptions. CFG-143's listener leak was clear, but the longer-session performance report needs a real profile; CFG-147 has a strong encoding clue but also raises production compatibility questions. Balancing Product's Quick Add sunset request against TechStyle's mandatory demo requirement was another judgment call.

### What would you do differently with more time?

I would add automated regression tests around the async pricing and add-on dependency flows, then profile a long session and run a broader accessibility audit. I would also verify the sample Unicode share URL against production before changing the encoding, since old shared links may need to keep working.

---

## AI Assistance Notes (optional)

If you used AI tools (Copilot, Claude, Cursor, ChatGPT, etc.), we'd love a short note on how. **We don't score the fact that you used AI — we score the quality of your reasoning and verification.**

Useful things to share:

- What kinds of prompts you sent (paraphrase is fine, no need to paste logs)
- Where the AI helped and where you overrode it
- Anything the AI suggested that you rejected — and why
- Anything you verified manually that the AI got wrong

I used Copilot as a coding and investigation assistant to trace the ticket-relevant code, implement small changes, and help verify behavior. I checked the implementation against the repository and exercised key flows in the browser. I did not accept suggested fixes that would mask the pricing bug, replace native controls unnecessarily, or choose a discount rule without confirmation. The remaining ambiguities and unverified claims are called out above.

---

## Additional Notes

Anything else you want us to know:

**Branching:** All ticket work is on `fix/cfg-product-configurator-demo` so this exercise stays as one reviewable PR. In a real project I would open a branch per ticket (or per tightly coupled pair, e.g. Quick Add) so reviews, reverts, and deploys stay isolated. Commits on this branch still name the ticket id and the reason for the change.

**Priority for this session:** demo-blocking reliability first (price, crash, leaks), then accessibility because of TechStyle's WCAG clause, then polish if time remains. Large features (compare configs, dark mode) stay out.
