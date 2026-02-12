# Kaspar Voice Firmware Data

Binaries, manifests, and board configurations for Kaspar Voice installer.

## Structure

- `boards.json` - Board definitions and available versions
- `v1.0.X/` - Firmware versions with manifest.json
- `esp32-s3/` - Shared bootloader + partitions for ESP32-S3

## Usage

This repo is pulled by the installer webhook handler to `/opt/install-kaspark/data/`
