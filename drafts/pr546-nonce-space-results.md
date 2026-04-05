@shufps here are our test results - you were right about the chip IDs being the key factor:

## Test Results (NerdQAxe++, 4x BM1370 @ 615MHz, SV2 Standard Channel)

### Test 1: HCN on register 0x10, original chip IDs (0,4,8,12)
**Result:** All 4 ASICs hash at full speed (~5 TH/s). V1 shares accepted, no regression. But in Standard Channel: first few unique shares accepted, then **duplicates within seconds**. Each nonce reported 4x simultaneously - all 4 chips find the same nonce because they search the same nonce space.

### Test 2: Same as Test 1 + disabled `checkVrFrequencyChanged` (was overwriting register 0x10)
**Result:** Same duplicate pattern. Confirmed the VR frequency overwrite was a problem, but HCN value alone doesn't partition the nonce space between chips.

### Test 3: HCN + Bitaxe-style chip IDs (0,64,128,192) + updated nonce-to-ASIC mapping
**Result:** All 4 ASICs hash, shares accepted, **no duplicate shares**, stable mining on Standard Channel! Chip ID distribution is the key to nonce partitioning.

Changes needed:
- `address_interval = 256 / chip_counter` (Bitaxe-style, instead of hardcoded 2 or 4)
- All per-chip `CMD_WRITE_SINGLE` commands use `i * address_interval`
- Nonce-to-ASIC mapping: `((bswap32(nonce) >> 17) & 0xff) / address_interval` (matching Bitaxe)
- `chipIndexFromAddr`: `addr / address_interval` (removed BM1370 override that used `addr >> 2`)
- Register 0x10: HCN value from `setNonceSpace()` instead of VR frequency
- `checkVrFrequencyChanged` disabled (was overwriting HCN on register 0x10)
- HCN recalculated on ASIC frequency change

This also works for multi-chip boards like OCTAXE (8 chips → address_interval=32).

## Open question

The VR frequency UI feature (`checkVrFrequencyChanged`) writes to the same register 0x10 that HCN uses. The Bitaxe doesn't have this feature - it sets register 0x10 once during init via `set_nonce_space()` and never touches it again. Should we remove the VR frequency feature, or is there a way to combine both?

## Known issue

After ~30 minutes, the pool increases difficulty via SetTarget. For Standard Channel, the old bm_job still has the previous pool_diff, causing shares to be submitted that the pool now considers too low. This is an SV2 Standard Channel issue (not nonce-space related) and will be fixed separately.
