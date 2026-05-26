# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

GitHub Actions CI config for building custom ImmortalWrt (OpenWrt) firmware for SCUT campus network. Does NOT build locally — all compilation happens on GitHub Actions runners. Contains workflow definitions, a feed-prep script, and build config seeds.

## Repo architecture

```
.github/workflows/
  mt7621.yml       # CI for MT7621 (ramips) — immortalwrt/immortalwrt openwrt-21.02
  mt798x.yml       # CI for MT798x (mediatek) — hanwckf/immortalwrt-mt798x openwrt-21.02
01_prepare.sh      # Runs inside cloned openwrt/ — sets up feeds, clones custom packages
02_add_package.sh  # Reads package.conf, appends CONFIG_PACKAGE_*=y to .config
mt7621_config.seed # Pre-made .config for MT7621 builds (devices + core packages)
package.conf       # User's extra package list (consumed by 02_add_package.sh)
```

## Build flow

1. `maximize-build-space` action frees disk on runner
2. Clone `openwrt/` from base repo
3. `01_prepare.sh` — feeds update, clone custom packages, patch configs
4. `cat config-seed > .config`, optionally run `02_add_package.sh`, then `make defconfig`
5. `make download -j8` (with timeout+retry)
6. `make -j$(nproc)` (falls back to serial `-j1 V=s` on failure)
7. Package artifacts into `.7z`, publish as GitHub release

## Custom packages (cloned directly in 01_prepare.sh)

| Package | Source | Purpose |
|---|---|---|
| luci-app-scutclient | hanwckf/luci-app-scutclient | SCUT campus network auth client |
| scut-unicom | wykdg/route_script (wget Makefile) | China Unicom accelerator |
| luci-theme-argon | jerrykuku/luci-theme-argon | UI theme |

## Critical: kenzo/small feeds are DEAD for openwrt-21.02

The community feeds `kenzok8/openwrt-packages` and `kenzok8/small` have dropped openwrt-21.02 support. Their packages now require `firewall4`, `kmod-nft-*`, `rust/host`, `bpf-headers` (all 23.05+ features). Using `./scripts/feeds install -a` with these feeds causes `make defconfig` to **segfault** (SIGSEGV, exit 139).

**Do NOT re-add kenzo/small feeds** unless also upgrading the base repo to openwrt-23.05 or newer.

Packages lost due to feed removal:
- `luci-app-passwall` and sub-components (Xray, Hysteria2, SingBox)
- `luci-app-ssr-plus`
- `luci-app-mosdns`
- Various third-party themes (except argon, which is cloned directly)

Still available from default feeds: `luci-app-openclash`, `luci-app-frpc`, `luci-app-wireguard`, `luci-app-openvpn`, `luci-app-ddns`.

## maximize-build-space constraints

GitHub runner root partition is small. Key parameters that have been tuned:
- `root-reserve-mb: 128` — keep minimal, or apt install runs out of space
- `temp-reserve-mb: 1` — must be 1 (not higher), else `fallocate` fails on temp PV creation
- Always do `apt clean && rm -rf /var/lib/apt/lists/*` BEFORE `apt update`

## MT798x notes

- Uses defconfig files from the cloned `hanwckf/immortalwrt-mt798x` repo (`defconfig/${build_variant}.config`)
- That repo only has `openwrt-21.02` branch — no 23.05 path exists
- Shares `01_prepare.sh` with MT7621 builds
- Already has `fail-fast: false`, timeout guards, download retry (was always more robust than mt7621)

## sed patches in 01_prepare.sh

All sed commands must use pattern-based matching, NOT hardcoded line numbers. The upstream base-files/ttyd/emortal packages may change. Current patches:
- rc.local: insert Unicom accelerator hints before `exit 0`
- ttyd.config: add `-f root` to default login command
- 99-default-settings: add trojan-go → trojan symlink fallback before `exit 0`

## Upgrading to openwrt-23.05

Viable path to recover passwall/mosdns/etc. Would require:
- MT7621: change `REPO_BRANCH` to `openwrt-23.05` (immortalwrt has this)
- MT798x: find a 23.05-compatible mt798x repo (hanwckf's is 21.02 only), or use immortalwrt's built-in mediatek support
- Re-add kenzo/small feeds after upgrade
- Config seed should migrate automatically via `make defconfig`
