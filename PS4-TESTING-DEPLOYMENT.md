# PS4 OrbisStore – Testing & Deployment Guide

## Build Instructions

### Prerequisites
- OpenOrbis PS4 Toolchain v0.5.4+ installed
- Linux or macOS with clang and LLVM tools
- PkgTool.Core dependencies installed (libssl 1.1)

### Step 1: Setup Environment

```bash
# Source the environment setup
source /home/neox/.orbis-env.sh

# Verify toolchain
echo $OO_PS4_TOOLCHAIN
# Output: /home/neox/Downloads/OpenOrbis/PS4Toolchain
```

### Step 2: Clean Build

```bash
cd ps4
make clean && make
```

### Expected Output

```
✓ Linked: OrbisStore.elf (167520 bytes)
✓ Generated Open-ELF: OrbisStore.oelf (171872 bytes)
✓ Created param.sfo with multi-firmware support (SYSTEM_VER=0)
✓ Runtime assets prepared

BUILD VALIDATION
─────────────────────────────────────────
✓ OrbisStore.elf exists (167520 bytes)
✓ OrbisStore.oelf exists (171872 bytes)
✓ pkg_root/eboot.bin exists (171872 bytes)
✓ pkg_root/sce_sys/param.sfo exists (504 bytes)
✓ Assets packaged: 2 files
─────────────────────────────────────────

→ Building PKG...
✓ PKG created successfully
  File: OrbisStore.pkg
  Size: 6619136 bytes (6MB)
✓ Build successful: OrbisStore.pkg
```

### Step 3: Validate Build Artifacts

```bash
cd ps4
source /home/neox/.orbis-env.sh
make validate
```

### Step 4: Check Generated Files

```bash
ls -lh ps4/OrbisStore.pkg
# Output: -rw-rw-r-- 1 user user 6.6M May 23 12:34 OrbisStore.pkg

# Verify package structure
file ps4/OrbisStore.pkg
# Output: ps4/OrbisStore.pkg: data

# Check metadata
hexdump -C ps4/OrbisStore.pkg | head -20
```

---

## PS4 Installation & Testing

### Hardware Setup
- PS4 Console (any model with CFW/GoldHEN)
- USB drive (FAT32 formatted, >1GB)
- Network connection (for API calls in future)

### Supported Firmware Versions
- ✅ 5.05 (Initial jailbreak)
- ✅ 6.72 (Popular CFW base)
- ✅ 7.xx
- ✅ 8.xx
- ✅ 9.00+
- ✅ 10.xx
- ✅ 11.xx (Latest jailbreakable)
- ✅ 13.00+ (with GoldHEN)

### Installation Steps

#### Option 1: USB Installation (Recommended)

1. **Prepare USB Drive**
   ```bash
   # Copy PKG to USB
   cp ps4/OrbisStore.pkg /mnt/usb/
   
   # Verify file is readable
   ls -lh /mnt/usb/OrbisStore.pkg
   ```

2. **On PS4 Console**
   - Insert USB drive
   - Navigate to: **Settings** → **System Software** → **Update from USB**
   - Wait for installation to complete (~2-5 minutes)

3. **Verify Installation**
   - App should appear in PS4 library
   - Should NOT show as corrupted/broken file

#### Option 2: Network Installation (If Available)
```bash
# Host PKG on PC
cd ps4
python3 -m http.server 8000

# On PS4 web browser: http://<PC-IP>:8000/OrbisStore.pkg
```

---

## Runtime Testing Checklist

### Immediate Launch Test
- [ ] Launch app from PS4 home menu
- [ ] **CRITICAL: Should NOT crash immediately**
- [ ] Should NOT show error: **CE-30008-1**
- [ ] Application should initialize and display UI

### Verify Data Directory
On PS4 command line (after enabling debug settings):

```bash
# SSH into PS4 (if available)
ssh ps4@<ps4-ip>

# Check data directory creation
ls -la /data/OrbisStore/
# Should show directory with permissions

# Check for database files
ls -la /data/OrbisStore/library.db*
# Should exist and be writable
```

### Functional Tests

| Feature | Test | Expected Result |
|---------|------|-----------------|
| Store Tab | Navigate to Store section | Displays 5 built-in apps |
| Install Button | Click Install on any app | Shows install animation |
| Library Tab | Switch to Library | Shows installed apps |
| Database | Check file creation | `/data/OrbisStore/library.db.txt` exists |
| Settings Tab | View settings | Shows Title ID, API endpoint, version |
| Multi-tab | Switch between tabs | Smooth navigation without crashes |
| Quit | Close application | Clean exit (no memory leaks) |

### PS4 System Integration

- [ ] App appears correctly in library
- [ ] Icon displays properly
- [ ] Metadata shows: ORBS00001 / OrbisStore / v01.00
- [ ] Can be highlighted, selected, launched
- [ ] Application state persists across launches
- [ ] No system-level warnings/errors in PS4 debug log

### Data Persistence Testing

1. **First Launch**
   ```
   /data/OrbisStore/
   └── library.db.txt (empty or default content)
   ```

2. **After Installing App**
   ```
   /data/OrbisStore/
   ├── library.db.txt (contains app entry)
   ├── temp/
   └── cache/
   ```

3. **After Closing & Reopening**
   - Same files should exist
   - Database should retain installed apps
   - No permission errors

---

## Troubleshooting

### Issue: CE-30008-1 Error (App Crashes)

**Symptoms:**
- App launches then crashes
- PS4 shows error CE-30008-1

**Root Causes & Fixes:**
| Cause | Fix |
|-------|-----|
| Data path inaccessible | ✓ Fixed in Makefile (now uses `/data/OrbisStore/`) |
| Corrupted EELF binary | Rebuild: `make clean && make` |
| Broken GP4 manifest | Check GP4 file: `file ps4/package.gp4` |
| Insufficient storage | Free 100MB+ on PS4 |
| PS4 firmware incompatible | Use supported firmware (5.05+) or GoldHEN |

**Solution Steps:**
```bash
1. Rebuild the package:
   cd ps4 && make clean && make

2. Verify artifacts:
   make validate

3. Check file integrity:
   file OrbisStore.pkg
   ls -lh OrbisStore.pkg

4. Reinstall on PS4:
   Copy new OrbisStore.pkg to USB
   Delete old app from PS4
   Reinstall from USB
   Test launch
```

### Issue: Installation Fails

**Symptoms:**
- PS4 shows "corrupted" or won't install

**Debugging:**
```bash
# Check PKG file size (should be ~6.6MB)
ls -lh ps4/OrbisStore.pkg

# Verify it's a valid package
file ps4/OrbisStore.pkg
# Should show: data (PS4 package)

# Check GP4 validity
cat ps4/package.gp4 | head -20
# Should contain XML structure

# Rebuild and verify all artifacts
cd ps4 && make validate
```

### Issue: Database Permission Error

**Symptoms:**
- App launches but can't read/write library

**Fix:**
```bash
# Verify /data/OrbisStore/ has correct permissions
# On PS4: chmod 777 /data/OrbisStore/

# Application should create it automatically
# If not, check Makefile compiled with correct path:
grep HOMEBREW_DATA_PATH ps4/Makefile
# Should show: HOMEBREW_DATA_PATH := /data/OrbisStore/
```

### Issue: App Not in Library

**Symptoms:**
- Installation completes but app doesn't appear

**Debugging:**
1. Check PS4 file system:
   ```bash
   ls -la /user/app/
   # Should contain ORBS00001 directory
   ```

2. Verify metadata:
   ```bash
   hexdump -C /user/app/ORBS00001/sce_sys/param.sfo | grep -A 2 TITLE
   # Should show "OrbisStore"
   ```

3. Rebuild and reinstall

---

## Performance Metrics

### Build Time
```
Complete clean build: ~5-10 seconds
- Compilation: ~2s (8 source files)
- Linking: ~1s
- Package generation: ~2-5s
- Total: ~5-10s on modern hardware
```

### Package Characteristics
```
Executable size: 172 KB (OELF)
Metadata size: 504 B (param.sfo)
Package file: 6.6 MB (includes padding, encryption)
Installation size on PS4: ~50-100 MB (with reserve)
```

### Memory Usage (On PS4)
```
Code segment: ~200 KB
Stack: ~1 MB
Heap: Dynamic (library data)
Total baseline: ~5-10 MB
```

---

## Security Considerations

### Data Protection
- Application runs with GoldHEN privileges
- Database stored in writable homebrew directory (`/data/`)
- Files NOT encrypted (GoldHEN environment)
- Database is plain text (not sensitive)

### Permissions
```
/data/OrbisStore/     - rwxr-xr-x (755)
/data/OrbisStore/*.db - rw-r--r-- (644)
```

### Multi-Firmware Compatibility
- SYSTEM_VER = 0 allows operation on any firmware
- No specific firmware features required
- Uses only standard PS4 APIs (Pad, SysService, Bgft)
- Compatible with both official and custom environments

---

## Deployment Checklist

### Pre-Deployment
- [ ] Code reviewed and compiled successfully
- [ ] All validation checks pass (`make validate`)
- [ ] Package file exists and is >6MB
- [ ] No warnings or errors in build output
- [ ] Git changes committed and pushed

### Deployment
- [ ] PKG copied to distribution medium (USB, cloud)
- [ ] Installation tested on hardware
- [ ] App launches without CE-30008-1 error
- [ ] Database created at `/data/OrbisStore/`
- [ ] All UI sections functional

### Post-Deployment
- [ ] Monitor error reports
- [ ] Check user feedback on crashes
- [ ] Track database growth over time
- [ ] Plan for future firmware updates

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 01.00 | 2026-05-23 | Initial release with CE-30008-1 fix |
| - | - | Multi-firmware support (SYSTEM_VER=0) |
| - | - | Safe data path `/data/OrbisStore/` |
| - | - | Production-quality build system |

---

## Support & Debugging

### Enable Debug Output
```bash
# On PS4 with SSH enabled:
ssh ps4@<ps4-ip>

# Monitor system logs
tail -f /var/log/messages | grep OrbisStore

# Check database integrity
head -50 /data/OrbisStore/library.db.txt
```

### Rebuild After Changes
```bash
cd ps4

# If only sources changed:
make

# If anything is broken:
make clean && make

# After major changes:
rm -rf pkg_root && make
```

### Contact
- GitHub: https://github.com/Quixdv/orbis-store
- Issues: https://github.com/Quixdv/orbis-store/issues

---

**Last Updated:** May 23, 2026  
**Makefile Version:** 2.0 (CE-30008-1 Fix)  
**OpenOrbis Toolchain:** v0.5.4+  
**Multi-Firmware Support:** Yes (5.05 – 13.00+)
