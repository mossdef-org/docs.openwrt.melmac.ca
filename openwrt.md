<!-- markdownlint-disable MD033 -->

# There's OpenWrt and then there's "OpenWrt"


- [What is OpenWrt?](#WhatisOpenWrt)
- [How is OpenWrt different from "OpenWrt"?](#HowisOpenWrtdifferentfromOpenWrt)
- [How can I tell if I have an official or a third-party OpenWrt installed on my device?](#HowcanItellifIhaveanofficialorathird-partyOpenWrtinstalledonmydevice)
- [But the vendor of my device claims it runs OpenWrt](#ButthevendorofmydeviceclaimsitrunsOpenWrt)
- [What do I gain by installing official OpenWrt on my device?](#WhatdoIgainbyinstallingofficialOpenWrtonmydevice)
- [What do you mean by support?](#Whatdoyoumeanbysupport)


## <a name='WhatisOpenWrt'></a>What is OpenWrt?

OpenWrt is a Linux-based open source (as in the source code is availble for anyone to view, copy and create their own firmware) operating system designed to work on a variety of "embedded systems", mostly networking devices with limited hardware/RAM/storage. Currently OpenWrt supports dozens architectures and hundreds of devices. There are usually major releases each year with some devices added and some devices (with limited RAM/storage no longer supoportable mostly due to changes in Linux kernel or overall image size) dropped.

The OpenWrt core team maintains the following resources:

- Source code for the project, from which anyone can build their own version of OpenWrt.
- A command-line Image Builder for different achitectures, allowing anyone to quickly create their own version of official OpenWrt build with custom settings/packages.
- An online [Firmware Selector](https://firmware-selector.openwrt.org/) allowing most of the functionality as an item above.
- The pre-made firmware [image downloads](https://downloads.openwrt.org/releases/24.10.2/targets/) for all supported devices. The OpenWrt version 24.10.2 images are linked in this section and usually the last release or two are what is being updated by OpenWrt core team and what most people have running on their devices.

The images availble from either [OpenWrt Downloads](https://downloads.openwrt.org/) or images created with an Image Builder/Firmware Selector are what is considered an official OpenWrt.

## <a name='HowisOpenWrtdifferentfromOpenWrt'></a>How is OpenWrt different from "OpenWrt"?

Given the open source nature of OpenWrt, sometimes device vendors copy a version of OpenWrt source code and create their own customized firmware based on OpenWrt. Most often:

- The vendor then does not release any updates for neither the included packages/components nor the OpenWrt itself, so it's stuck forever on the version originally copied/cloned from OpenWrt. As such, some parts of those customized OpenWrt-based firmwares might be vulnerable to attacks.
- It contains a non-open-source/proprietary software or binary blobs, especially for wireless radios or remote control.
- It includes some other proprietary customizations to any/all components of the device, like networking, DNS services, etc.
- You can't install or properly use any/many of the OpenWrt available packages or WebUI applications on such third-party made firmwares.
- There are both known and unknown backdoors into firmware originally created to allow vendor or your ISP access to your device. If backdoor is known, an attacker online may gain access to your device.
- Nobody but the vendor knows what exactly has been changed from the official OpenWrt setup/configuration.
- It is possible and advisable to install the official OpenWrt firmware on such devices.

## <a name='HowcanItellifIhaveanofficialorathird-partyOpenWrtinstalledonmydevice'></a>How can I tell if I have an official or a third-party OpenWrt installed on my device?

Create a new thread at [OpenWrt Forum](https://forum.openwrt.org/) and post the output of the following commands ran on your device:

```sh
ubus call system board
cat /etc/os-release
```

One of the OpenWrt forum gurus should be able to confim if it's an official OpenWrt.

## <a name='ButthevendorofmydeviceclaimsitrunsOpenWrt'></a>But the vendor of my device claims it runs OpenWrt

Usually, even the most OpenWrt-friendly (like [GL-iNet](https://www.gl-inet.com/)) device vendors ship their devices with a proprietary OpenWrt-based firmware. What vendors claiming OpenWrt being installed/supported usually mean that the included firmware is [OpenWrt-based](#HowisOpenWrtdifferentfromOpenWrt). For a variety of reasons (starting from vendor not submitting their patches to the official OpenWrt code to some hardware components of the device not having open source software supporting it, hence not acceptable to OpenWrt), it is not a guarantee tho, that you can install an [official OpenWrt image](#WhatisOpenWrt) on the device made by vendor.

The only notable exception is the [OpenWrt One device](https://openwrt.org/toh/openwrt/one) which comes with the genuine, proper OpenWrt version out of the box.

## <a name='WhatdoIgainbyinstallingofficialOpenWrtonmydevice'></a>What do I gain by installing official OpenWrt on my device?

There are many reasons to install an official OpenWrt on your device instead of the "OpenWrt" created by a vendor, like:

- Documentation: The OpenWrt has a great documentation site describing on how to install/customize/use OpenWrt and most prominent of its features and additional packages, like ad-blocking, secure DNS resolution, VPN tunnels and split tunneling, etc. You can visit [OpenWrt Documentation](https://openwrt.org/docs/start) for more details.
- Access to many packages and their WebUI applications: The OpenWrt is like a smartphone OS, where you can install additional packages and WebUI applications to enable additional features like the ones mentioned above.
- Full access/customization of your device: OpenWrt allows the SSH/terminal access and offers WebUI allowing you to fully customize your device.
- Updates: Unless your device has crippled/limited hardware, you should be able to upgrade it to newer OpenWrt images for years to come.
- Security: The OpenWrt core team publishes security updates for a few most recent releases to any of the packages/components with discovered vulnerabilities.
- Support at [OpenWrt Forum](https://forum.openwrt.org/) should you need it.

## <a name='Whatdoyoumeanbysupport'></a>What do you mean by support?

While there're no OpenWrt "employees" providing any support for the official OpenWrt firmwares, there are many very knowledgable volunteers who frequent [OpenWrt Forum](https://forum.openwrt.org/) and can provide information, troubleshooting and guidance on setting up and configuring official OpenWrt. Some OpenWrt core developers, as well as package developers/maintainers also frequent forums and can provide more guidance on a specific package/issue.
