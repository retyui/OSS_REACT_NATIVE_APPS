# perf(react-compiler): stop keying memo scopes on the whole props object
- **Repository:** keybase/client
- **PR Link:** <https://github.com/keybase/client/pull/29513>
- **Merged Date:** 2026-08-07
- **Category:** Rendering

## Problem
Checking the React Compiler's real output showed that reading `props.x` inside a callback makes the memo scope depend on the whole `props` object. JSX creates a new props object every parent render, so that guard always fails, the cache never hits, and the callback gets a new identity each render, cascading into children. A destructure placed below such a callback had the same effect (this is why `SimpleMarkdownComponent` re-parsed markdown every render). The PR reports no timings; 32 components were affected.

## Solution
- Destructure every prop once at the top of the component, above all callbacks, so the compiler emits fine-grained guards (`$[n] !== checked`).
- Alias props where a local shadows a name (`onReact: _onReact`).
- Fix 32 components, including Checkbox, CheckCircle, InlineDropdown, SearchFilter, CopyText, menu layout, SuggestionList and ReactButton; `Box2` is left alone since it truly needs the whole object via `...rest`.
- Extend the `react-compiler-bailouts` lint script to flag whole-props deps (baseline 0) so it cannot regress.

## Code Pattern (Before vs. After)
```tsx
// Before
const Checkbox = (props: Props) => {
  if (!isMobile) {
    return (
      <Kb.ClickableBox
        onClick={e =>
          props.disabled || e?.defaultPrevented ? undefined : props.onCheck?.(!props.checked)
        }
      >

// After
const Checkbox = (props: Props) => {
  const {checkboxColor, checkboxStyle, checked, disabled, label, labelComponent,
    labelSubtitle, labelType, onCheck, style} = props

  if (!isMobile) {
    return (
      <Kb.ClickableBox
        onClick={e =>
          disabled || e?.defaultPrevented ? undefined : onCheck?.(!checked)
        }
      >
```

## Takeaway
With the React Compiler, destructure props at the top and never read `props.x` inside a callback; check compiled output for `$[n] !== props` guards.
