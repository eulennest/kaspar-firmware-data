# Kaspar Voice Firmware Data

Binaries, manifests, and board configurations for Kaspar Voice installer.

## Structure

```
./                           ← REPO ROOT (self is the "data" folder!)
├── v1.0.29/                 ← Per-version folder (firmware.bin + manifest.json)
│   ├── firmware.bin         ← NEW: only this is copied per version
│   ├── manifest.json        ← REQUIRED: ESP Web Tools reads this!
│   ├── bootloader.bin       ← SYMLINK: points to ../esp32-s3/bootloader.bin
│   └── partitions.bin       ← SYMLINK: points to ../esp32-s3/partitions.bin
├── v1.0.28/
│   ├── firmware.bin
│   ├── manifest.json
│   ├── bootloader.bin → ../esp32-s3/bootloader.bin
│   └── partitions.bin → ../esp32-s3/partitions.bin
├── esp32-s3/                ← SHARED: bootloader + partitions (static, once!)
│   ├── bootloader.bin
│   └── partitions.bin
└── boards.json              ← Index with all versions
```

## ⚠️ CRITICAL: Common Mistakes (Learn from my failures!)

### ❌ MISTAKE 1: Wrong Directory Structure
```bash
# WRONG:
mkdir -p data/v1.0.29
cp firmware.bin data/v1.0.29/

# RIGHT:
mkdir -p v1.0.29
cp firmware.bin v1.0.29/
```
**Why:** This repo IS the data folder. When webhook deploys to `/opt/install-kaspark/`, it becomes the data folder. No nested `data/` subdirectory!

### ❌ MISTAKE 2: Missing manifest.json
```bash
# WRONG: Only firmware.bin in version folder

# RIGHT: Each v1.0.X/ must have manifest.json!
cat > v1.0.29/manifest.json << 'EOF'
{
  "name": "Kaspar Voice Assistant",
  "version": "1.0.29",
  "home_assistant_domain": "esphome",
  "new_install_skip_erase": false,
  "new_install_prompt_erase": true,
  "builds": [{
    "chipFamily": "ESP32-S3",
    "name": "ESP32-S3 1.28inch BOX",
    "parts": [
      {"path": "bootloader.bin", "offset": 0},
      {"path": "partitions.bin", "offset": 32768},
      {"path": "firmware.bin", "offset": 65536}
    ]
  }]
}
EOF
```
**Why:** ESP Web Tools expects manifest.json at the URL. It reads part offsets and file paths from here, then loads each binary.

### ❌ MISTAKE 3: boards.json URLs point to firmware.bin
```bash
# WRONG:
{
  "url": "data/v1.0.28/firmware.bin"  ← ESP Web Tools can't parse this
}

# RIGHT:
{
  "url": "v1.0.28/manifest.json"      ← ESP Web Tools loads manifest
}
```
**Why:** ESP Web Tools is a manifest-based flasher. It needs the manifest.json to determine chip, offsets, and all binary files.

### ❌ MISTAKE 4: Copying bootloader + partitions per version
```bash
# WRONG (wastes space & risk of inconsistency):
scp bootloader.bin v1.0.28/
scp bootloader.bin v1.0.29/
scp bootloader.bin v1.0.30/

# RIGHT (static, shared):
# Put once in esp32-s3/
scp bootloader.bin esp32-s3/
scp partitions.bin esp32-s3/

# Then SYMLINK in each version
cd v1.0.28 && ln -sf ../esp32-s3/bootloader.bin bootloader.bin
cd v1.0.29 && ln -sf ../esp32-s3/bootloader.bin bootloader.bin
```
**Why:** Bootloader + partitions are static across ALL firmware versions for the same chip. Only firmware.bin changes. Symlinking saves storage and prevents divergence bugs.

### ❌ MISTAKE 5: boards.json references non-existent versions
```bash
# WRONG: boards.json lists v1.0.27, but repo only has v1.0.28
{
  "versions": [
    {"version": "1.0.27", "url": "data/v1.0.27/firmware.bin"},  ← 404!
    {"version": "1.0.26", "url": "data/v1.0.26/firmware.bin"}   ← 404!
  ]
}

# RIGHT: Only list versions that actually exist in repo
{
  "versions": [
    {"version": "1.0.29", "url": "v1.0.29/manifest.json"},
    {"version": "1.0.28", "url": "v1.0.28/manifest.json"}
  ]
}
```
**Why:** Broken links = broken installer. Always sync boards.json with actual repo contents.

## Deployment Workflow

### Adding a new version (v1.0.30)

```bash
# 1. Create version folder
mkdir -p v1.0.30

# 2. Copy ONLY firmware.bin (new per version)
cp ../kaspar-voice-platformio/.pio/build/esp32-s3-xiaozhi/firmware.bin v1.0.30/

# 3. Create manifest.json (update version number!)
cat > v1.0.30/manifest.json << 'EOF'
{
  "name": "Kaspar Voice Assistant",
  "version": "1.0.30",
  "home_assistant_domain": "esphome",
  "new_install_skip_erase": false,
  "new_install_prompt_erase": true,
  "builds": [{
    "chipFamily": "ESP32-S3",
    "name": "ESP32-S3 1.28inch BOX",
    "parts": [
      {"path": "bootloader.bin", "offset": 0},
      {"path": "partitions.bin", "offset": 32768},
      {"path": "firmware.bin", "offset": 65536}
    ]
  }]
}
EOF

# 4. Create symlinks to shared bootloader/partitions
cd v1.0.30
ln -sf ../esp32-s3/bootloader.bin bootloader.bin
ln -sf ../esp32-s3/partitions.bin partitions.bin

# 5. Update boards.json (latest version + add to versions list)
# Edit boards.json:
# - Change "latest.version" to "1.0.30"
# - Change "latest.url" to "v1.0.30/manifest.json"
# - Add new entry to "versions" array

# 6. Commit & push
git add v1.0.30/ boards.json
git commit -m "v1.0.30: [Your changelog here]"
git push origin main

# 7. Webhook auto-deploys to /opt/install-kaspark/
# install.kaspark.de updates automatically!
```

### Verify after deployment

```bash
# On VM, check webhook log
tail -f /var/log/webhooks.log

# Check deployed files exist
ssh vm-22 "ls -la /opt/install-kaspark/v1.0.30/"

# Test installer
curl -s https://install.kaspark.de/v1.0.30/manifest.json | jq
```

## Usage

This repo is pulled by the installer webhook handler to `/opt/install-kaspark/`

On every push to main:
1. Webhook at `https://hooks.kaspark.de/webhook/kaspar-firmware-data` is triggered
2. Repo is cloned/updated to `/opt/install-kaspark/`
3. Files become available at `https://install.kaspark.de/`
