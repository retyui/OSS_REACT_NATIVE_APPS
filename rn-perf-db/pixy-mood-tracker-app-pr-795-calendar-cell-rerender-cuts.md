# perf(calendar): cut re-renders and formatter work in calendar cells
- **Repository:** mrzmyr/pixy-mood-tracker-app
- **PR Link:** <https://github.com/mrzmyr/pixy-mood-tracker-app/pull/795>
- **Merged Date:** 2026-10-18
- **Category:** Rendering (+ Lists)

## Problem
The calendar screen rendered one `CalendarDay` per cell, and several things defeated memoization. The filters provider ran `useContentStableValue` (a `JSON.stringify`) over every filtered entry on each render. Opening the filter sheet changed the same context as the data, so every week and timeline row re-rendered. Each day received a fresh `onPress` closure, subscribed to `scaleType` on its own, and built a new `Intl.DateTimeFormat` for its accessibility label. The PR reports no before/after timings; it is described as no visual change.

## Solution
- Split the filters context: sheet visibility (`isOpen/open/close`) moved to its own `CalendarFiltersSheetContext`, so toggling the sheet no longer re-renders cells.
- Content-compare only the small `filters` input instead of all matched entries, and skip matching entirely when no filter is active.
- Compute `filteredDates` (a `Set` of days) once in the provider instead of building an id `Set` per week.
- Pass one stable `onPressDay(dateString)` to all days so `memo(CalendarDay)` can bail out.
- Read `scaleType` once per week and pass it down as a prop.
- Replace per-cell `new Intl.DateTimeFormat` with a cached `getFormatter("day_label_long", ...)`.

## Code Pattern (Before vs. After)
```tsx
// Before
const filteredItemIds = useMemo(
  () => new Set(calendarFilters.data.filteredItems.map((item) => item.id)),
  [calendarFilters.data.filteredItems]
);

const renderDay = ({ date }) => {
  const isFiltered = items.some((item) => filteredItemIds.has(item.id));
  return (
    <CalendarDay
      dateString={date}
      isFiltered={isFiltered}
      onPress={() => onPressDay(date)}
    />
  );
};

// inside CalendarDay
const scaleType = useSetting("scaleType");
const date = new Intl.DateTimeFormat(getLocale(), {
  weekday: "long", year: "numeric", month: "long", day: "numeric",
}).format(dayjs(dateString).toDate());

// After
const scaleType = useSetting("scaleType"); // once per week
const renderDay = ({ date }) => (
  <CalendarDay
    dateString={date}
    scaleType={scaleType}
    isFiltered={filters.filteredDates.has(date)}
    onPress={onPressDay} // CalendarDay calls onPress(dateString)
  />
);

// inside CalendarDay
const date = getFormatter("day_label_long", DAY_LABEL_OPTIONS).format(
  dayjs(dateString).toDate()
);
```

## Takeaway
For repeated list cells, pass one stable handler that receives the item key, hoist per-cell subscriptions to the parent, and cache `Intl` formatters. Keep UI toggles in a separate context from data.
