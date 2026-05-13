# repro-rn-drawer-flash

Attempted minimal reproduction for a regression in `react-native-drawer-layout` where an Android drawer with `drawerType="front"` flashes open on mount.

Originally filed as https://github.com/react-navigation/react-navigation/issues/13107. A partial fix landed in `react-native-drawer-layout@4.2.4` / `@react-navigation/drawer@7.10.1`, but the issue persists in a different form — see below.

Follow-up issue: https://github.com/react-navigation/react-navigation/issues/13109

## Versions

| Package | Version |
|---|---|
| `@react-navigation/drawer` | 7.10.1 |
| `@react-navigation/native` | 7.2.4 |
| `react-native-drawer-layout` | 4.2.4 |
| `react-native` | 0.81.5 (Expo SDK 54, Old Architecture) |

## Steps to reproduce

1. Clone this repo
2. `npm install`
3. Run on Android: `npm run android` (requires Android SDK — runs `expo run:android`)

**Note:** The flash does not reliably reproduce in this minimal app. The timing gap only manifests in apps with a heavier startup sequence. The repo is useful for verifying a fix once applied.

To verify the original regression: pin `@react-navigation/drawer` to `7.7.13` (working) vs `7.10.1` (broken).

## Current behaviour (`drawer-layout@4.2.4`)

The drawer flashes open briefly and then **animates** to the closed position — previously it snapped instantly. The flash is still visible.

## Expected behaviour

The drawer should not be visible at all on mount when in the closed state.

## Root cause

The partial fix in `drawer-layout@4.2.4` introduced an `isHidden` shared value (initialised to `true` when `drawerWidth` is 0) and sets `display: none` via `drawerAnimatedStyle` until layout is measured. In a `useLayoutEffect`, once `drawerWidth` becomes valid, `translationX.value` is set to the off-screen position and `isHidden.value` is set to `false`.

However, these two shared value assignments are not committed atomically by Reanimated. There is a frame between them where the drawer becomes visible (`display: flex`) but `translationX` is still at 0 (on-screen). The `useEffect(() => toggleDrawer(open))` then fires and animates from position 0 to the off-screen position — which is the animation the user sees.

## Proposed fix

Initialise `translationX` using window dimensions as a fallback when `drawerWidth` is 0, so the position is never wrong regardless of Reanimated commit timing:

```js
const translationX = useSharedValue(
  drawerWidth > 0
    ? getDrawerTranslationX(open)
    : getDrawerTranslationX(open, windowDimensions.width)
);
```

Or alternatively restore the static `zIndex` to the drawer view so the drawer's z-ordering is established before Reanimated commits its first frame:

```js
style={[
  styles.drawer,
  { width: drawerWidth, position: 'absolute', zIndex: drawerType === 'back' ? -1 : 0 },
  drawerAnimatedStyle,
  drawerStyle,
]}
```

## Workaround

Pin `@react-navigation/drawer` to `7.7.13`:

```json
"@react-navigation/drawer": "7.7.13"
```
