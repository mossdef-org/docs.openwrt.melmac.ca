# mwan4 - Multi-WAN with nftables

## Migration from mwan3 to mwan4

### Configuration File Changes

The mwan4 configuration file format is **mostly backward compatible** with mwan3, with the following changes:

#### 1. `flush_conntrack` Option Format (Breaking Change)

**mwan3 format:**
```
config interface 'wan'
	option flush_conntrack 'always'  # or 'never'
```

**mwan4 format:**
```
config interface 'wan'
	list flush_conntrack 'ifup'    # optional
	list flush_conntrack 'ifdown'  # optional
```

**Migration:** The uci-defaults script (`/etc/uci-defaults/90-mwan4`) automatically converts:
- `always` → adds both `ifup` and `ifdown` to the list
- `never` → removes the option entirely

This change provides more granular control over when connection tracking is flushed.

#### 2. `family` Configuration on Interfaces - Dual-Stack Support (New)

**Single family per interface (legacy approach):**
```
config interface 'wan'
	option enabled '1'
	list family 'ipv4'
	list track_ip '1.1.1.1'

config interface 'wan6'
	option enabled '1'
	list family 'ipv6'
	list track_ip '2606:4700:4700::1111'
```

**Dual-stack on single interface (NEW - recommended):**
```
config interface 'wan'
	option enabled '1'
	list family 'ipv4'
	list family 'ipv6'
	list track_ip '1.1.1.1'           # IPv4 tracking
	list track_ip '2606:4700:4700::1111'  # IPv6 tracking
	option reliability '2'
```

**Benefits of dual-stack configuration:**
- Single configuration for both IPv4 and IPv6
- No duplicate settings to maintain
- Automatic family detection from track IPs
- Independent tracking per family
- Single interface ID with family-specific internal chains

**Backward compatibility:** The old `option family` format is automatically migrated to `list family` by the uci-defaults script.

#### 3. `family` Option on Rules (New)

**Allows rules to be IPv4 or IPv6 specific:**
```
config rule 'default_rule_v4'
	option dest_ip '0.0.0.0/0'
	option use_policy 'balanced'
	option family 'ipv4'

config rule 'default_rule_v6'
	option dest_ip '::/0'
	option use_policy 'balanced'
	option family 'ipv6'
```

Rules without a `family` option will apply to both IPv4 and IPv6 (if the rule is compatible with both).

#### 4. `mmx_mask` Now Configurable (New)

**Customizable packet marking mask:**
```
config globals 'globals'
	option mmx_mask '0x3F00'
```

Previously hardcoded, the marking mask can now be customized in the globals section. This allows integration with other routing systems that may use different parts of the packet mark.

**Default value:** `0x3F00` (supports up to 63 interfaces)

### Unchanged Configuration Sections

The following configuration sections remain fully compatible:

- **Interface tracking options:** `track_ip`, `reliability`, `count`, `timeout`, `interval`, `down`, `up`, `size`, `max_ttl`, `failure_interval`, `recovery_interval`, `check_quality`, `failure_latency`, `recovery_latency`, `failure_loss`, `recovery_loss`
- **Member definitions:** `interface`, `metric`, `weight`
- **Policy definitions:** `use_member` lists
- **Rule structure:** `sticky`, `timeout`, `dest_port`, `src_port`, `dest_ip`, `src_ip`, `proto`, `use_policy`, `ipset`, `logging`

### Example Migration

**mwan3 config:**
```
config interface 'wan'
	option enabled '1'
	list track_ip '1.1.1.1'
	option flush_conntrack 'always'
```

**mwan4 config (after automatic migration):**
```
config interface 'wan'
	option enabled '1'
	option family 'ipv4'
	list track_ip '1.1.1.1'
	list flush_conntrack 'ifup'
	list flush_conntrack 'ifdown'
```

### Documentation

For full mwan3 configuration documentation (still largely applicable):
https://openwrt.org/docs/guide-user/network/wan/multiwan/mwan3#mwan3_configuration
