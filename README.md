# repro-rn-drawer-flash

Attempted minimal reproduction for a regression in `react-native-drawer-layout@4.2.2+` where an Android drawer with `drawerType="front"` briefly flashes open on mount when the app first renders.

**Issue:** https://github.com/react-navigation/react-navigation/issues/13107

## Versions

| Package | Version |
|---|---|
| `@react-navigation/drawer` | 7.8.1 (buggy — depends on `drawer-layout@^4.2.2`) |
| `@react-navigation/native` | 7.1.28 |
| `react-native-drawer-layout` | 4.2.3 (**contains the regression**) |
| `react-native` | 0.81.5 (Expo SDK 54, Old Architecture) |

## Steps to reproduce

1. Clone this repo
2. `npm install`
3. Run on an Android device or emulator: `npm run android` (this runs `expo run:android` — requires Android SDK)

**Note:** The flash does not reliably reproduce in this minimal app. The timing gap that causes the flash only manifests in apps with a heavier startup sequence, where Reanimated's worklet thread takes long enough to initialise that a visual frame is committed to screen before the animated styles are applied. In a minimal app, both happen within the same vsync interval.

The repo is useful for verifying a fix once one is applied: downgrading to `react-native-drawer-layout@4.2.1` (by pinning `@react-navigation/drawer` to `7.7.13`) should keep the drawer off-screen on first render.

## Expected behaviour

The drawer should not be visible on mount when in the closed state.

## Workaround

Pin `@react-navigation/drawer` to `7.7.13` (which depends on `react-native-drawer-layout@^4.2.1`):

```json
"@react-navigation/drawer": "7.7.13"
```

## Root cause

The regression was introduced in `react-native-drawer-layout@4.2.2` as a side-effect of the `drawerType="back"` zIndex fix in PR #12953.

**Before 4.2.2**, the drawer `Animated.View` had a static `zIndex` applied as part of the regular style array:

```js
style={[
  styles.drawer,
  {
    width: drawerWidth,
    position: 'absolute',
    zIndex: drawerType === 'back' ? -1 : 0,  // static — applied on first render
  },
  drawerAnimatedStyle,  // animated zIndex + transform — applied by Reanimated worklet
  drawerStyle,
]}
```

**After 4.2.2**, that static `zIndex` was removed. `zIndex` and `transform` are now set solely via `drawerAnimatedStyle` (`useAnimatedStyle`), which is applied by Reanimated's worklet thread — not synchronously on the first React render.

This creates a window on mount where neither `transform: translateX(-drawerWidth)` nor `zIndex` has been applied. During this window the drawer renders at position 0 (on screen) and, since it appears after the content view in the tree, paints on top of content. Once Reanimated commits, the drawer snaps off-screen — but the user sees a one-frame flash.

## Proposed fix

Restore the static `zIndex` to the drawer view in `Drawer.native.tsx`:

```js
style={[
  styles.drawer,
  {
    width: drawerWidth,
    position: 'absolute',
    zIndex: drawerType === 'back' ? -1 : 0,  // restore
  },
  drawerAnimatedStyle,
  drawerStyle,
]}
```

The animated `zIndex` in `drawerAnimatedStyle` overrides the static value once Reanimated commits (later entries in the style array take precedence), so this does not conflict with the `drawerType="back"` fix from #12953.
