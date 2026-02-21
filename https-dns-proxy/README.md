<!-- markdownlint-disable MD013 -->
<!-- markdownlint-disable MD033 -->

# DNS Over HTTPS Proxy (https-dns-proxy)

![OpenWrt](https://img.shields.io/badge/OpenWrt-Compatible-blueviolet)
![Web UI](https://img.shields.io/badge/Web_UI-Available-blue)
![Minimal Footprint](https://img.shields.io/badge/Size-~40KB-green)
![Resolvers](https://img.shields.io/badge/Resolvers-40%2B%20Built--in-brightgreen)
![License](https://img.shields.io/badge/License-MIT-lightgrey)


- [DNS Over HTTPS Proxy (https-dns-proxy)](#dns-over-https-proxy-https-dns-proxy)
  - [Description](#description)
  - [Features](#features)
  - [Screenshots (luci-app-https-dns-proxy)](#screenshots-luci-app-https-dns-proxy)
  - [Requirements](#requirements)
    - [HTTP/2 Support](#http2-support)
    - [HTTP/3 (QUIC) Support](#http3-quic-support)
  - [Unmet Dependencies](#unmet-dependencies)
  - [How To Install](#how-to-install)
  - [Pre-Release/Testing Versions](#pre-releasetesting-versions)
  - [Default Settings](#default-settings)
  - [Configuration Settings](#configuration-settings)
    - [General Settings](#general-settings)
      - [canary\_domains\_icloud](#canary_domains_icloud)
      - [canary\_domains\_mozilla](#canary_domains_mozilla)
      - [dnsmasq\_config\_update](#dnsmasq_config_update)
      - [force\_dns](#force_dns)
      - [force\_dns\_port](#force_dns_port)
      - [force\_dns\_src\_interface](#force_dns_src_interface)
      - [procd\_trigger\_wan6](#procd_trigger_wan6)
      - [heartbeat\_domain](#heartbeat_domain)
      - [heartbeat\_sleep\_timeout](#heartbeat_sleep_timeout)
      - [heartbeat\_wait\_timeout](#heartbeat_wait_timeout)
    - [Global Settings for Instances](#global-settings-for-instances)
    - [Instance Settings](#instance-settings)
  - [Using https-dns-proxy for ad-blocking](#using-https-dns-proxy-for-ad-blocking)
  - [Donate](#donate)
  - [Getting Help](#getting-help)
  - [Thanks](#thanks)


## Description

A lightweight, RFC8484-compatible DNS-over-HTTPS (DoH) proxy service for OpenWrt with minimal footprint and seamless integration. It does not support outdated JSON API (by design), keeping the package lean and fast. The optional Web UI (`luci-app-https-dns-proxy`) provides access to an extensive built-in list of [more than 40 public DoH resolvers](https://github.com/stangri/source.openwrt.melmac.net/tree/master/luci-app-https-dns-proxy/root/usr/share/https-dns-proxy/providers) for easy configuration. Based on [@aarond10](https://github.com/aarond10)'s [https-dns-proxy](https://github.com/aarond10/https_dns_proxy).

Key advantages:

- Extremely small footprint (~40KB installed)
- Optional Web UI with built-in support for over 40 public DoH resolvers
- Seamless integration with dnsmasq, including automatic fallback and recovery
- Supports canary domain handling for Mozilla and iCloud Private Relay
- HTTP/2 ready; HTTP/3 status detection supported in UI

## Features

- [RFC8484](https://tools.ietf.org/html/rfc8484)-compatible DoH Proxy.
- Compact size (about 40Kb installed).
- (By default) automatically updates `dnsmasq` settings to use DoH proxy when it's started and reverts to old `dnsmasq` resolvers when DoH proxy is stopped.
- (By default) automatically adds records for canary domains[<sup>1</sup>](https://support.mozilla.org/en-US/kb/canary-domain-use-application-dnsnet)<sup>,</sup>[<sup>2</sup>](https://developer.apple.com/support/prepare-your-network-for-icloud-private-relay) upon start and removes them upon service stop.
- Web UI (`luci-app-https-dns-proxy`) available. [More than 40 public resolvers](https://github.com/stangri/source.openwrt.melmac.net/tree/master/luci-app-https-dns-proxy/root/usr/share/https-dns-proxy/providers) are supported within the WebUI for easy configuration.

## Screenshots (luci-app-https-dns-proxy)

Service Status

![screenshot](https://docs.openwrt.melmac.net/https-dns-proxy/screenshots/screenshot02-status.png "Service Status")

Service Configuration

![screenshot](https://docs.openwrt.melmac.net/https-dns-proxy/screenshots/screenshot02-config.png "Service Configuration")

Service Instances

![screenshot](https://docs.openwrt.melmac.net/https-dns-proxy/screenshots/screenshot02-instances.png "Service Instances")

## Requirements

This proxy requires the following packages to be installed on your router: `libc`, `libcares`, `libcurl`, `libev`, `ca-bundle`. They will be automatically installed when you're installing `https-dns-proxy`.

### HTTP/2 Support

Some resolvers may require `HTTP/2`. By default, `HTTP/2` is supported by `curl` in OpenWrt 22.03 and later, if you run an older version of OpenWrt I'd recommend you upgrade to a most recent released version and make sure the following packages are installed: `curl`, `libcurl4`, `libnghttp2`.

### HTTP/3 (QUIC) Support

As of OpenWrt version 24.10, the OpenWrt installation/repositories do not contain packages required for HTTP/3 support. However the `luci-app-https-dns-proxy` has facilities for marking some pre-configured providers as `HTTP/3` only and can detect the `HTTP/3` support on the OpenWrt device. You can compile `curl` with `HTTP/3` support using the out-of-tree fork of `OpenSSL`, additional information is available in [curl README](https://docs.openwrt.melmac.net/curl/).

## Unmet Dependencies

If you are running a development (trunk/snapshot) build of OpenWrt on your router and your build is outdated (meaning that packages of the same revision/commit hash are no longer available and when you try to satisfy the [requirements](#requirements) you get errors), please flash either current OpenWrt release image or current development/snapshot image.

## How To Install

Install `https-dns-proxy` and `luci-app-https-dns-proxy` packages from Web UI or run the following in the command line:

```sh
opkg update; opkg install https-dns-proxy luci-app-https-dns-proxy;
```

## Pre-Release/Testing Versions

The binaries newer than those in OpenWrt repositories for both most recent OpenWrt release and OpenWrt snapshots can usually be found at the [GitHub Releases](https://github.com/stangri/https-dns-proxy/releases) page.

## Default Settings

Default configuration has service enabled and starts the service with Google and Cloudflare DoH servers. In most configurations, you will keep the default `dnsmasq` service installed to handle requests from devices in your local network and point `dnsmasq` to use `https-dns-proxy` for name resolution.

By default, the service will intelligently override existing `dnsmasq` servers settings on start to use the DoH servers and restores original `dnsmasq` servers on stop. See the [Configuration Settings](#configuration-settings) section below for more information and how to disable this behavior.

## Configuration Settings

Configuration contains the general (named) "main" config section where you can configure which `dnsmasq` settings the service will automatically affect and the typed (unnamed) https-dns-proxy instance settings. The original config file is included below:

```text
config main 'config'
  option canary_domains_icloud '1'
  option canary_domains_mozilla '1'
  option dnsmasq_config_update '*'
  option force_dns '1'
  list force_dns_port '53'
  list force_dns_port '853'
  list force_dns_src_interface 'lan'
  option procd_trigger_wan6 '0'
  option heartbeat_domain 'heartbeat.melmac.ca'
  option heartbeat_sleep_timeout '10'
  option heartbeat_wait_timeout '10'
  option user 'nobody'
  option group 'nogroup'
  option listen_addr '127.0.0.1'

config https-dns-proxy
  option bootstrap_dns '1.1.1.1,1.0.0.1'
  option resolver_url 'https://cloudflare-dns.com/dns-query'
  option listen_port '5053'

config https-dns-proxy
  option bootstrap_dns '8.8.8.8,8.8.4.4'
  option resolver_url 'https://dns.google/dns-query'
  option listen_port '5054'
```

### General Settings

#### canary_domains_icloud

This setting enables router to block requests to iCloud Private Relay canary domains, indicating that the local device should use the router's DNS resolution (encrypted with `https-dns-proxy`) instead of the encrypted/proprietary iCloud Private Relay resolvers. This is set to `1` (enabled) by default. Shown in WebUI and processed only if `force_dns` is also set to 1.

#### canary_domains_mozilla

This setting enables router to block requests to Mozilla canary domains, indicating that the local device should use the router's dns resolution (encrypted with `https-dns-proxy`) instead of the encrypted Mozilla resolvers. This is set to `1` (enabled) by default. Shown in WebUI and processed only if `force_dns` is also set to 1.

#### dnsmasq_config_update

The `dnsmasq_config_update` option can be set to dash (set to `'-'` to not change `dnsmasq` server settings on start/stop), can be set to `'*'` to affect all `dnsmasq` instance server settings or have a space-separated list of `dnsmasq` instances or named sections to affect (like `'0 4 5'` or `'0 backup_dns 5'`). If this option is omitted, the default setting is `'*'`. When the service is set to update the `dnsmasq` servers setting on start/stop, it does not override entries which contain either `#` or `/`, so the entries like listed below will be kept in use:

```test
  list server '/onion/127.0.0.1#65453'
  list server '/openwrt.org/8.8.8.8'
  list server '/pool.ntp.org/8.8.8.8'
  list server '127.0.0.1#15353'
  list server '127.0.0.1#55353'
  list server '127.0.0.1#65353'
```

#### force_dns

The `force_dns` setting is used to force the router's default resolver to all connected devices even if they are set to use other DNS resolvers or if other DNS resolvers are hardcoded in connected devices' settings. This is set to `1` (enabled) by default. You can additionally control which ports the `force_dns` setting should be active on with the [force_dns_port](#force_dns_port) setting below.

#### force_dns_port

You can control which ports the `force_dns` setting is active on, the default values are `53` (regular DNS) and `853` (DNS over TLS). If the listed port is open/active on the OpenWrt device, the service will create a `redirect` to the indicated port number, otherwise the service will create a `REJECT` rule. The intention for `REJECT` is that if the encrypted DNS requests has failed for your local device, it will fall-back on an unencrypted DNS request which will be then intercepted by the router and sent to the https-dns-proxy service.

#### force_dns_src_interface

This option allows you to override the interface (`lan` by default) which is used in the PROCD firewall redirects/rules the service creates if `force_dns` is enabled. Only needed if you have renamed or deleted your `lan` interface. If you indicate more than one interface, separate them by spaces.

#### procd_trigger_wan6

The service is restarted on WAN interface updates. As [OpenWrt may have floods of WAN6 updates](https://github.com/openwrt/openwrt/issues/5723#issuecomment-1040233237), the workaround for having the service restarted (and cause two `dnsmasq` restarts in turn) was to implement the `procd_trigger_wan6` boolean option (set to '0' as default) to enable/disable service restarts to be triggered by the WAN6 updates.

#### heartbeat_domain

The `heartbeat_domain` setting specifies the domain used for connectivity checks after the service starts. If set to `-`, connectivity checks are disabled. Default: `heartbeat.melmac.ca`.

#### heartbeat_sleep_timeout

The `heartbeat_sleep_timeout` setting specifies the time in seconds to wait before performing the connectivity check. Default: `10`.

#### heartbeat_wait_timeout

The `heartbeat_wait_timeout` setting specifies the time in seconds to wait for the connectivity check response. Default: `30`.

### Global Settings for Instances

If defined in the main config, the settings below are inherited by all instances. So if you want all your instances to use the same setting for let's say `listen_addr`, you can define it in the main config and it will be inherited by all instances.

| Option             | Type                 | Default   | Description                                                                                                                                                                                                                           |
| ------------------ | -------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| listen_addr        | IP Address           | 127.0.0.1 | The local IP address to listen to requests.                                                                                                                                                                                           |
| tcp_client_limit   | Integer [1-200]      | 20        | Number of TCP clients to serve.                                                                                                                                                                                                       |
| polling_interval   | Integer [5-3600]     | 120       | Optional polling interval of DNS servers.                                                                                                                                                                                             |
| proxy_server       | URL                  |           | Optional HTTP proxy. e.g. socks5://127.0.0.1:1080. Remote name resolution will be used if the protocolsupports it (http, https, socks4a, socks5h), otherwise initial DNS resolution will still be done via the bootstrap DNS servers. |
| force_http1        | Boolean              | 0         | Force HTTP/1.1 instead of HTTP/2. Useful with broken/outdated `curl` package. Included for posterity reasons, you will most likely not ever need it on OpenWrt.                                                                       |
| force_http3        | Boolean              | 0         | Force HTTP/3 (QUIC) instead of HTTP/2.                                                                                                                                                                                                |
| force_ipv6         | Boolean              | 0         | If set to 1, forces IPv6 DNS resolvers instead of IPv4.                                                                                                                                                                               |
| max_idle_time      | Integer [0-3600]     | 118       | Maximum idle time in seconds allowed for reusing a HTTPS connection.                                                                                                                                                                  |
| conn_loss_time     | Integer [5-60]       | 15        | Time in seconds to tolerate connection timeouts of reused connections. This option mitigates half-open TCP connection issue (e.g. WAN IP change).                                                                                     |
| ca_certs_file      | Path                 |           | Optional file containing CA certificates.                                                                                                                                                                                             |
| user               | String               | nobody    | Local user to run instance under.                                                                                                                                                                                                     |
| group              | String               | nogroup   | Local group to run instance under.                                                                                                                                                                                                    |
| verbosity          | Integer              | 0         | Logging verbosity level. Fatal = 0, error = 1, warning = 2, info = 3, debug = 4.                                                                                                                                                      |
| logfile            | Filepath             |           | Full filepath to the file to log the instance events to.                                                                                                                                                                              |
| statistic_interval | Integer [1-3600]     | 0         | Optional statistic printout interval.                                                                                                                                                                                                 |
| log_limit          | Integer [100-100000] | 0         | Flight recorder: storing desired amount of logs from all levels in memory and dumping them on fatal error or on SIGUSR2 signal.                                                                                                       |
| source_addr        | IP Address           |           | Optional source IP address for outgoing connections.                                                                                                                                                                                  |

### Instance Settings

The https-dns-proxy instance settings are:

| Option             | Type                 | Default     | Description                                                                                                                                                                                                                           |
| ------------------ | -------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| resolver_url       | URL                  |             | The https URL to the RFC8484-compatible resolver.                                                                                                                                                                                     |
| listen_port        | Port                 | 5053 and up | If this setting is omitted, the service will start the first https-dns-proxy instance on port 5053, second on 5054 and so on.                                                                                                         |
| bootstrap_dns      | IP Address           |             | The non-encrypted DNS servers to be used to resolve the DoH server name on start.                                                                                                                                                     |
| dscp_codepoint     | Integer [0-63]       |             | Optional DSCP codepoint [0-63] to set on upstream DNS server connections.                                                                                                                                                             |
| listen_addr        | IP Address           | 127.0.0.1   | The local IP address to listen to requests.                                                                                                                                                                                           |
| tcp_client_limit   | Integer [1-200]      | 20          | Number of TCP clients to serve.                                                                                                                                                                                                       |
| polling_interval   | Integer [5-3600]     | 120         | Optional polling interval of DNS servers.                                                                                                                                                                                             |
| proxy_server       | URL                  |             | Optional HTTP proxy. e.g. socks5://127.0.0.1:1080. Remote name resolution will be used if the protocolsupports it (http, https, socks4a, socks5h), otherwise initial DNS resolution will still be done via the bootstrap DNS servers. |
| force_http1        | Boolean              | 0           | Force HTTP/1.1 instead of HTTP/2. Useful with broken/outdated `curl` package. Included for posterity reasons, you will most likely not ever need it on OpenWrt.                                                                       |
| force_http3        | Boolean              | 0           | Force HTTP/3 (QUIC) instead of HTTP/2.                                                                                                                                                                                                |
| force_ipv6         | Boolean              | 0           | If set to 1, forces IPv6 DNS resolvers instead of IPv4.                                                                                                                                                                               |
| max_idle_time      | Integer [0-3600]     | 118         | Maximum idle time in seconds allowed for reusing a HTTPS connection.                                                                                                                                                                  |
| conn_loss_time     | Integer [5-60]       | 15          | Time in seconds to tolerate connection timeouts of reused connections. This option mitigates half-open TCP connection issue (e.g. WAN IP change).                                                                                     |
| ca_certs_file      | Path                 |             | Optional file containing CA certificates.                                                                                                                                                                                             |
| user               | String               | nobody      | Local user to run instance under.                                                                                                                                                                                                     |
| group              | String               | nogroup     | Local group to run instance under.                                                                                                                                                                                                    |
| verbosity          | Integer              | 0           | Logging verbosity level. Fatal = 0, error = 1, warning = 2, info = 3, debug = 4.                                                                                                                                                      |
| logfile            | Filepath             |             | Full filepath to the file to log the instance events to.                                                                                                                                                                              |
| statistic_interval | Integer [1-3600]     | 0           | Optional statistic printout interval.                                                                                                                                                                                                 |
| log_limit          | Integer [100-100000] | 0           | Flight recorder: storing desired amount of logs from all levels in memory and dumping them on fatal error or on SIGUSR2 signal.                                                                                                       |
| source_addr        | IP Address           |             | Optional source IP address for outgoing connections.                                                                                                                                                                                  |

Please also refer to the [Usage section at upstream README](https://github.com/aarond10/https_dns_proxy/blob/master/README.md#usage) which may contain additional/more details on some parameters.

## Using https-dns-proxy for ad-blocking

There are some resolvers which offer customizable ad-blocking as part of their name resolution services, returning either NXDOMAIN (domain not found) or a local IP address for the domains in their ad/malware block-lists. This is one of the best ways to implement ad-blocking on OpenWrt if you don't want the fine-grained control over your block- and allow-lists with the minimal overhead on your router. Visit the following pages to learn more what customizable ad-blocking options are available from these resolvers:

- [AhaDNS Blitz](https://blitz-setup.ahadns.com/)
- [RethinkDNS](https://www.rethinkdns.com/configure)
- [NextDNS.io](https://my.nextdns.io)

If you're using both ad-blocking and non-blocking resolvers and want the ad-blocking resolver replies to have a priority over non-blocking resolver replies, it is highly recommended to:

- move your ad-blocking resolvers to the top of the list of resolvers at the WebUI page or config file
- set the `strictorder` option for your `dhcp.@dnsmasq[..]` instances to `1`

If you have any questions about setting up `https-dns-proxy` for use with any of the above customizable ad-blocker resolvers, feel free to post in the OpenWrt forum using the link in the [Getting Help](#getting-help) section below.

If you do want full control over your block- and allow-lists with the minimal footprint package on your router, use the [adblock-fast](https://docs.openwrt.melmac.net/adblock-fast/) package and its WebUI: `luci-app-adblock-fast`.

## <a name='Donate'></a>Donate

If you find `https-dns-proxy` or `luci-app-https-dns-proxy`, know that your help is needed. Please consider donating to support development of this project. I've been developing it in my spare time without any external funding, outside of my GitHub sponsors. You can donate by:

- Sponsoring me on GitHub with [monthly donation](https://github.com/sponsors/stangri?frequency=recurring&sponsor=stangri).
- Sponsoring me on GitHub with [one-time donation](https://github.com/sponsors/stangri?frequency=one-time&sponsor=stangri).
- Sending a donation [thru PayPal](https://paypal.me/stan).

## Getting Help

If things are not working as intended, please run the following commands and include their output in your post/issue:

```sh
ubus call system board
curl -V
dnsmasq --version
https-dns-proxy -V
uci export dhcp
uci export network
uci export https-dns-proxy
service https-dns-proxy status
service https-dns-proxy info
nslookup google.com 127.0.0.1:5053
```

You can post your issues/concerns in the [OpenWrt Forum Thread](https://forum.openwrt.org/t/doh-proxy-https-dns-proxy-new-rfc8484-supporting-package-and-web-ui/50832).

## Thanks

This OpenWrt package wouldn't have been possible without contribitions from [@aarond10](https://github.com/aarond10) and [@baranyaib90](https://github.com/baranyaib90) to the upstream [https_dns_proxy](https://github.com/aarond10/https_dns_proxy) package and their active participation in the OpenWrt package development. Thanks to [@oldium](https://github.com/oldium) and [@curtdept](https://github.com/curtdept) for their contributions to the OpenWrt package. Special thanks to [@jow-](https://github.com/jow-) for general package/luci guidance.

<!-- markdownlint-disable MD033 -->

<script defer src='https://static.cloudflareinsights.com/beacon.min.js' data-cf-beacon='{"token": "911798f2c34b45338f8f8182830a3eb6"}'></script>
