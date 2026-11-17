# perf: Avatar missing selector comparison
- **Repository:** RocketChat/Rocket.Chat.ReactNative
- **PR Link:** <https://github.com/RocketChat/Rocket.Chat.ReactNative/pull/6498>
- **Merged Date:** 2025-07-23
- **Category:** Rendering

## Problem
`AvatarContainer` used a `useSelector` that returned a fresh object (`{ avatarExternalProviderUrl, roomAvatarExternalProviderUrl, cdnPrefix }`) on every store update. Redux compares selector results by reference, so each unrelated store change re-rendered every avatar. The PR shows before/after "highlight updates" screenshots (changing the `create-p` permission in the admin panel) and notes the difference is small but visible; no timings are reported.

## Solution
- Pass `shallowEqual` as the equality function to the multi-value selector, so a new object with identical fields does not trigger a re-render.
- Switch the other selectors in the file to the typed `useAppSelector` hook (no behavioral change).
- The author calls it a quick win with more left to improve.

## Code Pattern (Before vs. After)
```tsx
// Before
const { avatarExternalProviderUrl, roomAvatarExternalProviderUrl, cdnPrefix } = useSelector((state: IApplicationState) => ({
	avatarExternalProviderUrl: state.settings.Accounts_AvatarExternalProviderUrl as string,
	roomAvatarExternalProviderUrl: state.settings.Accounts_RoomAvatarExternalProviderUrl as string,
	cdnPrefix: state.settings.CDN_PREFIX as string
}));

// After
const { avatarExternalProviderUrl, roomAvatarExternalProviderUrl, cdnPrefix } = useAppSelector(
	state => ({
		avatarExternalProviderUrl: state.settings.Accounts_AvatarExternalProviderUrl as string,
		roomAvatarExternalProviderUrl: state.settings.Accounts_RoomAvatarExternalProviderUrl as string,
		cdnPrefix: state.settings.CDN_PREFIX as string
	}),
	shallowEqual
);
```

## Takeaway
Any `useSelector` that builds an object or array literal needs `shallowEqual` (or one selector per field), especially in leaf components rendered hundreds of times.
