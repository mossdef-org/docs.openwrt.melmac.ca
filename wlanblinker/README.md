<!-- markdownlint-disable MD013 -->
<!-- markdownlint-disable MD030 -->

# WLAN Blinker

## Description

This service can be used to indicate WLAN status by blinking the unused LED.

## Features

-   Supports blinking to indicate the current AP's channel
-   Supports blinking to indicate current STA's link quality (each blink is ~10% of link quality)

## How to install

Please make sure that the requirements are satisfied and install `wlanblinker` and `luci-app-wlanblinker` from Web UI or connect to your router via ssh and run the following commands:

```sh
opkg update
opkg install wlanblinker luci-app-wlanblinker
```

If these packages are not found in the official feed/repo for your version of OpenWrt, you will need to add a custom repo to your router following instructions on [GitHub](https:/docs.openwrt.melmac.net/#on-your-router)/[jsDelivr](https://cdn.jsdelivr.net/gh/stangri/docs.openwrt.melmac.net/README.md#on-your-router) first.

These packages have been designed to be backwards compatible with OpenWrt 19.07, OpenWrt 18.06, OpenWrt Project 17.01 and OpenWrt 15.05. However, on systems older than OpenWrt 18.06.6 and/or a system which has deviated too far (or haven't been updated to keep in-sync) with official OpenWrt release you may get a message about missing `luci-compat` dependency, which (and only which) you can safely ignore and force-install the luci app using `opkg install --force-depends` command instead of `opkg install`.

## Default Settings

Default configuration has service disabled (use Web UI to enable/start service or run `uci set wlanblinker.config.enabled=1; uci commit wlanblinker;`).

<!-- markdownlint-disable MD033 -->
<script defer src='https://static.cloudflareinsights.com/beacon.min.js' data-cf-beacon='{"token": "911798f2c34b45338f8f8182830a3eb6"}'></script>
