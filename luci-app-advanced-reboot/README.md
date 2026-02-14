<!-- markdownlint-disable MD030 -->

# Advanced Reboot Web UI (luci-app-advanced-reboot)

## Description

This package allows you to reboot to an alternative partition on the supported (dual-firmware) routers and to power off (power down) your OpenWrt device.

## Supported Devices

Currently supported dual-firmware devices include:

- Linksys E4200v2
- Linksys E7350
- Linksys EA3500
- Linksys EA4500
- Linksys EA6350v3
- Linksys EA6350v4
- Linksys EA7300v1
- Linksys EA7300v2
- Linksys EA7500v1
- Linksys EA7500v2
- Linksys EA7500v3
- Linksys EA8100v1
- Linksys EA8100v2
- Linksys EA8300
- Linksys EA8500
- Linksys MR5500
- Linksys MR8300
- Linksys MR9000
- Linksys MX2000
- Linksys MX4200v1
- Linksys MX4200v2
- Linksys MX4300
- Linksys MX5300
- Linksys MX5500
- Linksys MX6200
- Linksys SPNMX56
- Linksys WHW01v1
- Linksys WHW03v2
- Linksys WRT32X
- Linksys WRT1200AC
- Linksys WRT1900AC
- Linksys WRT1900ACS
- Linksys WRT1900ACv2
- Linksys WRT3200ACM
- Xiaomi AX3600
- Xiaomi AX9000
- ZyXEL NBG6817

If your device is not in the list above, however it is a [dual-firmware device](https://openwrt.org/tag/dual_firmware?do=showtag&tag=dual_firmware) and you're interested in having your device supported, please refer to [How to add a new device](#how-to-add-a-new-device).

The package in the official OpenWrt repo only supports the routers supported in the official snapshots or release builds. In rare cases, the package in my own private repo may support routers not yet supported in the package from the official OpenWrt repo.

## Screenshot (luci-app-advanced-reboot)

![screenshot](https://docs.openwrt.melmac.net/luci-app-advanced-reboot/screenshots/screenshot02.png "screenshot")

## How to install

Install `luci-app-advanced-reboot` from Web UI or connect to your router via ssh and run the following commands:

```sh
opkg update
opkg install luci-app-advanced-reboot
```

If the `luci-app-advanced-reboot` package with support of your device is not found in the official feed/repo for your version of OpenWrt, you will need to add a custom repo to your router following instructions on [GitHub](https://docs.openwrt.melmac.net/#on-your-router)/[jsDelivr](https://cdn.jsdelivr.net/gh/stangri/docs.openwrt.melmac.net/README.md#on-your-router) first.

## How to add a new device

This package does not implement dual-firmware support to the OpenWrt device, rather it uses built-in OpenWrt tools to browse/switch partitions on dual-firmware devices supported by OpenWrt (usually by the device maintainer/committer into the OpenWrt tree).

The dual-firmware devices need to be explicitly supported by this package (there's no auto-discovery of supported models). If you are interested in having a device supported by this package, you can either create the support file yourself or request it.

### Requesting Support

Please post in the [OpenWrt Forum Support Thread](https://forum.openwrt.org/t/3423) the following information:

- Link to the device OpenWrt wiki Table of Hardware page and/or link to the git commit to OpenWrt tree with support of the device being added.
- The output of the following commands from the console:

  ```sh
  ubus call system board
  cat /tmp/sysinfo/board_name
  cat /proc/mtd
  fw_printenv
  ```

### Adding Device Support Yourself

To support a new device, you need to create a `.json` file that tells the Advanced Reboot app how your router switches partitions.

#### Step 1: Get the Board Name

Log into your router via SSH and run:

```bash
cat /tmp/sysinfo/board_name
```

This string (e.g., `linksys,mx4200v1`) goes into the `board` list in your JSON file.

#### Step 2: Identify Boot Variables

Find out which environment variables change when you switch partitions.

1. Run `fw_printenv` to see current settings.
2. Look for variables like `boot_part`, `bootcmd`, or `boot_part_ready`.
3. Note their values for Partition 1 vs. Partition 2.

_If your device doesn't use environment variables (like some ZyXEL models), it might use a "dual-boot flag" bytes on a specific partition._

#### Step 3: Identify Partitions

Run `cat /proc/mtd` to see your partitions. You need to find:

- The MTD device for Partition 1 (e.g., `mtd21`, `firmware`).
- The MTD device for Partition 2 (e.g., `mtd23`, `alt_firmware`).
- (Optional) An offset to find the "OpenWrt" label.

#### Step 4: Write the JSON File

Create a file named `your-device-model.json`. Use the template below:

**Example (Environment Variable Method):**

```json
{
  "device": {
    "vendor": "Linksys",
    "model": "MX4200v1",
    "board": ["linksys,mx4200v1"]
  },
  "commands": {
    "params": ["boot_part"],
    "get": "fw_printenv",
    "set": "fw_setenv",
    "save": null
  },
  "partitions": [
    {
      "number": 1,
      "param_values": ["1"],
      "mtd": "mtd21",
      "labelOffsetBytes": 192
    },
    {
      "number": 2,
      "param_values": ["2"],
      "mtd": "mtd23",
      "labelOffsetBytes": 192
    }
  ]
}
```

- **device**: Vendor, Model, and the board name from Step 1.
- **commands**:
  - `params`: The variable names from Step 2.
  - `get`/`set`: Usually `fw_printenv` and `fw_setenv`.
- **partitions**:
  - `number`: 1 or 2.
  - `param_values`: The values for the variables in `params` (must be in the same order).
  - `mtd`: The MTD identifier (e.g., `mtd21`).
  - `labelOffsetBytes`: Where to look for the OS label (try 32, 64, or 192).

#### Push the JSON to the Router

Use `scp` (Secure Copy) to transfer your file to the router.

**Command:**

```bash
scp your-device-model.json root@192.168.1.1:/usr/share/advanced-reboot/devices/
```

_(Replace `192.168.1.1` with your router's IP address)_

If that directory doesn't exist yet, create it on the router first:

```bash
mkdir -p /usr/share/advanced-reboot/devices/
```

#### Test with CLI Commands

After copying the file, log in to the router and test it using `ubus` commands.

1.  **Check if the device is recognized**:

    ```bash
    ubus -v call luci.advanced-reboot obtain_device_info
    ```

    You should see your device info and partition status.

2.  **Test Partition Switching** (Warning: effectively reboots/changes boot env!):

    ```bash
    ubus -S call luci.advanced-reboot boot_partition '{ "number": "1" }'
    ubus -S call luci.advanced-reboot boot_partition '{ "number": "2" }'
    ```

## Notes/Known Issues

- The package in the official OpenWrt repo only supports the routers supported in the official snapshots or release builds. In rare cases, the package in my own private repo may support routers not yet supported in the package from the official OpenWrt repo.
- When you reboot to a different partition, your current settings (WiFi SSID/password, etc.) will not apply to a different partition. Different partitions might have completely different settings and even firmware.
- If you reboot to a partition which doesn't allow you to switch boot partitions (like stock vendor firmware), you might not be able to boot back to OpenWrt unless you reflash it, losing all the settings.
- Some devices allow you to trigger reboot to an alternative partition by interrupting boot 3 times in a row (by resetting/switching off the device or pulling power). As these methods might be different for different devices, do your own homework.
- Newer versions of this package try to mount alternative partition on compatible NAND routers in order to retrieve detailed firmware information. When that happens, it is normal to have messages similar to the below in the system log:

  ```sh
  Tue Nov 19 15:45:03 2019 user.notice luci-app-advanced-reboot: attempting to mount alternative   partition (mtd6)
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.673826] ubi2: attaching mtd6
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.876698] ubi2: scanning is finished
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.885267] ubi2: attached mtd6 (name "rootfs1", size   74 MiB)
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.891063] ubi2: PEB size: 131072 bytes (128 KiB),   LEB size: 126976 bytes
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.898011] ubi2: min./max. I/O unit sizes: 2048/2048,   sub-page size 2048
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.904878] ubi2: VID header offset: 2048 (aligned   2048), data offset: 4096
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.911928] ubi2: good PEBs: 592, bad PEBs: 0,   corrupted PEBs: 0
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.917962] ubi2: user volume: 2, internal volumes: 1,   max. volumes count: 128
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.925252] ubi2: max/mean erase counter: 48/32, WL   threshold: 4096, image sequence number: 1659081076
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.934623] ubi2: available PEBs: 0, total reserved   PEBs: 592, PEBs reserved for bad PEB handling: 40
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.944346] ubi2: background thread "ubi_bgt2d"   started, PID 26780
  Tue Nov 19 15:45:03 2019 kern.info kernel: [30392.952596] block ubiblock2_0: created from ubi2:0  (rootfs)
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30392.964083] UBIFS (ubi2:1): background thread   "ubifs_bgt2_1" started, PID 26787
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30393.009298] UBIFS (ubi2:1): UBIFS: mounted UBI device   2, volume 1, name "rootfs_data"
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30393.017185] UBIFS (ubi2:1): LEB size: 126976 bytes   (124 KiB), min./max. I/O unit sizes: 2048 bytes/2048 bytes
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30393.027213] UBIFS (ubi2:1): FS size: 61075456 bytes   (58 MiB, 481 LEBs), journal size 3047424 bytes (2 MiB, 24 LEBs)
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30393.037733] UBIFS (ubi2:1): reserved for root: 2884744   bytes (2817 KiB)
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30393.044389] UBIFS (ubi2:1): media format: w4/r0   (latest is w5/r0), UUID 76F0C52C-6197-4E00-B306-747262B06545, small LPT model
  Tue Nov 19 15:45:03 2019 user.notice luci-app-advanced-reboot: attempting to unmount alternative   partition (mtd6)
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30393.132743] UBIFS (ubi2:1): un-mount UBI device 2
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30393.137481] UBIFS (ubi2:1): background thread   "ubifs_bgt2_1" stops
  Tue Nov 19 15:45:03 2019 kern.info kernel: [30393.390961] block ubiblock2_0: released
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30393.396576] ubi2: detaching mtd6
  Tue Nov 19 15:45:03 2019 kern.notice kernel: [30393.400117] ubi2: mtd6 is detached
  ```

## Thanks

I'd like to thank everyone who helped create, test and troubleshoot this package. Without help from [@hnyman](https://github.com/hnyman), [@jpstyves](https://github.com/jpstyves), [@imi2003](https://github.com/imi2003), [@jeffsf](https://github.com/jeffsf), [@Ansuel](https://github.com/Ansuel), [@djadair](https://github.com/djadair), [@tiagofreire-pt](https://github.com/tiagofreire-pt), and many contributions from [@slh](https://github.com/pkgadd) it wouldn't have been possible.

<!-- markdownlint-disable MD033 -->

<script defer src='https://static.cloudflareinsights.com/beacon.min.js' data-cf-beacon='{"token": "911798f2c34b45338f8f8182830a3eb6"}'></script>
