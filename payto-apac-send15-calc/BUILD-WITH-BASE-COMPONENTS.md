# "Top rate" label — build & animate with base-components

How to add the animated **"Top rate"** benefit label (flame + text that fades in, expands open,
then gently zoom-pulses) using Remitly **base-components** (`@remitly/base-components`, React
Native + react-native-reanimated). Scoped to this one element only.

> Verified against the library source at `/Users/konstantinosp/code/narwhal/packages/base-components`
> and the spring presets at `src/libs/animationConstants.ts`. Props quoted below are real.
>
> Live prototype: https://dansoares26.github.io/design-prototypes/payto-apac-send15-calc/
> Handoff files: `payto-apac-send15-calc/handoff/src/components/topRateTag/`

---

## 1. The component

Use the **`Tag`** component with the `benefit` type:

```ts
import { Tag } from "@remitly/base-components";
```

| Prop | Value | Note |
|------|-------|------|
| `type` | `"benefit"` | the teal benefit pill style |
| `description` | `"Top rate"` | the text label |
| `iconName` | a benefit/flame icon id | static-icon fallback if you don't use a Lottie flame (see §5) |
| `uppercase` | `{false}` | **required** — `Tag` defaults to `uppercase: true`; "Top rate" is sentence case |
| `textStylePreset` | `"tagT…"` | optional; pick the tag text size that matches the design |

`Tag` itself is **static** — it has no entrance or pulse animation. The motion is added by wrapping
it (or its parts) in reanimated views, below.

---

## 2. The animation, in three parts

Target behavior (matches the refined prototype):

```
0ms       Mount — tag shows flame icon only, text hidden (width 0, opacity 0)
1000ms    Text starts expanding (width 0 → measured, ease-out, 900ms)
1300ms    Text fades in (opacity 0 → 1, 600ms — starts 300ms after expand)
2100ms    One-time scale pulse (1.0 → 1.05 → 1.0, 500ms) — subtle attention nudge, fires once only
```

### Timing constants

| Constant | Value | Description |
|---|---|---|
| `REVEAL_DELAY_MS` | 1000ms | Delay before text reveal starts |
| `EXPAND_DURATION_MS` | 900ms | Width expand duration |
| `FADE_DELAY_MS` | 300ms | Opacity starts this long after expand |
| `FADE_DURATION_MS` | 600ms | Opacity transition duration |
| `PULSE_DELAY_MS` | 200ms | Delay after reveal before pulse |
| `PULSE_DURATION_MS` | 500ms | Scale pulse duration (one-time only) |
| `TEXT_MAX_WIDTH` | 58pt | Measured width of "Top rate" label |
| `LOOP_PAUSE_MS` | 2000ms | Pause between fire animation loops |

### 2.1 The springs we use (`src/libs/animationConstants.ts`)
`MOVEMENT_SPRING_CONFIGS`:

| Preset | stiffness / damping / mass | ζ (damping ratio) | Bounce |
|--------|----------------------------|-------------------|--------|
| `MOVE_IN_SMALL`  | 815 / 48 / 1   | 0.84 | minimal — quickest settle |
| `MOVE_IN_MEDIUM` | 504 / 38 / 1   | 0.85 | light ← **use this** |
| `MOVE_IN_LARGE`  | 342 / 31 / 1.3 | 0.74 | bounciest |
| `MOVE_OUT_SMALL` | 1218 / 69 / 1  | 0.99 | none — exits |
| `MOVE_OUT_MEDIUM`| 987 / 62 / 1   | 0.99 | none |
| `MOVE_OUT_LARGE` | 816 / 57 / 1   | 1.0  | none |

Codebase convention: **`MOVE_IN_*` to enter/expand** (overshoot allowed), **`overshootClamping: true`
to collapse** (no bounce). That gives "bounce on the way up, clean on the way down." We picked the
lighter `MOVE_IN_MEDIUM`.

> **RN bonus:** a `scale` transform on a `View` originates from its **center** by default, so the
> "scales rightward toward the border" issue we fixed in CSS (`transform-origin: center`) does
> **not** arise in RN — the pulse is naturally centered.

### 2.2 Entrance — reveal text after 1s, then open

```tsx
const textWidth   = useSharedValue(0);  // label width animates 0 → measured
const textOpacity = useSharedValue(0);  // label fades in after expand starts

useEffect(() => {
  if (reduceMotion) {
    textWidth.value   = CONTENT_W;
    textOpacity.value = 1;
    return;
  }
  // Start expand after 1s
  textWidth.value = withDelay(
    REVEAL_DELAY_MS,
    withSpring(CONTENT_W, MOVE_IN_MEDIUM),
  );
  // Fade in 300ms after expand starts
  textOpacity.value = withDelay(
    REVEAL_DELAY_MS + FADE_DELAY_MS,
    withTiming(1, { duration: FADE_DURATION_MS }),
  );
}, [reduceMotion]);

const textStyle = useAnimatedStyle(() => ({
  width: textWidth.value,
  opacity: textOpacity.value,
  overflow: "hidden",
}));
```

Measure the text width once via `onLayout` and animate `width` 0 → measured. Animating the real
width (not a fixed `maxWidth`) avoids clipping and stays correct when the font scales (see §4).

### 2.3 One-time zoom pulse (fires once after reveal settles, not on repeat)

The prototype uses a **single** pulse — scale 1.0 → 1.05 → 1.0 — as a subtle nudge the moment
the text finishes appearing. It does **not** repeat.

```tsx
const scale = useSharedValue(1);

// Trigger once, ~200ms after reveal settles
useEffect(() => {
  if (reduceMotion) return;
  const timeout = setTimeout(() => {
    scale.value = withSequence(
      withSpring(1.05, MOVE_IN_MEDIUM),
      withSpring(1, { ...MOVE_IN_MEDIUM, overshootClamping: true }),
    );
  }, REVEAL_DELAY_MS + EXPAND_DURATION_MS + PULSE_DELAY_MS);
  return () => clearTimeout(timeout);
}, [reduceMotion]);

const pulseStyle = useAnimatedStyle(() => ({
  transform: [{ scale: scale.value }],
}));
```

### 2.4 Putting it together

```tsx
<View>                                              {/* pill shell */}
  <Tag
    type="benefit"
    uppercase={false}
    iconName={undefined}                            {/* flame rendered separately, see §5 */}
    description=""                                  {/* text rendered separately so it can open + pulse */}
  />
  {/* flame + animated text live inside the pill: */}
  <Flame reduceMotion={reduceMotion} />             {/* §5 */}
  <Animated.View style={textStyle}>                 {/* opens left→right */}
    <Animated.View style={pulseStyle}>              {/* one-time pulse */}
      <Text stylePreset="tagT3">Top rate</Text>
    </Animated.View>
  </Animated.View>
</View>
```

> If you prefer to keep `Tag` intact, render `Tag` for the pill shell + flame, and animate only the
> "Top rate" `Text` you place inside it. Wrap **only the text** in the pulsing view (not the flame),
> matching the prototype.

---

## 3. Reduced motion (no hook exists — query it yourself)

base-components does **not** export a `useReducedMotion` hook. Query the OS setting:

```tsx
const [reduceMotion, setReduceMotion] = useState(false);
useEffect(() => {
  AccessibilityInfo.isReduceMotionEnabled().then(setReduceMotion);
  const sub = AccessibilityInfo.addEventListener("reduceMotionChanged", setReduceMotion);
  return () => sub.remove();
}, []);
```

When `reduceMotion` is true: skip the fade, the open, and the pulse — render the tag in its final
state (opacity 1, full width, scale 1) — and do **not** loop the flame (show a static frame / the
benefit icon). This mirrors the prototype's `@media (prefers-reduced-motion: reduce)` block.

---

## 4. Font scaling for the label

`Text`/`Body` scale with Dynamic Type by default. For this small `tagT` pill text:
- Test at large font scales; cap modestly with `maxFontSizeMultiplier` if it wraps or clips inside
  the pill.
- **Critical:** the animated open uses the text's measured width. Measure the **scaled** text
  (`onLayout` after render at the current font scale), not a hardcoded value — otherwise the label
  clips at large Dynamic Type settings. Don't put a fixed height/width on the pill.

---

## 5. The flame

The flame is a Lottie (`fire.lottie.json` — included in the handoff folder). In RN use `lottie-react-native`.

The flame plays **once**, pauses **2 seconds**, then replays — indefinitely. It does **not** use
`loop: true`. Instead, listen to `onAnimationFinish` and replay after the pause:

```tsx
function Flame({ reduceMotion }) {
  const [playKey, setPlayKey] = useState(0);

  const handleComplete = useCallback(() => {
    setTimeout(() => setPlayKey(k => k + 1), LOOP_PAUSE_MS); // 2000ms
  }, []);

  if (reduceMotion) {
    return <Icon name="…benefit-flame" size="small" accessible={false} />;
  }
  return (
    <LottieView
      key={playKey}
      source={flameJson}
      autoPlay
      loop={false}
      onAnimationFinish={handleComplete}
      style={{ width: 16, height: 16 }}
    />
  );
}
```

It is **decorative** — `accessible={false}` (and `aria-hidden` on web).

---

## 6. Accessibility for the label

- The flame is decorative; the **word "Top rate" carries the meaning** — never rely on the flame or
  color alone.
- Make the whole pill a single screen-reader stop so it's announced as one phrase, not fragmented:
  ```tsx
  <Animated.View style={pillStyle} accessible accessibilityLabel="Top rate">
  ```
  and mark the inner flame/text `accessible={false}` so they aren't focused separately.
- The pulse and fade are decorative motion — already gated behind reduced motion (§3).
- If the label sits next to the exchange-rate value, consider grouping both into one
  `accessibilityLabel` on their shared row (e.g. "Top rate. 1 AUD = 16,402 VND") so the benefit and
  the rate are read together.

---

## 7. Narwhal tokens

| Property | Token |
|---|---|
| Background | `backgroundActionSecondarySubtle` |
| Text colour | `textDefault` |
| Border radius | `full` |
| Horizontal padding | `space2` (8px) |
| Vertical padding | `space1` (4px) |
