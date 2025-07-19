# Split Peripheral Build Fix

## Issue
GitHub Actions builds for the left keyboard (roBa_L) were failing with undefined reference errors:
```
undefined reference to `zmk_keymap_highest_layer_active'
undefined reference to `zmk_ble_active_profile_index'
```

## Root Cause
In ZMK split keyboards, the peripheral side (left keyboard) doesn't have access to certain functions that are only available on the central side:
- `zmk_keymap_highest_layer_active()` - Keymap layer information is managed by the central
- `zmk_ble_active_profile_index()` - BLE profile management is handled by the central

## Solution
Added conditional compilation guards in `status_advertisement.c` to check if the code is running on:
1. A non-split keyboard, OR
2. The central side of a split keyboard

### Code Changes
```c
// Get active layer (only available on central or non-split keyboards)
#if !IS_ENABLED(CONFIG_ZMK_SPLIT) || IS_ENABLED(CONFIG_ZMK_SPLIT_ROLE_CENTRAL)
    uint8_t layer = zmk_keymap_highest_layer_active();
    adv_data.active_layer = layer;
#else
    // Split peripheral doesn't have access to keymap layer information
    adv_data.active_layer = 0;
#endif

// Get active profile (only available on central or non-split keyboards)
#if !IS_ENABLED(CONFIG_ZMK_SPLIT) || IS_ENABLED(CONFIG_ZMK_SPLIT_ROLE_CENTRAL)
    uint8_t profile = zmk_ble_active_profile_index();
    adv_data.profile_slot = profile;
#else
    // Split peripheral doesn't have access to BLE profile information
    adv_data.profile_slot = 0;
#endif
```

## Test Results
- ✅ Left keyboard (peripheral) builds successfully locally
- ✅ Build process completes without undefined reference errors
- ✅ Conditional guards ensure proper fallback values for split peripherals

## Behavior
- **Non-split keyboards**: Full functionality with actual layer and profile data
- **Split central**: Full functionality with actual layer and profile data  
- **Split peripheral**: Uses fallback values (layer 0, profile 0) but builds and functions correctly

## Files Modified
- `modules/prospector-zmk-module/src/status_advertisement.c`

The fix is also available as a patch file: `split-peripheral-fix.patch`