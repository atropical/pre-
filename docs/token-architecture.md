# Token architecture — design principles

General strategies for structuring a design token system, covering colour semantics, theming, and spacing/type rhythm. Not tied to any specific token file — apply the shape of these rules to whatever collections/values a given system already has.

## At a glance

```
color
├─ intent        one shared meaning-vocabulary (§1)
│   ├─ default   qualifier · disqualifier · clear · warn · critical
│   └─ inverse   same roles, for a locally-inverted island (§2)
│
├─ contrast      fg/bg — the one axis that DOES invert with theme (§2)
│   ├─ default
│   │   ├─ fg            solid, full emphasis
│   │   ├─ fg (72% α)    alpha ramp instead of a second solid token (§3)
│   │   ├─ fg (46% α)    ...as many steps as needed, no new names
│   │   └─ bg
│   └─ inverse    mirror shape, for a locally-inverted island
│
├─ action        hierarchy + state, no meaning of its own (§1, §4)
│   ├─ main / optional / auxiliary  emphasis tier (primary → secondary → low-commitment/link)
│   └─ hover / selected / disabled  state (literal per-mode, or composited if tooling allows)
│
├─ chrome        purpose-anchored — never mechanically inverted (§2)
│   ├─ backdrop
│   ├─ scrim           authored per mode directly
│   └─ translucent / transparent
│
└─ shadow        elevation/lighting alpha ramp
    ├─ default   for current theme
    └─ inverse   for a locally-inverted island
```

| Token purpose | Inverts with theme? | What the `default`/`inverse` axis means |
|---|---|---|
| `contrast.fg` / `contrast.bg` | **Yes** — literal swap | light theme ↔ dark theme |
| `chrome.scrim` / `chrome.backdrop` | **No** — authored per mode | n/a — always "dim/recede," direction doesn't flip |
| `shadow.*` | Partially | `default` = matches current theme, `inverse` = for a locally-inverted island composed within it (not theme polarity) |
| `intent.*` | Yes, same hue family shifts lightness per mode | same as `contrast` |
| `action.main/optional` | Yes (brand colour adapts) | same as `contrast`; states (`hover`/`selected`/`disabled`) don't carry the axis at all |
| `action.auxiliary` | Yes, but deliberately *not* brand-coloured | aliases the neutral muted text tone, not the brand hue — see §1a |

Full rationale for each row below.

## 1. Give every family the same role vocabulary

A common failure mode: each token group invents its own naming for the same underlying concept. E.g. a "text meaning" group uses `positive/negative/info/caution/severe`, while a "button" group separately invents `confirm/oppose/danger` for the exact same ideas. Two vocabularies for one concept means every new consumer has to learn both and guess which applies where.

Fix: pick **one** meaning-vocabulary (call it `intent`, or whatever fits the product's voice — it doesn't have to be the generic success/warning/danger set if that reads as too literal for the brand). Anything needing semantic colour — text, icon, banner, **or** a button that needs to read as confirm/destructive — consumes that single vocabulary directly, rather than each family re-deriving its own copy.

Keep a *separate* family for pure visual hierarchy (primary/secondary emphasis) that carries no semantic meaning of its own — a "primary button" isn't inherently positive or negative, it's just the default action. Don't let hierarchy tokens and meaning tokens collapse into the same list; they answer different questions ("how prominent" vs "what does this mean").

### 1a. Two buttons isn't the whole hierarchy — plan for a third, low-commitment tier

A common layout gets missed if the hierarchy family only has two levels: a dialog with **Agree** / **Cancel** buttons plus a supporting **Learn more** link. That link isn't secondary, it's a different kind of action entirely — non-blocking, doesn't compete for the user's decision, usually rendered without any button container at all.

Give the hierarchy family three tiers, not two:

- **Primary** (`main`) — the committing action, filled/high-weight.
- **Secondary** (`optional`) — the alternate/dismissing action, lower weight but still a real choice.
- **Tertiary** (`auxiliary`) — supporting, non-blocking, lowest visual weight (text/link style).

Resist colouring the third tier with the same brand hue as the first two at full strength — if "Learn more" is exactly as vivid as "Agree," it competes with the actual decision instead of receding from it. Alias it to the neutral muted-emphasis tone from the alpha ramp (§3) instead, and let the component's *style* (underline, no fill/border) do the work of signalling "this is a link," not its colour.

### 1b. A colour used as a filled surface needs its own paired "content colour" — and it isn't one constant

Any hierarchy or meaning colour that can double as a *filled* surface (a solid primary button, a tinted alert banner) needs a partner token for whatever text/icon sits on top of that fill. It's tempting to solve this once with a single shared "on-accent" colour (e.g. "always white text on colour"). Don't — verify it with actual contrast maths first. Two things break that assumption:

- **The right neutral flips per hue, not just per theme mode.** A saturated, darker hue (blue, red, orange at typical "600"-style depth) wants a light/paper label. A hue that's inherently lighter even at its "dark" step (yellow, cyan) can flip the other way — a paper label sitting on a light yellow fill is often *worse* than a near-black one. Compute both and pick per hue, don't assume one direction for the whole family.
- **Some hues fail against both neutrals.** At a given hue's chosen lightness, it's possible for neither white nor black text to clear a real threshold (e.g. 4.5:1) against it — the fill itself is the problem, not the label choice. When that happens, no amount of choosing between the two available neutrals fixes it; the fill's lightness needs adjusting (same technique as §5's compliance-mode fixes), or that hue shouldn't be used as a filled surface at all.

Only tokens that are genuinely used as fills need this pairing — a hierarchy tier that's a ghost/outline/link style (no container) never needs one, since its colour is already functioning as content-on-page, not content-on-fill.

## 2. Solve the "does this axis actually mean theme, or something else" question up front

A `default`/`inverse` (or `light`/`dark`) axis on a colour family can silently mean two different things, and conflating them causes real bugs:

- **Theme polarity** — literally, "this is light mode" vs "this is dark mode." Foreground/background pairs should swap here; that's the entire point.
- **Local composition context** — "this token is for content matching the current page theme" vs "this token is for a locally-inverted island composed within it" (a dark card dropped on a light page, or vice versa). This is a legitimate and useful axis, but it is **not** the same thing as theme polarity, even though it often uses the same `default`/`inverse` naming.

Before building automated dark-mode generation (a script that inverts a light-mode file to produce dark mode), work out which of these two meanings each family actually needs. Blind lightness-inversion is *correct* for foreground/background contrast pairs but is *wrong* for anything purpose-anchored to "dim/recede" — a modal scrim, an overlay, a backdrop. A dark-mode scrim generated by mechanically inverting a light-mode scrim turns into a bright white overlay, which defeats the reason a scrim exists. Purpose-anchored roles like this must be **authored directly per mode**, never derived by formula from the other mode.

Rule of thumb: if a token's job is "provide contrast against content," it can invert. If a token's job is "always recede/dim/obscure," it cannot — author it per mode explicitly, and document that constraint on the token itself so a future contributor (or script) doesn't "simplify" it back into a formula.

## 3. Prefer an alpha ramp over hand-picked solid emphasis stops

Instead of naming three separate solid colours for "primary/secondary/tertiary" text emphasis, define one base colour and express weaker emphasis as that same colour at reduced alpha (e.g. 100% / 72% / 46%). Benefits:

- Adding a new emphasis level is a new alpha value, not a new named token that needs its own alias resolution across every theme mode.
- Borders, dividers, and small decorative strokes can pull from the same ramp (very low alpha, e.g. 10–20%) instead of needing a dedicated "border colour" family.
- It composes correctly against different backgrounds automatically, since it's a function of the base colour rather than a separately-tuned solid.

This only requires the base colour to be legible at each alpha step against its expected background — verify contrast at the lowest step you intend to use for anything carrying real information (not just decoration).

## 4. Keep "emphasis" and "state" as separate axes

Two things get inconsistently mixed into one flat list otherwise:

- **Emphasis (alpha-based)** — secondary/tertiary text, subtle borders. Always available, same hue, reduced opacity.
- **State (interaction)** — hover, selected, disabled. These usually need to stay as literal per-mode values if the tooling can't composite colours at build/reference time (most design-tool variable systems can't evaluate `color-mix()`-style operations). Where the tooling *can* compute a composite (a build pipeline transform, or CSS `color-mix()` at consume time), prefer deriving state colours from the same alpha-ramp approach rather than hand-picking a new hue per state — it's one less thing to keep in sync across modes.

Don't let a single family (e.g. "action") hold both meaning-colours and state-modifiers in one flat namespace — separate them so each axis can evolve independently.

## 5. Contrast-compliance modes (WCAG/APCA/etc.) can exist as structural placeholders

If a system needs to support contrast-driven modes before the actual compliant values are worked out, scaffold them with the identical shape as the reference mode, but mark them explicitly as pending/unset rather than leaving them as an undocumented copy. That keeps the eventual work a value-swap, not a restructure, and prevents an unset placeholder from being silently treated as production-ready.

## 6. Spacing and type should share one grid, derived from type — not defined independently

The classic Müller-Brockmann grid axioms, translated into token terms:

1. **Type sets the module.** Pick a base unit from the body type's rendered line height (font-size × line-height), not an arbitrary pixel value. Every other spacing value should be a multiple or fraction of that unit.
2. **Baseline grid governs vertical placement.** Any element's height should snap to a whole multiple of the base unit, so mixed content (text, images, cards) lines up across a layout regardless of what's in each column.
3. **Proportional structure, not eyeballing.** Column widths, gutters, and spacing scales should derive from the same base unit via a chosen ratio, reproducibly — not tuned ad hoc per component.
4. **White space is a first-class module**, sized with the same discipline as content blocks, not treated as leftover space.

Practically: define spacing tokens as references into the typography system's line-height (which itself should be quantized to a small atomic grid unit, e.g. 4px), rather than maintaining spacing and type as two independently-tuned scales that happen to look similar today and drift apart later.

Separately, the **type-size scale itself** should be internally ratio-consistent (pick one ratio — a musical-interval-style ratio like 1.25 or 1.333 is a reasonable default — and generate sizes from it), rather than being a list of sizes chosen individually per use case. It's fine for "display" sizes to break to a second, steeper ratio once you cross into headline/banner territory — that's normal type-system practice — as long as each size's line-height still quantizes back onto the shared baseline grid from principle 2.

## 7. Metadata and tooling quirks are a separate concern from token structure

Design-tool export metadata (scopes, resolved types, plugin-specific extensions) belongs alongside the core token value but shouldn't drive the naming or structure decisions above — treat it as a serialization detail of whatever export pipeline produces the file, not part of the token model itself. Likewise, path/reference fragility introduced by an export tool (e.g. spaces or special characters in alias paths) is a tooling bug to fix at the source, not something to work around in the token structure.
