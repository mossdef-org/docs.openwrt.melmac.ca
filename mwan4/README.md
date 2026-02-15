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

#### 3. Renamed Config Sections: `member` → `route`, `policy` → `strategy` (Breaking Change)

**mwan3 / old mwan4 format:**
```
config member 'wan_m1_w3'
	option interface 'wan'
	option metric '1'
	option weight '3'

config policy 'balanced'
	list use_member 'wan_m1_w3'

config rule 'https'
	option use_policy 'balanced'
```

**New mwan4 format:**
```
config route 'wan_m1_w3'
	option interface 'wan'
	option metric '1'
	option weight '3'

config strategy 'balanced'
	list use_route 'wan_m1_w3'

config rule 'https'
	option use_strategy 'balanced'
```

**Summary of renames:**
| Old | New | Description |
|-----|-----|-------------|
| `config member` | `config route` | Weighted route entry (interface + metric + weight) |
| `config policy` | `config strategy` | Routing strategy (how to distribute/failover traffic) |
| `list use_member` | `list use_route` | Route references within a strategy |
| `option use_policy` | `option use_strategy` | Strategy reference within a rule |

**Migration:** The uci-defaults script (`/etc/uci-defaults/92-mwan4-rename`) automatically converts old section types and option names in-place.

#### 4. `family` on Rules Uses List Format (New)

**Allows rules to be IPv4 or IPv6 specific (using list format):**
```
config rule 'default_rule_v4'
	option dest_ip '0.0.0.0/0'
	option use_strategy 'balanced'
	list family 'ipv4'

config rule 'default_rule_v6'
	option dest_ip '::/0'
	option use_strategy 'balanced'
	list family 'ipv6'
```

Rules without a `family` list will apply to both IPv4 and IPv6 (if the rule is compatible with both).

**Migration:** The uci-defaults script automatically converts the old `option family` format to `list family`.

#### 5. `mmx_mask` Now Configurable (New)

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
- **Route definitions (formerly member):** `interface`, `metric`, `weight`
- **Strategy definitions (formerly policy):** `use_route` lists, `last_resort`
- **Rule structure:** `sticky`, `timeout`, `dest_port`, `src_port`, `dest_ip`, `src_ip`, `proto`, `use_strategy`, `ipset`, `logging`

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
	list family 'ipv4'
	list track_ip '1.1.1.1'
	list flush_conntrack 'ifup'
	list flush_conntrack 'ifdown'
```

### Dual-Stack Configuration Example

Here's a complete example showing how to configure a dual-stack WAN interface with both IPv4 and IPv6:

```
config interface 'wan'
	option enabled '1'
	list family 'ipv4'
	list family 'ipv6'
	list track_ip '1.1.1.1'
	list track_ip '8.8.8.8'
	list track_ip '2606:4700:4700::1111'
	list track_ip '2001:4860:4860::8888'
	option reliability '2'
	option count '1'
	option timeout '2'
	option interval '5'
	option down '3'
	option up '8'

config route 'wan_m1_w1'
	option interface 'wan'
	option metric '1'
	option weight '1'

config strategy 'balanced'
	list use_route 'wan_m1_w1'

config rule 'default_rule_v4'
	option dest_ip '0.0.0.0/0'
	option use_strategy 'balanced'
	list family 'ipv4'

config rule 'default_rule_v6'
	option dest_ip '::/0'
	option use_strategy 'balanced'
	list family 'ipv6'
```

**How it works:**
- The interface spawns two tracking instances: `track_wan_ipv4` and `track_wan_ipv6`
- IPv4 track IPs (1.1.1.1, 8.8.8.8) are automatically used for IPv4 tracking
- IPv6 track IPs (2606:..., 2001:...) are automatically used for IPv6 tracking
- Each family maintains independent online/offline status
- nftables chains are created per family: `mwan4_iface_in_wan_ipv4` and `mwan4_iface_in_wan_ipv6`
- Status is reported separately per family: `mwan4 status` will show both

## nftables Integration

mwan4 integrates with firewall4 by writing nftables rules into the shared `inet fw4` table. All chains and sets use the `mwan4_` prefix.

### Generated nft Files

mwan4 generates persistent nftables rule files that are loaded by fw4 via the `ruleset-post` mechanism:

| File | Description |
|------|-------------|
| `/usr/share/nftables.d/ruleset-post/10-mwan4-base.nft` | Base chains and sets: `mwan4_prerouting` (mangle hook), `mwan4_output` (route hook), `mwan4_ifaces_in`, `mwan4_rules`, connected/custom/dynamic IP sets for both IPv4 and IPv6. Created at startup. |
| `/usr/share/nftables.d/ruleset-post/11-mwan4-interfaces.nft` | Per-interface chains (e.g. `mwan4_iface_in_wan_ipv4`, `mwan4_iface_in_wan_ipv6`). Each chain marks traffic from a specific interface/family with the interface's fwmark. Created on interface ifup. |
| `/usr/share/nftables.d/ruleset-post/12-mwan4-strategies.nft` | Strategy chains (e.g. `mwan4_strategy_balanced_ipv4`). Each chain implements load balancing via `numgen random` or failover by setting fwmarks for the selected route. Rebuilt whenever interface status changes. |
| `/usr/share/nftables.d/ruleset-post/13-mwan4-rules.nft` | User rule chains (`mwan4_rules_ipv4`, `mwan4_rules_ipv6`). Maps traffic matching criteria (dest/src IP, port, protocol) to strategy chains. Includes sticky session sets when configured. |

### Runtime Temp Files

| File | Description |
|------|-------------|
| `/var/run/mwan4.nft` | Temporary nft file prefix used during atomic rule generation. Individual operations append suffixes (e.g. `.strategy_balanced_ipv4`, `.user_rules`, `.rule_*_sticky`, `.connected_v4`, `.custom_sets`). Cleaned up after each operation. |

### nftables Chain Naming

| Chain Pattern | Purpose |
|---------------|---------|
| `mwan4_prerouting` | Main hook at mangle priority — restores marks, processes interfaces, checks sets, applies rules |
| `mwan4_output` | Output hook at mangle priority — same logic for locally-generated traffic |
| `mwan4_ifaces_in` | Dispatches to per-interface chains based on input device |
| `mwan4_iface_in_{iface}_{family}` | Marks traffic arriving on a specific interface/family |
| `mwan4_strategy_{name}_{family}` | Implements a routing strategy (load balance / failover) |
| `mwan4_rules_{family}` | Evaluates user-defined rules and jumps to strategy chains |
| `mwan4_rule_{name}_{family}` | Per-rule sticky session chain (only when sticky is enabled) |

### nftables Set Naming

| Set Pattern | Purpose |
|-------------|---------|
| `mwan4_connected_{family}` | Directly connected networks — traffic to these is marked default |
| `mwan4_custom_{family}` | Networks from custom routing table lookups (`rt_table_lookup`) |
| `mwan4_dynamic_{family}` | Dynamically added networks |
| `mwan4_rule_{family}_{rule}` | Sticky session map (IP + mark → timeout) for a specific rule |

### Documentation

For full mwan3 configuration documentation (still largely applicable):
https://openwrt.org/docs/guide-user/network/wan/multiwan/mwan3#mwan3_configuration
