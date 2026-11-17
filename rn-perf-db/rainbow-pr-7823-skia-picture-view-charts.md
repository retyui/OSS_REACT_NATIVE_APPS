# perf(charts): render through native Skia picture views
- **Repository:** rainbow-me/rainbow
- **PR Link:** <https://github.com/rainbow-me/rainbow/pull/7823>
- **Merged Date:** 2026-10-16
- **Category:** Rendering (+ Native)

## Problem
The Candlestick, Polymarket and Sparkline charts drew into Skia pictures held in Reanimated shared values (`chartPicture`, `crosshairPicture`, `indicatorPicture`) and rendered them through a JS-side `<Canvas>`/`<Picture>`, with drawing driven on the UI thread. The PR describes this as less efficient than publishing pictures directly to a native view. Chart managers also needed `useWorkletClass` plus per-chart cleanup hooks, and the manager built a full initial chart synchronously at construction. The PR gives no before/after timings.

## Solution
- Charts now render through the shared `SkiaPictureView` (added in #7816), which publishes Skia pictures straight to `SkiaPictureViewNativeComponent`.
- Manager creation and disposal moved into the shared view (`useSkiaRenderer({ deferred: true })` + `prepare`), removing `useWorkletClass` and the `useCleanup` boilerplate.
- Pictures are plain fields on the manager instead of shared values. Chart, indicator and crosshair layers are composited into one picture and published once via `setSkiaPicture`; a crosshair move only republishes when its picture changed.
- Small wins: line paths are reused for interaction markers and fills, and grid shaders are kept between draws. Unused paints and blank-picture placeholders were removed.
- `dispose()` now also cancels animations and resets gesture state. Sparkline charts now react to dimension changes.

## Code Pattern (Before vs. After)
```tsx
// Before
const chartPicture = useSharedValue(initialPicture);
const crosshairPicture = useSharedValue(initialPicture);
const indicatorPicture = useSharedValue(initialPicture);

const chartManager = useWorkletClass(() => {
  'worklet';
  return new CandlestickChartManager({ activeCandle, candles, chartPicture, crosshairPicture, indicatorPicture, /* ... */ });
}, true);

// in the manager
const oldPicture = this.crosshairPicture.value;
crosshairPicture.value = this.pictureRecorder.finishRecordingAsPicture();
oldPicture.dispose();

// After
const renderer = useSkiaRenderer<CandlestickChartManager>({ deferred: true });

<SkiaPictureView
  prepare={() => output => {
    'worklet';
    return new CandlestickChartManager({ activeCandle, output, /* ... */ });
  }}
  renderer={renderer}
  style={styles.canvas}
/>

// in the manager
private publishPicture(): void {
  const previousComposite = this.compositePicture;
  if (this.indicatorPicture || this.crosshairPicture) {
    const canvas = this.pictureRecorder.beginRecording();
    if (this.chartPicture) canvas.drawPicture(this.chartPicture);
    if (this.indicatorPicture) canvas.drawPicture(this.indicatorPicture);
    if (this.crosshairPicture) canvas.drawPicture(this.crosshairPicture);
    this.compositePicture = this.pictureRecorder.finishRecordingAsPicture();
  } else {
    this.compositePicture = undefined;
  }
  setSkiaPicture(this.output, this.compositePicture ?? this.chartPicture);
  previousComposite?.dispose();
}
```

## Takeaway
For frequently redrawn Skia content, record pictures in a worklet-owned manager and hand them to a native picture view instead of routing them through shared values and a JS-mounted `<Canvas>`.
