# IGMP/MLD upstream VLAN

## Background

`kmod-iptv` tags upstream frames according to the frame type.  Unicast uses
the service VID and IPv4/IPv6 multicast uses the multicast VID.  IGMP/MLD
reports also have a multicast destination MAC, so the original implementation
tagged them with the multicast VID as well.

Some operators keep the multicast VLAN downstream-only.  Their OLT expects
IGMP/MLD reports on the service VLAN; a report sent on the multicast VLAN has
no upstream mapping and is dropped by `airoha_pon_select_flow()`
(`tx_no_mapping`).  The visible symptom is:

- the STB obtains an address and authentication/VOD works;
- live TV stays black;
- `tcpdump -ni pon0 -e -n 'vlan and igmp'` only shows the report on the
  multicast VLAN.

## Configuration

`/etc/config/iptv`:

```uci
config iptv 'config'
	option enabled '1'
	option uplink 'pon0'
	option name 'itv'
	option service_vid '43'
	option mcast_vid '3990'
	option igmp_vid 'mcast'
	option macaddr ''
```

- `mcast` (default): IGMP/MLD uses `mcast_vid`, preserving the original
  behavior;
- `service`: IGMP/MLD uses `service_vid`.

The LuCI page exposes the same choice as "Multicast control VLAN".

## Control frame matching

Only these packets use the selected control VLAN:

- IPv4: `IPPROTO_IGMP`;
- IPv6: ICMPv6 type 130 (MLD query), 131 (MLDv1 report),
  132 (MLDv1 done), 143 (MLDv2 report).

All other multicast remains on `mcast_vid`, including IPv6 NDP (types
133..137), IPv6 multicast data, mDNS and SSDP.

## Diagnostics

```sh
# VLAN of the actual IGMP report (pon0 carries the VLAN tag)
tcpdump -ni pon0 -e -n -vv 'vlan and igmp'

# Untagged logical interface
tcpdump -ni itv -e -n -vv 'igmp'

# PON datapath counters
cat /sys/kernel/debug/airoha-xpon-pon0/datapath

# PON/OMCI status (reference only)
ponctl --device pon0 status --json
```

Both downstream queries and upstream reports carry a VLAN tag on `pon0`.
The query PCP is set by the OLT (often 0); the report PCP follows the `iptv`
`priority` option.  The PON path has no hardware VLAN TX, so the tag is
inserted into the frame data; capture filters must include `vlan`.

`tx_no_mapping` growing while zapping channels usually means the selected
control VLAN has no upstream flow.  Selecting `service` should stop the
counter from growing for the IPTV reports.

## Limitations

- Untagged IGMP/MLD upstream (OMCI tag control 0) is not implemented;
- the ONU does not apply the OMCI Class 309 IGMP upstream TCI automatically;
  select the VLAN according to the operator;
- RX demultiplexing still covers only the service and multicast VIDs; the
  control VLAN is not a third RX VLAN.
