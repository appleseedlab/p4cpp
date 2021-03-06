Source: https://github.com/epicLevi/onos-ai-docker/blob/e2aefa79fbe7464671bbfd5e4e3b2ffa609bc099/pipelines/fabric/impl/src/main/resources/include/parser.p4
```
state parse_vlan_tag {
    packet.extract(hdr.vlan_tag);
#ifdef WITH_BNG
    fabric_metadata.bng.s_tag = hdr.vlan_tag.vlan_id;
#endif // WITH_BNG
    transition select(packet.lookahead<bit<16>>()){
#if defined(WITH_XCONNECT) || defined(WITH_DOUBLE_VLAN_TERMINATION)
        ETHERTYPE_VLAN: parse_inner_vlan_tag;
#endif // WITH_XCONNECT || WITH_DOUBLE_VLAN_TERMINATION
        default: parse_eth_type;
    }
}
```
Using `ifdef` to insert new statements within a state. `if` is also used to evaluate a condition and insert new transition states
<br>
<br>

```
#ifdef WITH_IPV6
    state parse_ipv6 {
        ...
        }
    }
#endif // WITH_IPV6
```
`ifdef` being used to insert a new state in the parser depending on the IPV6 flag
<br>
<br>

```
#ifdef WITH_INT_SINK
        // Parse INT metadata stack, but not tail
        packet.extract(hdr.int_data, (bit<32>) (hdr.intl4_shim.len_words - INT_HEADER_LEN_WORDS) << 5);
        transition parse_intl4_tail;
#else // not interested in INT data
        transition accept;
#endif // WITH_INT_SINK
```
Using `ifdef` and `else` to determine the last transition state
<br>
<br>

```
control FabricDeparser(packet_out packet,in parsed_headers_t hdr) {

    apply {
        packet.emit(hdr.packet_in);
#ifdef WITH_INT_SINK
        packet.emit(hdr.report_ethernet);
        packet.emit(hdr.report_eth_type);
        packet.emit(hdr.report_ipv4);
        packet.emit(hdr.report_udp);
        packet.emit(hdr.report_fixed_header);
#endif // WITH_INT_SINK
        packet.emit(hdr.ethernet);
        packet.emit(hdr.vlan_tag);
#if defined(WITH_XCONNECT) || defined(WITH_DOUBLE_VLAN_TERMINATION)
        packet.emit(hdr.inner_vlan_tag);
#endif // WITH_XCONNECT || WITH_DOUBLE_VLAN_TERMINATION
        ...
    }
}
```
`ifdef` and `if` being used to insert specific emit statements in the deparser

***

Source: https://github.com/epicLevi/onos-ai-docker/blob/e2aefa79fbe7464671bbfd5e4e3b2ffa609bc099/pipelines/fabric/impl/src/main/resources/fabric.p4

```
#ifdef WITH_PORT_COUNTER
#include "include/control/port_counter.p4"
#endif // WITH_PORT_COUNTER
```
`ifdef` being used to include header files based on the flags
<br>
<br>

```
    apply {
        _PRE_INGRESS
#ifdef WITH_SPGW
        spgw_normalizer.apply(hdr.gtpu.isValid(), hdr.gtpu_ipv4, hdr.gtpu_udp,
                              hdr.ipv4, hdr.udp, hdr.inner_ipv4, hdr.inner_udp);
#endif // WITH_SPGW
        ...
    }
```
Inserting statements inside apply block based on flags (using `ifdef`)

***

Source: https://github.com/mc36/freeRouter/blob/5c81951a58b7bc3b79dcae7fd0ab9aa707330ad5/misc/p4lang/include/hdr_eg_headers.p4
```
#ifndef _EGRESS_HEADERS_P4_
#define _EGRESS_HEADERS_P4_

/*------------------ E G R E S S  H E A D E R S ----------------------------- */
struct egress_headers_t {
}

#endif // _EGRESS_HEADERS_P4_
```
Multiple files use the `ifndef` directive to make sure they aren't declaring something multiple times

***

Source: https://github.com/opennetworkinglab/onos/blob/f4add194ade23c003230e209e8fcc1b7dfc86af4/pipelines/fabric/impl/src/main/resources/include/control/forwarding.p4
```
    action nop_routing_v4() {
        // no-op
#ifdef WTIH_DEBUG
        routing_v4_counter.count();
#endif // WITH_DEBUG
    }

    #ifdef _ROUTING_V4_TABLE_ANNOT
    _ROUTING_V4_TABLE_ANNOT
    #endif
    table routing_v4 {
        ...
        }
        ...
#ifdef WTIH_DEBUG
        counters = routing_v4_counter;
#endif // WITH_DEBUG
        ...
    }
```
`ifdef` being used as a debug flag to track the routing counts
<br>
<br>

```
#ifdef WITH_IPV6
    /*
     * IPv6 Routing Table.
     */
    direct_counter(CounterType.packets_and_bytes) routing_v6_counter;

    action set_next_id_routing_v6(next_id_t next_id) {
        ...
    }

    table routing_v6 {
        ...
    }
#endif // WITH_IPV6

    apply {
        if ...;
        else if...
#ifdef WITH_IPV6
        else if (fabric_metadata.fwd_type == FWD_IPV6_UNICAST) routing_v6.apply();
#endif // WITH_IPV6
    }
```
`ifdef` being used to add IPV6 routing table and call that in the apply statement depending on the IPV6 flag