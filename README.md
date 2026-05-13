# repro-rn-drawer-flash

Minimal reproduction for a regression in `@react-navigation/elements@2.8.5` where an Android drawer with `drawerType="front"` briefly flashes open on mount.

**Issue:** https://github.com/react-navigation/react-navigation/issues/13107

## Versions

| Package | Version |
|---|---|
| `@react-navigation/drawer` | 7.7.13 |
| `@react-navigation/elements` | 2.9.17 (resolved transitively — **contains the bug**) |
| `@react-navigation/native` | 7.1.28 |
| `react-native-drawer-layout` | 4.2.1 |
| `react-native` | 0.81.5 (Expo SDK 54) |

## Steps to reproduce

1. Clone this repo
2. `npm install`
3. Run on an Android device or emulator: `npm run android` (this runs `expo run:android` — requires Android SDK)
4. Observe the drawer flash briefly into view as the app mounts

## Expected behaviour

The drawer should not be visible on mount when in the closed state.

## Workaround

Add the following override to `package.json` and re-run `npm install`:

```json
"overrides": {
  "@react-navigation/elements": "2.8.3"
}
```

The flash disappears with `elements@2.8.3`.

## Root cause

`elements@2.8.5` replaced the synchronous `SafeAreaListener`-based frame measurement in `FrameSizeProvider` with an async `onLayout` + `useEffect.measure()` approach (PR #12882, fixing tvOS focus breakage). On Android, this creates a one-frame window where `useFrameSize()` returns zero dimensions. The drawer uses these to compute `drawerWidth`, which makes `getDrawerTranslationX(false)` return `0` (on-screen) instead of a negative off-screen offset. The drawer snaps to the correct position once `onLayout` fires, but the user sees the flash.
