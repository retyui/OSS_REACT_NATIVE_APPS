# [No QA] Enable 120 fps on iOS
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/20348>
- **Merged Date:** 2023-06-07
- **Category:** Animations (+ Native)

## Problem
On iPhones with ProMotion displays, iOS caps app rendering at 60 fps unless the app opts in. The PR notes the flag is only necessary on iPhone 13 Pro and 13 Pro Max, because the app already seemed to run at 120 fps on iPhone 14 Pro. No measurements are included.

## Solution
- Set `CADisableMinimumFrameDurationOnPhone` to `true` in `ios/NewExpensify/Info.plist`. This lets Core Animation drive the app above 60 fps on supporting iPhones, so animations and scrolling can run at the display's higher refresh rate.
- It is a native configuration change with no JS changes. The PR references Reanimated and Expo PRs and Apple's ProMotion documentation.

## Code Pattern (Before vs. After)
```xml
<!-- Before -->
	<key>UIViewControllerBasedStatusBarAppearance</key>
	<false/>
</dict>

<!-- After -->
	<key>UIViewControllerBasedStatusBarAppearance</key>
	<false/>
	<key>CADisableMinimumFrameDurationOnPhone</key>
	<true/>
</dict>
```

## Takeaway
Add `CADisableMinimumFrameDurationOnPhone` to the iOS `Info.plist` so animation-heavy apps can use 120 Hz on ProMotion iPhones that otherwise stay at 60 fps.
