# perf: memoize ThemeContext provider value to decouple theme consumers from dimension changes
- **Repository:** RocketChat/Rocket.Chat.ReactNative
- **PR Link:** <https://github.com/RocketChat/Rocket.Chat.ReactNative/pull/7420>
- **Merged Date:** 2026-06-23
- **Category:** Rendering

## Problem
The root `App` built the `ThemeContext` provider value as an inline object literal, so its identity changed on every `App` render. `App` state also holds window dimensions updated on `Dimensions.change`, so every rotation or fold/multi-window resize invalidated `ThemeContext` for all consumers (about 315 `useTheme()` and 15 `withTheme()` call sites) although the theme was unchanged. On Android emulator rotation, the React DevTools profiler showed `ThemeContext` invalidations falling from 1405 to 923 (-482).

## Solution
- Extract a `ThemeContextProvider` wrapper that builds the value with `useMemo` keyed on `theme`, `themePreferences` and `setTheme`.
- `setTheme` is a stable class property and `colors[theme]` is a module constant, so the wrapper object was the only unstable input.
- `DimensionsContext` is deliberately not memoized because its value must change on resize.
- A unit test asserts the value reference stays stable across parent re-renders and changes when `theme` changes.

## Code Pattern (Before vs. After)
```tsx
// Before
<ThemeContext.Provider
	value={{
		theme,
		themePreferences,
		setTheme: this.setTheme,
		colors: colors[theme]
	}}>
	<ResponsiveLayoutProvider>...</ResponsiveLayoutProvider>
</ThemeContext.Provider>

// After
const ThemeContextProvider = ({ theme, themePreferences, setTheme, children }) => {
	const value = useMemo<IThemeContextProps>(
		() => ({ theme, themePreferences, setTheme, colors: colors[theme] }),
		[theme, themePreferences, setTheme]
	);
	return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
};

<ThemeContextProvider theme={theme} themePreferences={themePreferences} setTheme={this.setTheme}>
	<ResponsiveLayoutProvider>...</ResponsiveLayoutProvider>
</ThemeContextProvider>
```

## Takeaway
Never pass an object literal as a Context `value` in a component that re-renders for unrelated state; memoize it or move the provider into its own component.
