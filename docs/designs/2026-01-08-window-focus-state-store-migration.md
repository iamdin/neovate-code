# Window Focus State Store Migration

**Date:** 2026-01-08

## Context

The cursor in `ChatInput.tsx` sometimes disappears after `AskQuestionModal` completes. The issue is intermittent - it happens when users switch terminal windows while the modal is open.

The current implementation uses a `useWindowFocus` hook in `ChatInput.tsx` that relies on `TextInput`'s `onFocusChange` callback to track terminal focus state. The cursor visibility is controlled by `showCursor={isWindowFocused}`.

## Discussion

**Root Cause Analysis:**

1. Terminal focus events are sent as escape sequences: `\x1b[I` (focus gained) and `\x1b[O` (focus lost)
2. Ink strips the `\x1b` prefix, leaving `[I` and `[O` which are intercepted in `TextInput`'s `useInput` handler
3. When `AskQuestionModal` is active, it uses `useInput` with `{ isActive: true }`, capturing ALL input including focus events
4. Focus events go to the modal's input handler instead of `TextInput`, so `isWindowFocused` never updates
5. When the modal closes and user had switched away/back, `ChatInput` still has stale `isWindowFocused: false`

**Why "sometimes":**
- If user doesn't switch terminal windows while modal is open → works fine
- If user switches away and back while modal is open → `[I` event goes to modal, cursor stays hidden

**Options Considered:**

1. **Option A**: Move focus tracking to a global level in `App.tsx` - cleanest but requires more refactoring
2. **Option B**: Have `AskQuestionModal` handle focus events and forward to shared state
3. **Option C**: Force refresh `isWindowFocused` to `true` when modal unmounts - simple workaround

## Approach

Move `isWindowFocused` state to the Zustand store (`store.ts`). This makes focus state globally accessible and updatable from any component that captures input.

## Architecture

### Changes Required

**1. `src/ui/store.ts`**
- Add state: `isWindowFocused: boolean` (default `true`)
- Add action: `setWindowFocused: (focused: boolean) => void`

**2. `src/ui/TextInput/index.tsx`**
- Update focus event handler to use store directly:
```typescript
if (input === '[I' || input === '[O') {
  useAppStore.getState().setWindowFocused(input === '[I');
  return;
}
```
- Remove `onFocusChange` prop usage (can keep prop for backward compatibility)

**3. `src/ui/ChatInput.tsx`**
- Remove `useWindowFocus` hook import and usage
- Read `isWindowFocused` from `useAppStore()` selector
- Remove `onFocusChange={handleFocusChange}` prop from `TextInput`

**4. `src/ui/useWindowFocus.ts`**
- Keep only the terminal escape sequence setup effect (`\x1b[?1004h` / `\x1b[?1004l`)
- Move this effect to `App.tsx` or keep hook for just the escape sequence registration
- Remove state management (now handled by store)

**5. `src/ui/AskQuestionModal.tsx`** (and any other modal with `useInput`)
- Add focus event handling in `useInput` callback:
```typescript
useInput((input, key) => {
  if (input === '[I' || input === '[O') {
    useAppStore.getState().setWindowFocused(input === '[I');
    return;
  }
  // ... existing logic
}, { isActive: true });
```

### Flow After Migration

1. User opens terminal → escape sequence `\x1b[?1004h` enables focus reporting
2. Any active `useInput` handler (ChatInput, AskQuestionModal, etc.) receives `[I`/`[O`
3. Handler updates store via `setWindowFocused()`
4. `ChatInput` reads `isWindowFocused` from store, cursor visibility updates correctly
5. Focus state persists correctly across modal open/close cycles
