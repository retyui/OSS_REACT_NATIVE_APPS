# Improve textinput performance in login and account creation
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/4673>
- **Merged Date:** 2024-07-02
- **Category:** Rendering

## Problem
The login and account-creation forms showed visible flicker and input lag, because every keystroke updated React state held in controlled `TextInput`s (and account creation re-rendered on every value change to compute whether "Next" was enabled). No numeric measurements are given; it was tested on three platforms and on-device on an iPhone.

## Solution
- Login form: switch to uncontrolled inputs. Values are stored in refs (`useRef`) and `defaultValue` replaces `value`; a small `checkIsReady()` only calls `setIsReady` when the ready boolean actually changes.
- Identifier lowercasing/trimming moves from each keystroke to submit time.
- Account creation: move validation into the "Next" button handler and remove the "can next" logic, so value changes no longer trigger renders. Back/Next buttons were extracted into a shared `BackNextButtons` component.
- Side effect: the password accessibility hint no longer includes the identifier.

## Code Pattern (Before vs. After)
```tsx
// Before
const [identifier, setIdentifier] = useState<string>(initialHandle)
const [password, setPassword] = useState<string>('')
// ...
value={identifier}
onChangeText={str => setIdentifier((str || '').toLowerCase().trim())}
// ...
value={password}
onChangeText={setPassword}
// ...
const isReady = !!serviceDescription && !!identifier && !!password

// After
const identifierValueRef = useRef<string>(initialHandle || '')
const passwordValueRef = useRef<string>('')
// ...
defaultValue={initialHandle || ''}
onChangeText={v => {
  identifierValueRef.current = v
  checkIsReady()
}}
// ...
onChangeText={v => {
  passwordValueRef.current = v
  checkIsReady()
}}
// ...
const identifier = identifierValueRef.current.toLowerCase().trim()
const password = passwordValueRef.current
```

## Takeaway
For forms, keep text in refs and set React state only when a derived flag (like "can submit") changes.
