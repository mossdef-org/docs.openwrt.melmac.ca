# mwan3 to mwan4 (nftables) Transition & PBR Integration Analysis

## Executive Summary

This document analyzes the requirements for transitioning the mwan3 package from iptables to nftables (mwan4), and explores the optimal integration strategy with the pbr (Policy-Based Routing) package. The goal is to have mwan4 manage all interface tracking, routing tables, metrics, and markings, while pbr creates atomic nftables fw4 files for routing policies.

**CRITICAL DESIGN CONSTRAINT**: nftables does NOT allow `jump` or `goto` between different tables. Therefore, mwan4 **MUST** use the same `table inet fw4` as pbr and firewall4, rather than creating its own `table inet mwan4`. This shared table approach is already proven by pbr's implementation.

---

## 1. Current mwan3 Architecture (iptables-based)

### 1.1 Key Components

#### Firewall Marking System
- **Technology**: iptables mangle table with fwmark (firewall marks)
- **Default Mask**: `MMX_MASK = 0x3F00`
- **Mark Allocation**:
  - Interface marks: `0x0100` to `0x3D00` (max 61 interfaces)
  - Reserved marks:
    - `MMX_DEFAULT`: Last possible mark (all bits set in mask)
    - `MMX_BLACKHOLE`: Second to last mark
    - `MMX_UNREACHABLE`: Third to last mark

#### Core iptables Chains

```
mwan3_hook (PREROUTING, OUTPUT)
├── mwan3_ifaces_in
│   ├── mwan3_iface_in_wan1
│   ├── mwan3_iface_in_wan2
│   └── mwan3_iface_in_<interface>
├── mwan3_custom_ipv4/ipv6
├── mwan3_connected_ipv4/ipv6
├── mwan3_dynamic_ipv4/ipv6
└── mwan3_rules
    ├── mwan3_policy_balanced
    ├── mwan3_policy_wan1_only
    └── mwan3_rule_<sticky_rule>
```

**Chain Descriptions**:
- `mwan3_hook`: Main entry point attached to PREROUTING and OUTPUT
- `mwan3_ifaces_in`: Container for per-interface input marking chains
- `mwan3_iface_in_<interface>`: Marks incoming packets on specific interfaces
- `mwan3_rules`: User-defined policy rules
- `mwan3_policy_<policy>`: Policy chains with weighted load balancing
- `mwan3_custom_ipv4/ipv6`: Custom routing table lookup sets
- `mwan3_connected_ipv4/ipv6`: Connected network sets
- `mwan3_dynamic_ipv4/ipv6`: Dynamic sets for runtime updates

#### ipset Usage

Extensive use of ipset for various purposes:

```bash
# Connected networks
ipset create mwan3_connected_ipv4 hash:net
ipset create mwan3_connected_ipv6 hash:net family inet6

# Custom routing table lookups
ipset create mwan3_custom_ipv4 hash:net
ipset create mwan3_custom_ipv6 hash:net family inet6

# Dynamic sets
ipset create mwan3_dynamic_ipv4 list:set
ipset create mwan3_dynamic_ipv6 hash:net family inet6

# Sticky session tracking (per rule)
ipset create mwan3_rule_ipv4_<rule> hash:ip,mark markmask $MMX_MASK timeout 600
ipset create mwan3_rule_ipv6_<rule> hash:ip,mark markmask $MMX_MASK timeout 600 family inet6
```

#### Routing Infrastructure

**Per-interface Routing Tables**:
- Table IDs: 1 to N (based on interface ID from config order)
- Stored in `/etc/iproute2/rt_tables`

**IP Rule Priority Scheme**:
```bash
# For interface ID = 1:
ip rule add pref 1001 iif <device> lookup 1           # Input interface match
ip rule add pref 2001 fwmark 0x100/0x3F00 lookup 1   # Fwmark match
ip rule add pref 3001 fwmark 0x100/0x3F00 unreachable # Unreachable fallback

# Special rules (same for all interfaces):
ip rule add pref 2254 fwmark 0x3E00/0x3F00 blackhole     # MM_BLACKHOLE
ip rule add pref 2255 fwmark 0x3F00/0x3F00 unreachable  # MM_UNREACHABLE
```

**Formula**:
- Input interface rule: `priority = interface_id + 1000`
- Fwmark rule: `priority = interface_id + 2000`
- Unreachable rule: `priority = interface_id + 3000`
- Blackhole rule: `priority = MM_BLACKHOLE + 2000`
- Unreachable rule: `priority = MM_UNREACHABLE + 2000`

### 1.2 Key Implementation Files

| File | Purpose |
|------|---------|
| `files/lib/mwan3/common.sh` | Shared constants, utilities, fwmark calculations |
| `files/lib/mwan3/mwan3.sh` | Core iptables/routing logic, policy management |
| `files/etc/init.d/mwan3` | Service init script, start/stop logic |
| `files/usr/sbin/mwan3` | CLI interface for status and control |
| `files/usr/sbin/mwan3track` | Interface tracking daemon |
| `files/usr/sbin/mwan3rtmon` | Route monitoring daemon |

### 1.3 Core Functions

**Mark Allocation** (`mwan3_id2mask`):
```bash
# Maps interface ID to only use bits set in MMX_MASK
# Example: ID=1, MASK=0x3F00 → 0x0100
# Example: ID=2, MASK=0x3F00 → 0x0200
```

**iptables-restore Usage**:
```bash
# Build update string incrementally
update="*mangle"
update="$update"$'\n'"-N mwan3_hook"
update="$update"$'\n'"-A PREROUTING -j mwan3_hook"
update="$update"$'\n'"COMMIT"

# Apply atomically
echo "$update" | iptables-restore -T mangle -w -n
```

### 1.4 Load Balancing Implementation

Current iptables approach uses statistic module:

```bash
# Policy with two members, equal weight (1:1)
-A mwan3_policy_balanced \
   -m mark --mark 0x0/$MMX_MASK \
   -m statistic --mode random --probability 0.5 \
   -m comment --comment "wan1 1 2" \
   -j MARK --set-xmark 0x100/0x3F00

-A mwan3_policy_balanced \
   -m mark --mark 0x0/$MMX_MASK \
   -m comment --comment "wan2 1 2" \
   -j MARK --set-xmark 0x200/0x3F00
```

**Weight Calculation**:
```bash
probability = (weight * 1000) / total_weight
# Example: weight=1, total=2 → probability=0.5
# Example: weight=3, total=5 → probability=0.6
```

---

## 2. Transition Requirements: mwan3 → mwan4 (nftables)

### 2.1 Replace iptables with nftables

#### A. Critical nftables Limitation: No Cross-Table Jumps

**IMPORTANT**: Unlike iptables, nftables does **NOT** allow `jump` or `goto` statements between different tables.

**Example of what DOES NOT WORK**:
```nft
table inet mwan4 {
    chain mwan4_mark_0x100 { ... }
}

table inet fw4 {
    chain pbr_rules {
        # This FAILS - cannot jump to chain in different table
        ip daddr @netflix goto mwan4_mark_0x100  # ❌ ERROR!
    }
}
```

**The ONLY Solution**: Both packages must add chains to the **same table**:
```nft
table inet fw4 {
    chain mwan4_mark_0x100 { ... }  # mwan4 adds this

    chain pbr_rules {
        # This WORKS - both chains in same table
        ip daddr @netflix goto mwan4_mark_0x100  # ✓ OK
    }
}
```

This is why pbr already uses `table inet fw4` instead of creating `table inet pbr`. mwan4 must follow the same pattern.

#### B. Table Structure

**Current (iptables)**:
```bash
IPT4="iptables -t mangle -w"
IPT6="ip6tables -t mangle -w"
IPT4R="iptables-restore -T mangle -w -n"
IPT6R="ip6tables-restore -T mangle -w -n"
```

**New (nftables)**:
```nft
# CRITICAL: mwan4 MUST use the same table as fw4/pbr
# nftables does NOT allow jumping between different tables
table inet fw4 {
    # Combined IPv4/IPv6 in single table
    # Shared with firewall4 and pbr
}
```

#### B. Chain Migration

**Hook Chains**:
```nft
# IMPORTANT: Must use 'table inet fw4' (shared with firewall4/pbr)
# NOT 'table inet mwan4' - cannot jump between tables in nftables!
table inet fw4 {
    chain mwan4_prerouting {
        type filter hook prerouting priority mangle; policy accept;

        # IPv6 Router Advertisement exemptions
        ip6 nexthdr icmpv6 icmpv6 type {
            nd-router-solicit,
            nd-router-advert,
            nd-neighbor-solicit,
            nd-neighbor-advert,
            nd-redirect
        } return

        # Restore mark from conntrack
        meta mark & $MMX_MASK == 0 ct mark set meta mark & $MMX_MASK

        # Interface input processing
        meta mark & $MMX_MASK == 0 jump mwan4_ifaces_in

        # Custom/connected/dynamic sets
        meta mark & $MMX_MASK == 0 jump mwan4_custom_ipv4
        meta mark & $MMX_MASK == 0 jump mwan4_connected_ipv4
        meta mark & $MMX_MASK == 0 jump mwan4_dynamic_ipv4

        # IPv6 equivalents (if enabled)
        meta mark & $MMX_MASK == 0 jump mwan4_custom_ipv6
        meta mark & $MMX_MASK == 0 jump mwan4_connected_ipv6
        meta mark & $MMX_MASK == 0 jump mwan4_dynamic_ipv6

        # User rules
        meta mark & $MMX_MASK == 0 jump mwan4_rules

        # Save mark to conntrack
        ct mark set meta mark & $MMX_MASK

        # Post-marking custom/connected/dynamic checks
        meta mark != $MMX_DEFAULT & $MMX_MASK jump mwan4_custom_ipv4
        meta mark != $MMX_DEFAULT & $MMX_MASK jump mwan4_connected_ipv4
        meta mark != $MMX_DEFAULT & $MMX_MASK jump mwan4_dynamic_ipv4
    }

    chain mwan4_output {
        type route hook output priority mangle; policy accept;

        # Similar structure to prerouting
        # IPv6 RA exemptions
        ip6 nexthdr icmpv6 icmpv6 type {
            nd-router-solicit,
            nd-router-advert,
            nd-neighbor-solicit,
            nd-neighbor-advert,
            nd-redirect
        } return

        meta mark & $MMX_MASK == 0 ct mark set meta mark & $MMX_MASK
        meta mark & $MMX_MASK == 0 jump mwan4_ifaces_in
        meta mark & $MMX_MASK == 0 jump mwan4_custom_ipv4
        meta mark & $MMX_MASK == 0 jump mwan4_connected_ipv4
        meta mark & $MMX_MASK == 0 jump mwan4_dynamic_ipv4
        meta mark & $MMX_MASK == 0 jump mwan4_rules
        ct mark set meta mark & $MMX_MASK
        meta mark != $MMX_DEFAULT & $MMX_MASK jump mwan4_custom_ipv4
        meta mark != $MMX_DEFAULT & $MMX_MASK jump mwan4_connected_ipv4
        meta mark != $MMX_DEFAULT & $MMX_MASK jump mwan4_dynamic_ipv4
    }
}
```

**Interface Chains**:
```nft
chain mwan4_ifaces_in {
    meta mark & $MMX_MASK == 0 jump mwan4_iface_in_wan1
    meta mark & $MMX_MASK == 0 jump mwan4_iface_in_wan2
    # ... per interface
}

chain mwan4_iface_in_wan1 {
    # Check source against custom/connected/dynamic sets
    iifname "eth0" ip saddr @mwan4_custom_ipv4 meta mark 0 \
        comment "default" \
        meta mark set (meta mark & ~$MMX_MASK) | $MMX_DEFAULT

    iifname "eth0" ip saddr @mwan4_connected_ipv4 meta mark 0 \
        comment "default" \
        meta mark set (meta mark & ~$MMX_MASK) | $MMX_DEFAULT

    iifname "eth0" ip saddr @mwan4_dynamic_ipv4 meta mark 0 \
        comment "default" \
        meta mark set (meta mark & ~$MMX_MASK) | $MMX_DEFAULT

    # Mark packets from this interface
    iifname "eth0" meta mark 0 \
        comment "wan1" \
        meta mark set (meta mark & ~$MMX_MASK) | 0x100
}
```

### 2.2 Convert ipset to nft sets

#### A. Basic Sets

**Connected Networks**:
```nft
set mwan4_connected_ipv4 {
    type ipv4_addr
    flags interval
    auto-merge
}

set mwan4_connected_ipv6 {
    type ipv6_addr
    flags interval
    auto-merge
}
```

**Custom Routing Tables**:
```nft
set mwan4_custom_ipv4 {
    type ipv4_addr
    flags interval
    auto-merge
}

set mwan4_custom_ipv6 {
    type ipv6_addr
    flags interval
    auto-merge
}
```

**Dynamic Sets**:
```nft
set mwan4_dynamic_ipv4 {
    type ipv4_addr
    flags interval
    auto-merge
}

set mwan4_dynamic_ipv6 {
    type ipv6_addr
    flags interval
    auto-merge
}
```

#### B. Sticky Session Sets (Complex)

**Challenge**: ipset `hash:ip,mark` with timeout needs to become nft set with tuple.

**Solution**:
```nft
# For sticky sessions per rule
set mwan4_rule_ipv4_netflix {
    type ipv4_addr . mark
    flags timeout
    timeout 600s
}

set mwan4_rule_ipv6_netflix {
    type ipv6_addr . mark
    flags timeout
    timeout 600s
}
```

**Usage in Rules**:
```nft
chain mwan4_rule_netflix {
    # Check if source IP has existing mark
    meta mark & $MMX_MASK != 0 ip saddr . meta mark @mwan4_rule_ipv4_netflix \
        meta mark set 0

    # If no mark, apply policy
    meta mark 0 jump mwan4_policy_balanced

    # Delete non-matching marks from set
    meta mark != 0xfc00 & 0xfc00 \
        delete @mwan4_rule_ipv4_netflix { ip saddr . meta mark }

    # Add current mark to set
    meta mark != 0xfc00 & 0xfc00 \
        add @mwan4_rule_ipv4_netflix { ip saddr . meta mark timeout 600s }
}
```

### 2.3 Policy Chains with Load Balancing

This is the most complex conversion due to the statistic module differences.

#### A. iptables Statistic Module

```bash
# Current approach: use random probability
-A mwan3_policy_balanced \
   -m mark --mark 0x0/$MMX_MASK \
   -m statistic --mode random --probability 0.5 \
   -m comment --comment "wan1 1 2" \
   -j MARK --set-xmark 0x100/0x3F00

-A mwan3_policy_balanced \
   -m mark --mark 0x0/$MMX_MASK \
   -m comment --comment "wan2 1 2" \
   -j MARK --set-xmark 0x200/0x3F00
```

#### B. nftables numgen

**Equal Weight (1:1)**:
```nft
chain mwan4_policy_balanced {
    # Two interfaces, equal weight
    meta mark & $MMX_MASK == 0 \
        numgen random mod 2 0 \
        meta mark set (meta mark & ~$MMX_MASK) | 0x100 \
        comment "wan1 1 2"

    meta mark & $MMX_MASK == 0 \
        meta mark set (meta mark & ~$MMX_MASK) | 0x200 \
        comment "wan2 1 2"
}
```

**Weighted (3:1)**:
```nft
chain mwan4_policy_weighted {
    # Three interfaces: wan1 (weight 3), wan2 (weight 1), total 4
    meta mark & $MMX_MASK == 0 \
        numgen random mod 4 0-2 \
        meta mark set (meta mark & ~$MMX_MASK) | 0x100 \
        comment "wan1 3 4"

    meta mark & $MMX_MASK == 0 \
        meta mark set (meta mark & ~$MMX_MASK) | 0x200 \
        comment "wan2 1 4"
}
```

**Last Resort Handling**:
```nft
chain mwan4_policy_balanced {
    # ... weight-based rules above ...

    # Last resort: blackhole
    meta mark & $MMX_MASK == 0 \
        comment "blackhole" \
        meta mark set (meta mark & ~$MMX_MASK) | $MMX_BLACKHOLE

    # Or: default
    meta mark & $MMX_MASK == 0 \
        comment "default" \
        meta mark set (meta mark & ~$MMX_MASK) | $MMX_DEFAULT

    # Or: unreachable
    meta mark & $MMX_MASK == 0 \
        comment "unreachable" \
        meta mark set (meta mark & ~$MMX_MASK) | $MMX_UNREACHABLE
}
```

### 2.4 User Rules Migration

#### A. Current iptables Rules

```bash
# Example: Route Netflix traffic via wan1
-A mwan3_rules \
   -p tcp \
   -m set --match-set netflix_domains dst \
   -m mark --mark 0/$MMX_MASK \
   -j mwan3_policy_wan1_only

# With sticky sessions
-A mwan3_rules \
   -s 192.168.1.100 \
   -m mark --mark 0/$MMX_MASK \
   -j mwan3_rule_sticky_client
```

#### B. nftables Equivalent

```nft
chain mwan4_rules {
    # Route Netflix via wan1
    ip daddr @netflix_domains \
        meta mark 0 \
        jump mwan4_policy_wan1_only

    # Sticky session for specific client
    ip saddr 192.168.1.100 \
        meta mark 0 \
        jump mwan4_rule_sticky_client
}
```

### 2.5 Routing Infrastructure (Unchanged)

The `ip route` and `ip rule` commands remain largely the same:

```bash
# Still create per-interface routing tables
ip route add default via $GATEWAY dev $DEVICE table $TABLE_ID

# Still use fwmark-based rules
ip rule add fwmark $MARK/$MASK table $TABLE_ID priority $PRIORITY

# Still use input interface rules
ip rule add iif $DEVICE table $TABLE_ID priority $PRIORITY

# Still use unreachable fallback
ip rule add fwmark $MARK/$MASK unreachable priority $PRIORITY
```

### 2.6 Migration Challenges

| Challenge | iptables | nftables | Solution |
|-----------|----------|----------|----------|
| **Atomic Updates** | iptables-restore batch | nft -f file | Generate complete ruleset, apply atomically |
| **Table Isolation** | Can use separate tables | Must share same table | Use `table inet fw4` (cannot jump between tables) |
| **Statistics** | -m statistic --probability | numgen random mod | Convert probability to modulo ranges |
| **IPv4/IPv6 Separation** | Separate tables | Single inet table | Use `ip`/`ip6` selectors in rules |
| **Set Syntax** | ipset commands | nft set definitions | Convert to nft set syntax |
| **Sticky Sessions** | hash:ip,mark ipset | tuple set with timeout | Use type `ipv4_addr . mark` |
| **Debugging** | iptables -L -v -n | nft list ruleset | Update logging/debugging tools |

### 2.7 Script Structure Changes

#### Current (iptables-restore)
```bash
update="*mangle"
mwan3_push_update() {
    update="$update"$'\n'"$*"
}
mwan3_push_update "-N mwan3_hook"
mwan3_push_update "-A PREROUTING -j mwan3_hook"
mwan3_push_update "COMMIT"

echo "$update" | iptables-restore -T mangle -w -n
```

#### New (nft -f)
```bash
nft_file="/var/run/mwan4.nft"

mwan4_nft_write() {
    echo "$*" >> "$nft_file"
}

# Generate complete ruleset
# CRITICAL: Use 'table inet fw4' NOT 'table inet mwan4'
cat > "$nft_file" <<'EOF'
table inet fw4 {
    # Sets first
    set mwan4_connected_ipv4 {
        type ipv4_addr
        flags interval
    }

    # Then chains (added to shared fw4 table)
    chain mwan4_prerouting {
        type filter hook prerouting priority mangle;
        # ... rules ...
    }
}
EOF

# Apply atomically via fw4
fw4 reload
```

---

## 3. Integration with PBR Package

### 3.0 Critical Integration Requirement

**⚠️ NFTABLES LIMITATION**: You **CANNOT** use `jump` or `goto` between different nftables tables.

This means:
- ❌ **WRONG**: mwan4 creates `table inet mwan4`, pbr uses `table inet fw4` → **Cannot reference each other's chains**
- ✅ **CORRECT**: Both mwan4 and pbr add chains to the **same** `table inet fw4` → **Can freely reference chains**

**Real-World Proof**: pbr already encountered this limitation. That's why pbr uses `table inet fw4` instead of creating `table inet pbr`. mwan4 must use the same approach.

**Example**:
```nft
# What pbr needs to do (and currently does):
table inet fw4 {
    # pbr adds its chains to fw4
    chain pbr_rules {
        ip daddr @netflix goto mwan4_mark_0x100  # Must reference mwan4's chain
    }

    # mwan4 MUST add its chains to fw4 (not a separate table)
    chain mwan4_mark_0x100 {
        meta mark set (meta mark & 0xFFFFFF00) | 0x100
    }
}
```

### 3.1 Current PBR Architecture (nftables-based)

PBR is already using nftables exclusively:

#### A. Table Structure
- **Table**: `inet fw4` (firewall4 table - shared with OpenWrt firewall)
- **Priority**: `mangle + N` (runs after base firewall)

#### B. Chain Structure
```nft
table inet fw4 {
    chain pbr_prerouting {
        type filter hook prerouting priority mangle + 5;
        meta mark & $fw_mask != 0 return  # Skip if already marked
        jump pbr_rules
    }

    chain pbr_forward {
        type filter hook forward priority mangle + 5;
        # Forward chain processing
    }

    chain pbr_output {
        type route hook output priority mangle + 5;
        meta mark & $fw_mask != 0 return
        jump pbr_rules
    }

    chain pbr_mark_<mark> {
        # Per-interface marking chains
        meta mark set (meta mark & ~$fw_mask) | $mark
        return
    }

    chain pbr_rules {
        # User-defined policy rules
        ip daddr @pbr_wan1_4_dst_netflix goto pbr_mark_0x010000
        ip saddr @pbr_lan_4_src_iot goto pbr_mark_0x020000
    }
}
```

#### C. Set Structure
```nft
# Domain-based routing sets
set pbr_wan1_4_dst_netflix {
    type ipv4_addr
    flags interval, timeout
    timeout 600s
}

# Source-based routing sets
set pbr_lan_4_src_iot {
    type ipv4_addr
    flags interval
}

# User-defined sets
set pbr_wan1_4_dst_ip_user {
    type ipv4_addr
    flags interval
}
```

#### D. File Generation
```bash
# Main rules file
/usr/share/nftables.d/ruleset-post/30-pbr.nft

# Netifd integration
/usr/share/nftables.d/ruleset-post/20-pbr-netifd.nft

# Applied via:
fw4 reload
```

#### E. DNS Integration
PBR integrates with dnsmasq/unbound for domain-based routing:

```bash
# dnsmasq.conf additions
nftset=/netflix.com/4#inet#fw4#pbr_wan1_4_dst_netflix
nftset=/netflix.com/6#inet#fw4#pbr_wan1_6_dst_netflix
```

### 3.2 Integration Strategy: Shared fw4 Table

**CRITICAL REQUIREMENT**: Both mwan4 and pbr **MUST** use the same `inet fw4` table.

**Why?** nftables does NOT allow `jump` or `goto` between different tables. If mwan4 creates `table inet mwan4` and pbr uses `table inet fw4`, then pbr rules cannot reference mwan4 chains (e.g., `goto mwan4_mark_0x100` would fail).

**Solution**: Both packages add their chains to the shared `table inet fw4` with coordinated priorities.

#### A. Proposed Architecture

```nft
table inet fw4 {
    #
    # ===== mwan4-managed section =====
    #

    # mwan4 sets (interface tracking, connected networks)
    set mwan4_connected_ipv4 { type ipv4_addr; flags interval; }
    set mwan4_custom_ipv4 { type ipv4_addr; flags interval; }

    # mwan4 hook chains (priority: mangle)
    chain mwan4_prerouting {
        type filter hook prerouting priority mangle;

        # Core mwan4 logic: interface tracking, load balancing, failover
        jump mwan4_ifaces_in
        jump mwan4_policies
    }

    chain mwan4_output {
        type route hook output priority mangle;
        jump mwan4_ifaces_in
        jump mwan4_policies
    }

    # mwan4 interface/policy chains
    chain mwan4_ifaces_in { ... }
    chain mwan4_policy_balanced { ... }
    chain mwan4_mark_0x100 { ... }
    chain mwan4_mark_0x200 { ... }

    #
    # ===== pbr-managed section =====
    #

    # pbr sets (domain-based routing, user policies)
    set pbr_wan1_4_dst_netflix { type ipv4_addr; flags interval, timeout; }
    set pbr_wan1_4_src_ip_user { type ipv4_addr; flags interval; }

    # pbr hook chains (priority: mangle + 5, runs AFTER mwan4)
    chain pbr_prerouting {
        type filter hook prerouting priority mangle + 5;

        # Skip if mwan4 already marked
        meta mark & $fw_mask != 0 return

        # Apply user policies
        jump pbr_rules
    }

    chain pbr_output {
        type route hook output priority mangle + 5;
        meta mark & $fw_mask != 0 return
        jump pbr_rules
    }

    # pbr policy chains (reference mwan4 marks)
    chain pbr_rules {
        ip daddr @pbr_wan1_4_dst_netflix goto mwan4_mark_0x100
        ip saddr @pbr_lan_4_src_iot goto mwan4_mark_0x200
    }
}
```

#### B. Chain Priority Ordering

```
Priority Timeline:
─────────────────────────────────────────────────────────
  mangle (0)          mangle + 5           mangle + 10
     │                    │                      │
     ▼                    ▼                      ▼
┌─────────┐         ┌─────────┐           ┌─────────┐
│ mwan4   │   →     │   pbr   │     →     │  other  │
│ chains  │         │ chains  │           │  hooks  │
└─────────┘         └─────────┘           └─────────┘
     │                    │
     ▼                    ▼
Interface          Policy-based
selection,         routing rules
load balancing,    referencing
failover           mwan4 marks
```

### 3.3 Division of Responsibilities

| Component | mwan4 Manages | pbr Manages |
|-----------|---------------|-------------|
| **Interface Tracking** | ✓ (monitors link state) | - |
| **Interface Health** | ✓ (ping tracking) | - |
| **Routing Tables** | ✓ (creates and populates) | Uses existing tables |
| **Table IDs** | ✓ (allocates from iproute2) | Reads from rt_tables |
| **IP Rules** | ✓ (fwmark→table rules) | - |
| **Metrics/Weights** | ✓ (per-interface) | - |
| **Load Balancing** | ✓ (weighted distribution) | - |
| **Failover** | ✓ (automatic rerouting) | - |
| **Fwmark Base** | ✓ (interface base marks) | - |
| **Fwmark Extended** | - | ✓ (user policy marks) |
| **Policy Rules** | - | ✓ (src/dst matching) |
| **Domain Routing** | - | ✓ (DNS integration) |
| **User Sets** | - | ✓ (custom IP sets) |
| **DSCP Tagging** | - | ✓ (QoS integration) |

### 3.4 Coordination Points

#### A. Fwmark Range Allocation

**Critical Requirement**: Both packages must use the same mask and coordinate mark ranges.

```bash
# /etc/config/mwan4
config globals
    option mmx_mask '0xFF00'          # mwan4 uses 0x0100-0xFF00
    option max_interfaces '254'       # Max 254 interfaces

# /etc/config/pbr
config config
    option fw_mask '0xFF00'           # MUST match mwan4 mmx_mask
    option uplink_mark '0x010000'     # pbr user marks start outside mwan4 range
```

**Mark Allocation**:
```
0x00000000 - 0x000000FF : Unused (reserved)
0x00000100 - 0x0000FF00 : mwan4 interface marks (254 interfaces max)
0x00010000 - 0xFFFF0000 : pbr user policy marks
```

#### B. Interface Name Coordination

Both packages must reference the same interface names from `/etc/config/network`.

**Example**:
```bash
# /etc/config/network
config interface 'wan1'
    option device 'eth0'
    option proto 'dhcp'

config interface 'wan2'
    option device 'eth1'
    option proto 'dhcp'

# /etc/config/mwan4
config interface 'wan1'
    option enabled '1'
    option track_ip '8.8.8.8'
    # mwan4 assigns mark 0x100, creates table 1

# /etc/config/pbr
config policy 'netflix_via_wan1'
    option interface 'wan1'  # References mwan4-managed interface
    option dest_addr 'netflix.com'
    # pbr creates: ip daddr @pbr_wan1_4_dst_netflix goto mwan4_mark_0x100
```

#### C. Routing Table Coordination

**mwan4 Actions**:
1. Creates routing table entry in `/etc/iproute2/rt_tables`:
   ```
   1 mwan4_wan1
   2 mwan4_wan2
   ```

2. Populates routing table:
   ```bash
   ip route add default via 192.168.1.1 dev eth0 table 1
   ```

3. Creates fwmark rules:
   ```bash
   ip rule add fwmark 0x100/0xFF00 table 1 priority 2001
   ```

**pbr Actions**:
1. Reads `/etc/iproute2/rt_tables` to discover mwan4 tables
2. References mwan4 marks in policies:
   ```nft
   ip daddr @pbr_wan1_4_dst_netflix goto mwan4_mark_0x100
   ```
3. Does NOT create or modify routing tables

#### D. Atomic File Generation

Both packages generate separate nft files that are loaded together:

```
/usr/share/nftables.d/ruleset-post/
├── 10-mwan4-base.nft          # mwan4: sets and base chains (adds to fw4 table)
├── 11-mwan4-interfaces.nft    # mwan4: interface tracking chains (adds to fw4 table)
├── 12-mwan4-policies.nft      # mwan4: load balancing policies (adds to fw4 table)
├── 13-mwan4-rules.nft         # mwan4: user rules if any (adds to fw4 table)
├── 20-pbr-netifd.nft          # pbr: netifd integration (adds to fw4 table)
└── 30-pbr.nft                 # pbr: policy rules (adds to fw4 table)

# All files add chains/sets to the SAME table: inet fw4
```

**Load Order**:
```bash
# fw4 automatically loads files in order from ruleset-post/
fw4 reload  # Loads all files atomically
```

### 3.5 Example Integration Scenario

#### Scenario: Netflix via WAN1, Gaming via WAN2, Everything Else Load Balanced

**Network Configuration** (`/etc/config/network`):
```bash
config interface 'wan1'
    option device 'eth0'
    option proto 'dhcp'

config interface 'wan2'
    option device 'eth1'
    option proto 'dhcp'
```

**mwan4 Configuration** (`/etc/config/mwan4`):
```bash
config globals
    option mmx_mask '0xFF00'

config interface 'wan1'
    option enabled '1'
    option family 'ipv4'
    list track_ip '8.8.8.8'
    list track_ip '1.1.1.1'
    option track_method 'ping'
    option reliability '1'
    option count '1'
    option timeout '4'
    option interval '10'
    option down '5'
    option up '5'

config interface 'wan2'
    option enabled '1'
    option family 'ipv4'
    list track_ip '8.8.4.4'
    list track_ip '1.0.0.1'
    option track_method 'ping'
    option reliability '1'
    option count '1'
    option timeout '4'
    option interval '10'
    option down '5'
    option up '5'

config member 'wan1_m1_w3'
    option interface 'wan1'
    option metric '1'
    option weight '3'

config member 'wan2_m1_w1'
    option interface 'wan2'
    option metric '1'
    option weight '1'

config policy 'balanced'
    list use_member 'wan1_m1_w3'
    list use_member 'wan2_m1_w1'
    option last_resort 'unreachable'
```

**mwan4 Actions**:
1. Creates routing tables:
   ```
   1 mwan4_wan1
   2 mwan4_wan2
   ```

2. Assigns marks:
   - wan1 → 0x0100
   - wan2 → 0x0200

3. Creates nftables chains:
   ```nft
   chain mwan4_policy_balanced {
       # 3:1 weighted (wan1:wan2)
       meta mark & 0xFF00 == 0 numgen random mod 4 0-2 \
           meta mark set (meta mark & 0xFFFFFF00) | 0x100

       meta mark & 0xFF00 == 0 \
           meta mark set (meta mark & 0xFFFFFF00) | 0x200
   }
   ```

4. Monitors link health (mwan3track daemon)

**pbr Configuration** (`/etc/config/pbr`):
```bash
config config
    option enabled '1'
    option fw_mask '0xFF00'  # Match mwan4
    option verbosity '2'
    option resolver_set 'dnsmasq.nftset'

config policy
    option name 'Netflix via WAN1'
    option interface 'wan1'
    option dest_addr 'netflix.com nflxvideo.net nflximg.net nflxext.com nflxso.net'

config policy
    option name 'Gaming via WAN2'
    option interface 'wan2'
    option dest_port '27015-27030 3074 3478-3479'
    option proto 'udp'

config policy
    option name 'Default Load Balanced'
    option interface 'balanced'
    option dest_addr '0.0.0.0/0'
```

**pbr Actions**:
1. Reads mwan4 marks from coordination:
   - wan1 → 0x0100 (from mwan4)
   - wan2 → 0x0200 (from mwan4)
   - balanced → jump to mwan4_policy_balanced

2. Creates dnsmasq nftset entries:
   ```
   nftset=/netflix.com/4#inet#fw4#pbr_wan1_4_dst_netflix
   nftset=/nflxvideo.net/4#inet#fw4#pbr_wan1_4_dst_netflix
   ```

3. Creates nftables rules:
   ```nft
   chain pbr_rules {
       # Netflix → WAN1
       ip daddr @pbr_wan1_4_dst_netflix goto mwan4_mark_0x100

       # Gaming ports → WAN2
       udp dport 27015-27030 goto mwan4_mark_0x200
       udp dport 3074 goto mwan4_mark_0x200
       udp dport 3478-3479 goto mwan4_mark_0x200

       # Everything else → Load balanced
       ip daddr 0.0.0.0/0 goto mwan4_policy_balanced
   }
   ```

**Result**:
- Netflix traffic is resolved by dnsmasq → IPs added to nft set → marked 0x100 → routed via WAN1
- Gaming UDP traffic → marked 0x200 → routed via WAN2
- All other traffic → processed by mwan4_policy_balanced → 75% WAN1, 25% WAN2
- If WAN1 fails, mwan4 automatically updates policy → all traffic to WAN2

### 3.6 Communication Interface

#### A. Shared State File (JSON)

mwan4 publishes interface state for pbr to consume:

```json
{
  "interfaces": [
    {
      "name": "wan1",
      "enabled": true,
      "status": "online",
      "device": "eth0",
      "gateway_ipv4": "192.168.1.1",
      "gateway_ipv6": "fe80::1",
      "table_id": 1,
      "mark": "0x0100",
      "priority": 2001
    },
    {
      "name": "wan2",
      "enabled": true,
      "status": "offline",
      "device": "eth1",
      "gateway_ipv4": null,
      "table_id": 2,
      "mark": "0x0200",
      "priority": 2002
    }
  ],
  "policies": [
    {
      "name": "balanced",
      "members": ["wan1", "wan2"],
      "last_resort": "unreachable"
    }
  ]
}
```

**Location**: `/var/run/mwan4.json`

**pbr Usage**:
```bash
# pbr reads this to discover mwan4-managed interfaces
json_load "$(cat /var/run/mwan4.json)"
json_select interfaces
json_get_var wan1_mark wan1.mark
# Use wan1_mark to reference in nft rules
```

#### B. ubus Interface

mwan4 could expose state via ubus:

```bash
# Query mwan4 status
ubus call mwan4 status

# Returns:
{
  "wan1": {
    "status": "online",
    "mark": "0x0100",
    "table_id": 1,
    "uptime": 3600
  },
  "wan2": {
    "status": "offline",
    "mark": "0x0200",
    "table_id": 2,
    "downtime": 120
  }
}

# pbr usage:
wan1_mark=$(ubus call mwan4 status | jsonfilter -e '@.wan1.mark')
```

### 3.7 Benefits of Integration

| Benefit | Description |
|---------|-------------|
| **Single Atomic Ruleset** | Both packages generate nft files loaded together via `fw4 reload` |
| **No Conflicts** | Clear separation: mwan4 = interfaces, pbr = policies |
| **Better Performance** | nft's internal VM faster than iptables |
| **Unified Debugging** | `nft list ruleset` shows complete picture |
| **Automatic Failover** | mwan4 handles failover, pbr rules automatically follow |
| **Cleaner Code** | Each package focuses on its core competency |
| **Easier Maintenance** | Separate update cycles for each component |
| **DNS Integration** | pbr's dnsmasq integration works seamlessly with mwan4 marks |

---

## 4. Implementation Roadmap

### Phase 1: mwan4 Core Framework (Weeks 1-4)

**Goal**: Create basic mwan4 package with nftables support

**Tasks**:
1. Package structure setup
   - Create `mwan4` package directory
   - Port configuration schema from mwan3
   - Setup build system (Makefile)

2. Core library migration
   - Port `common.sh` → `common4.sh`
   - Implement nft wrapper functions
   - Port mark calculation logic

3. Basic nftables generation
   - Implement nft file generation (adds to fw4 table)
   - Create base chain structure in fw4
   - Test basic marking functionality

**Deliverables**:
- `mwan4` package builds successfully
- Basic nft chains created in `table inet fw4`
- Fwmark assignment working
- Verified no `table inet mwan4` created

### Phase 2: Interface Management (Weeks 5-8)

**Goal**: Implement interface tracking and routing

**Tasks**:
1. Interface enumeration
   - Port interface discovery from mwan3
   - Implement routing table creation
   - Port device detection logic

2. Routing table management
   - Create per-interface tables
   - Implement gateway detection (IPv4/IPv6)
   - Setup fwmark-based ip rules

3. Interface tracking daemon
   - Port mwan3track to mwan4track
   - Implement health checking
   - Create state change handlers

**Deliverables**:
- Interfaces automatically tracked
- Routing tables populated correctly
- Health monitoring functional

### Phase 3: Load Balancing & Policies (Weeks 9-12)

**Goal**: Implement weighted load balancing with nftables

**Tasks**:
1. Policy chain generation
   - Implement numgen-based weighting
   - Port weight calculation logic
   - Create policy chains dynamically

2. Failover logic
   - Implement policy updates on link state change
   - Test automatic failover scenarios
   - Port offline interface handling

3. Last resort handling
   - Implement blackhole/unreachable/default
   - Test all last resort modes

**Deliverables**:
- Load balancing working with correct weights
- Automatic failover functional
- All policy modes tested

### Phase 4: Advanced Features (Weeks 13-16)

**Goal**: Implement sticky sessions, custom sets, user rules

**Tasks**:
1. Sticky sessions
   - Implement nft tuple sets with timeout
   - Port sticky session logic
   - Test session persistence

2. Custom/connected/dynamic sets
   - Port ipset logic to nft sets
   - Implement set population
   - Test set matching

3. User rules
   - Implement rule configuration
   - Port rule generation logic
   - Test complex rule scenarios

**Deliverables**:
- Sticky sessions working
- Custom sets functional
- User rules processed correctly

### Phase 5: pbr Integration (Weeks 17-20)

**Goal**: Integrate mwan4 with pbr package

**Tasks**:
1. Define coordination interface
   - Implement shared state file (JSON)
   - Document fwmark allocation scheme
   - Create coordination helpers in pbr

2. Update pbr to reference mwan4
   - Modify pbr to read mwan4 state
   - Update pbr nft generation to use mwan4 marks
   - Test policy references

3. Shared fw4 table
   - Coordinate nft file priorities
   - Test atomic reloads
   - Verify chain execution order

**Deliverables**:
- pbr reads mwan4 interface state
- pbr policies reference mwan4 marks correctly
- Combined ruleset loads atomically

### Phase 6: Testing & Validation (Weeks 21-24)

**Goal**: Comprehensive testing of mwan4 + pbr

**Tasks**:
1. Functional testing
   - Test all mwan4 features in isolation
   - Test all pbr features with mwan4
   - Test combined scenarios

2. Performance testing
   - Benchmark nft vs iptables performance
   - Test with high connection counts
   - Verify failover speed

3. Edge case testing
   - Test interface flapping
   - Test concurrent policy changes
   - Test recovery from failures

**Deliverables**:
- Comprehensive test suite
- Performance benchmarks
- Bug fixes and optimizations

### Phase 7: Documentation & Release (Weeks 25-26)

**Goal**: Documentation and release preparation

**Tasks**:
1. Documentation
   - User guide for mwan4
   - Migration guide from mwan3
   - Integration guide for pbr

2. Release preparation
   - Version tagging
   - Changelog creation
   - Package repository setup

**Deliverables**:
- Complete documentation
- Release packages
- Migration tools

---

## 5. Configuration Examples

### 5.1 Basic Two-WAN Setup

**Network Configuration**:
```bash
# /etc/config/network
config interface 'wan1'
    option device 'eth0'
    option proto 'dhcp'

config interface 'wan2'
    option device 'eth1'
    option proto 'dhcp'
```

**mwan4 Configuration**:
```bash
# /etc/config/mwan4
config globals
    option mmx_mask '0xFF00'

config interface 'wan1'
    option enabled '1'
    option family 'ipv4'
    list track_ip '8.8.8.8'
    option track_method 'ping'

config interface 'wan2'
    option enabled '1'
    option family 'ipv4'
    list track_ip '1.1.1.1'
    option track_method 'ping'

config member 'wan1_m1_w1'
    option interface 'wan1'
    option metric '1'
    option weight '1'

config member 'wan2_m1_w1'
    option interface 'wan2'
    option metric '1'
    option weight '1'

config policy 'balanced'
    list use_member 'wan1_m1_w1'
    list use_member 'wan2_m1_w1'
    option last_resort 'default'
```

**pbr Configuration**:
```bash
# /etc/config/pbr
config config
    option enabled '1'
    option fw_mask '0xFF00'

# All traffic uses mwan4 load balancing
config policy
    option name 'Default Balanced'
    option interface 'balanced'
    option dest_addr '0.0.0.0/0'
```

**Generated nftables** (simplified):
```nft
table inet fw4 {
    # mwan4 section
    chain mwan4_prerouting {
        type filter hook prerouting priority mangle;
        meta mark & 0xFF00 == 0 jump mwan4_policy_balanced
    }

    chain mwan4_policy_balanced {
        meta mark & 0xFF00 == 0 numgen random mod 2 0 \
            meta mark set (meta mark & 0xFFFFFF00) | 0x100
        meta mark & 0xFF00 == 0 \
            meta mark set (meta mark & 0xFFFFFF00) | 0x200
    }

    # pbr section
    chain pbr_prerouting {
        type filter hook prerouting priority mangle + 5;
        meta mark & 0xFF00 != 0 return
        ip daddr 0.0.0.0/0 goto mwan4_policy_balanced
    }
}
```

### 5.2 Policy-Based Routing with Failover

**mwan4 Configuration**:
```bash
config globals
    option mmx_mask '0xFF00'

config interface 'wan1'
    option enabled '1'
    option family 'ipv4'
    list track_ip '8.8.8.8'
    list track_ip '8.8.4.4'
    option reliability '1'

config interface 'wan2'
    option enabled '1'
    option family 'ipv4'
    list track_ip '1.1.1.1'
    list track_ip '1.0.0.1'
    option reliability '1'

config member 'wan1_m1_w1'
    option interface 'wan1'
    option metric '1'
    option weight '1'

config member 'wan2_m2_w1'
    option interface 'wan2'
    option metric '2'
    option weight '1'

config policy 'wan1_failover_wan2'
    list use_member 'wan1_m1_w1'
    list use_member 'wan2_m2_w1'
    option last_resort 'unreachable'
```

**pbr Configuration**:
```bash
config config
    option enabled '1'
    option fw_mask '0xFF00'
    option resolver_set 'dnsmasq.nftset'

config policy
    option name 'Streaming via WAN1 with failover'
    option interface 'wan1_failover_wan2'
    option dest_addr 'netflix.com youtube.com hulu.com'

config policy
    option name 'Work VPN always via WAN1'
    option interface 'wan1'
    option dest_addr '10.0.0.0/8'
```

**Behavior**:
- Streaming traffic normally goes to WAN1 (metric 1)
- If WAN1 fails, streaming automatically fails over to WAN2 (metric 2)
- Work VPN always uses WAN1, becomes unreachable if WAN1 down

### 5.3 Complex Weighted Distribution

**mwan4 Configuration**:
```bash
config member 'wan1_m1_w3'
    option interface 'wan1'
    option metric '1'
    option weight '3'

config member 'wan2_m1_w1'
    option interface 'wan2'
    option metric '1'
    option weight '1'

config member 'lte_m2_w1'
    option interface 'lte'
    option metric '2'
    option weight '1'

config policy 'weighted_with_backup'
    list use_member 'wan1_m1_w3'
    list use_member 'wan2_m1_w1'
    list use_member 'lte_m2_w1'
    option last_resort 'default'
```

**Generated nftables**:
```nft
chain mwan4_policy_weighted_with_backup {
    # Metric 1 members (total weight 4)
    # wan1: 3/4 = 75%
    meta mark & 0xFF00 == 0 numgen random mod 4 0-2 \
        meta mark set (meta mark & 0xFFFFFF00) | 0x100 \
        comment "wan1 3 4"

    # wan2: 1/4 = 25%
    meta mark & 0xFF00 == 0 \
        meta mark set (meta mark & 0xFFFFFF00) | 0x200 \
        comment "wan2 1 4"

    # If both metric 1 members fail, use metric 2 (lte)
    # This rule only evaluated if above rules don't match
    # (which happens when both wan1 and wan2 are offline)
}
```

**Behavior**:
- Normal operation: 75% WAN1, 25% WAN2
- WAN1 fails: 100% WAN2
- WAN1 and WAN2 fail: 100% LTE (backup)
- All fail: use default routing

---

## 6. Technical Specifications

### 6.1 Mark Allocation

| Mark Range | Purpose | Example |
|------------|---------|---------|
| `0x00000000 - 0x000000FF` | Reserved/Unused | - |
| `0x00000100` | mwan4 interface 1 | wan1 |
| `0x00000200` | mwan4 interface 2 | wan2 |
| `...` | ... | ... |
| `0x0000FE00` | mwan4 interface 254 | Maximum |
| `0x0000FF00` | mwan4 default/last resort | - |
| `0x00010000 - 0xFFFF0000` | pbr user policies | Netflix, Gaming, etc. |

### 6.2 Routing Table IDs

| Table ID | Purpose | Created By |
|----------|---------|------------|
| `0` | Local | System |
| `1-254` | Per-interface tables | mwan4 |
| `255` | Main | System |
| `256+` | Available for other uses | - |

### 6.3 IP Rule Priorities

| Priority Range | Purpose | Example |
|----------------|---------|---------|
| `1001-1254` | Input interface rules | `iif eth0 lookup 1` |
| `2001-2254` | Fwmark rules | `fwmark 0x100/0xFF00 lookup 1` |
| `2254` | Blackhole rule | `fwmark 0xFE00/0xFF00 blackhole` |
| `2255` | Unreachable rule | `fwmark 0xFF00/0xFF00 unreachable` |
| `3001-3254` | Unreachable fallback | `fwmark 0x100/0xFF00 unreachable` |
| `32765` | Main table (default) | System |
| `32766` | Default table | System |
| `32767` | Local table | System |

### 6.4 nftables Chain Priorities

| Priority | Chain | Package | Purpose |
|----------|-------|---------|---------|
| `mangle` | `mwan4_prerouting` | mwan4 | Interface selection, load balancing |
| `mangle` | `mwan4_output` | mwan4 | Local traffic handling |
| `mangle + 5` | `pbr_prerouting` | pbr | Policy-based routing rules |
| `mangle + 5` | `pbr_output` | pbr | Local policy routing |

### 6.5 File Locations

| Path | Package | Purpose |
|------|---------|---------|
| `/etc/config/mwan4` | mwan4 | Configuration |
| `/etc/config/pbr` | pbr | Configuration |
| `/etc/iproute2/rt_tables` | System | Routing table names |
| `/var/run/mwan4.json` | mwan4 | State export for pbr |
| `/var/run/mwan4.nft` | mwan4 | Temporary nft file (for testing) |
| `/usr/share/nftables.d/ruleset-post/10-mwan4-base.nft` | mwan4 | Sets/chains (added to fw4 table) |
| `/usr/share/nftables.d/ruleset-post/11-mwan4-interfaces.nft` | mwan4 | Interface chains (added to fw4 table) |
| `/usr/share/nftables.d/ruleset-post/12-mwan4-policies.nft` | mwan4 | Policy chains (added to fw4 table) |
| `/usr/share/nftables.d/ruleset-post/20-pbr-netifd.nft` | pbr | Netifd integration (added to fw4 table) |
| `/usr/share/nftables.d/ruleset-post/30-pbr.nft` | pbr | Policy rules (added to fw4 table) |

---

## 7. Key Differences: mwan3 vs mwan4

| Aspect | mwan3 (iptables) | mwan4 (nftables) |
|--------|------------------|------------------|
| **Table Type** | Separate IPv4/IPv6 mangle tables | Single `inet fw4` table (shared) |
| **Update Method** | `iptables-restore` batch | `nft -f` atomic file |
| **Set Management** | `ipset` commands | Native nft sets |
| **Load Balancing** | `-m statistic --probability` | `numgen random mod` |
| **Chain Syntax** | `-A`, `-N`, `-j` flags | Declarative syntax |
| **Performance** | Userspace iptables | Kernel nft VM |
| **Debugging** | `iptables -L -v -n -t mangle` | `nft list ruleset` |
| **Integration** | Standalone | Shares `table inet fw4` with pbr/firewall |

---

## 8. Migration Guide: mwan3 → mwan4

### 8.1 Pre-Migration Checklist

- [ ] Backup current `/etc/config/mwan3`
- [ ] Document current routing policies
- [ ] Test failover scenarios with mwan3
- [ ] Verify nftables support: `nft list ruleset`
- [ ] Verify kernel version: `uname -r` (need 4.14+)

### 8.2 Migration Steps

1. **Install mwan4** (keep mwan3 for now):
   ```bash
   opkg update
   opkg install mwan4
   ```

2. **Convert configuration**:
   ```bash
   # mwan4 will read /etc/config/mwan3 and create /etc/config/mwan4
   mwan4-convert /etc/config/mwan3
   ```

3. **Test mwan4 in parallel**:
   ```bash
   # Start mwan4 without stopping mwan3
   /etc/init.d/mwan4 start

   # Verify nft rules (check fw4 table for mwan4 chains)
   nft list table inet fw4 | grep mwan4

   # Check routing tables
   ip rule list
   ip route show table all
   ```

4. **Switch over**:
   ```bash
   # Stop mwan3
   /etc/init.d/mwan3 stop
   /etc/init.d/mwan3 disable

   # Enable mwan4
   /etc/init.d/mwan4 enable
   /etc/init.d/mwan4 restart
   ```

5. **Update pbr** (if using):
   ```bash
   # Update pbr to use mwan4 coordination
   opkg upgrade pbr
   /etc/init.d/pbr restart
   ```

6. **Verify**:
   ```bash
   # Check interface status
   mwan4 status

   # Check policies
   mwan4 policies

   # Check nftables (mwan4 chains are in fw4 table)
   nft list table inet fw4 | grep mwan4
   ```

### 8.3 Rollback Plan

If issues occur:
```bash
# Stop mwan4
/etc/init.d/mwan4 stop
/etc/init.d/mwan4 disable

# Restart mwan3
/etc/init.d/mwan3 enable
/etc/init.d/mwan3 start

# Verify
mwan3 status
```

---

## 9. Troubleshooting

### 9.1 Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No load balancing | pbr overriding mwan4 | Check pbr priority, ensure `meta mark & $fw_mask != 0 return` |
| Interface always offline | Track IPs unreachable | Check `mwan4track` logs, verify track_ip addresses |
| Failover not working | Metric not configured | Ensure different metrics for primary/backup |
| nft syntax error | Invalid rule generation | Check `/var/run/mwan4.nft` for errors |
| Routing loops | Conflicting rules | Use `ip rule list` and `nft list ruleset` |

### 9.2 Debugging Commands

```bash
# Check mwan4 status
mwan4 status
mwan4 interfaces
mwan4 policies

# Check nftables (all in fw4 table)
nft list ruleset
nft list table inet fw4
nft list chain inet fw4 mwan4_prerouting
nft list chain inet fw4 pbr_prerouting

# Check routing
ip rule list
ip route show table all
cat /etc/iproute2/rt_tables

# Check tracking
ps aux | grep mwan4track
cat /var/run/mwan4.json

# Check logs
logread | grep mwan4
logread | grep pbr
```

---

## 10. Conclusion

The transition from mwan3 (iptables) to mwan4 (nftables) represents a significant architectural improvement that enables:

1. **Better Performance**: nftables' kernel VM is faster and more efficient
2. **Cleaner Integration**: Shared fw4 table with pbr eliminates conflicts
3. **Simplified Management**: Atomic updates via nft files
4. **Future-Proof**: nftables is the future of Linux packet filtering

The integration with pbr creates a powerful combination where:
- **mwan4** handles the complex multi-WAN infrastructure (tracking, metrics, failover, load balancing)
- **pbr** focuses on policy creation and DNS integration
- Both work together seamlessly via coordinated fwmarks and the **shared `table inet fw4`**

This architecture provides users with enterprise-grade multi-WAN capabilities combined with flexible policy-based routing, all managed through simple UCI configuration files.

**Key Technical Point**: The shared table approach is not optional—it's a fundamental requirement of nftables architecture. Since pbr already uses `table inet fw4` and cannot jump to chains in other tables, mwan4 must also add its chains to the same `table inet fw4` to enable cross-referencing between the packages.
