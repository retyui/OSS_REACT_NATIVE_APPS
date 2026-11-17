# fix(city-guide): stop the featured pill glow from freezing the app
- **Repository:** artsy/eigen
- **PR Link:** <https://github.com/artsy/eigen/pull/14254>
- **Merged Date:** 2026-10-01
- **Category:** Animations (+ Rendering)

## Problem
A rotating glow on a featured navigation pill animated `strokeDashoffset` on nine SVG rects every frame. On Fabric, Reanimated 4 can only update a small set of props (transform, opacity, colours) without a shadow-tree commit. SVG props are not in that set, so each frame forced nine commits, which starved React's own commits and froze the UI on the simulator. The PR reports no numeric before/after.

## Solution
- The glow now animates only `transform: rotate`, which runs on the UI thread with no commits.
- The comet is a static SVG disc of colour wedges (same nine colours and proportions) spinning behind the pill. A rounded `overflow: hidden` container plus an inner cover view leaves only a 1px ring visible.
- A worklet computes where the comet edge would be at constant speed along the pill outline and sets the rotation angle accordingly, so the speed looks even around the rounded ends.
- The glow is rendered before the pill and extends 1px past its edge, because `Pill` paints its background on an internal pressable.
- Visual caveats: the ring sits 1px outside the pill, and the tail looks slightly shorter on straight edges of wide pills.

## Code Pattern (Before vs. After)
```tsx
// Before
const AnimatedRect = Animated.createAnimatedComponent(Rect)
// ...
const animatedProps = useAnimatedProps(() => ({
  strokeDashoffset: offset.get(),
}))
// ... nine of these, one per ring
{RINGS.map((ring) => (
  <AnimatedRect key={ring.color} /* ... */ stroke={ring.color} animatedProps={animatedProps} />
))}

// After
const rotatingStyle = useAnimatedStyle(() => {
  // ...compute x, y of the comet edge along the pill outline from progress.get()
  return { transform: [{ rotate: `${Math.atan2(y, x)}rad` }] }
})

<Animated.View style={[{ position: "absolute", width: diameter, height: diameter }, rotatingStyle]}>
  <Svg width={diameter} height={diameter}>
    {BANDS.map((band) => (
      <Path key={band.color} d={wedgePath(center, band.from, band.to)} fill={band.color} />
    ))}
  </Svg>
</Animated.View>
```

## Takeaway
On Fabric, animate only transform/opacity/colour props with Reanimated; animating SVG or layout props commits the shadow tree each frame. Rotate a static asset instead.
