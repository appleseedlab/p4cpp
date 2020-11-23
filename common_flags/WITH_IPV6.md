[Including the WITH_IPV6 flag](https://github.com/OpenNetworkingFoundation/onos-sdran/blob/master/pipelines/fabric/src/main/resources/Makefile#L19) through command line:
```
@./bmv2-compile.sh "fabric-full" " -DWITH_MULTICAST -DWITH_IPV6 ...
```
<br/>
<br/>

[Header file](https://github.com/OpenNetworkingFoundation/onos-sdran/blob/8e99a8827af956946fa395546328ec021e63f9d3/pipelines/fabric/src/main/resources/include/header.p4#L223)
```
struct parsed_headers_t {
    ethernet_t ethernet;
    vlan_tag_t vlan_tag;
    ...
    ipv4_t ipv4;
#ifdef WITH_IPV6
    ipv6_t ipv6;
#endif // WITH_IPV6
    ...
}
```
Including new structFields specific for IPV6 iff the WITH_IPV6 flag is defined
<br/>
<br/>

[Parser file](https://github.com/OpenNetworkingFoundation/onos-sdran/blob/8e99a8827af956946fa395546328ec021e63f9d3/pipelines/fabric/src/main/resources/include/parser.p4#L49)
```
parser FabricParser (packet_in packet,
                     out parsed_headers_t hdr,
                     inout fabric_metadata_t fabric_metadata,
                     inout standard_metadata_t standard_metadata) {

    ...

    state parse_ethernet {
        packet.extract(hdr.ethernet);
        fabric_metadata.last_eth_type = hdr.ethernet.eth_type;
        fabric_metadata.vlan_id = DEFAULT_VLAN_ID;
        transition select(hdr.ethernet.eth_type){
            ETHERTYPE_VLAN: parse_vlan_tag;
            ETHERTYPE_MPLS: parse_mpls;
            ETHERTYPE_IPV4: pre_parse_ipv4;
#ifdef WITH_IPV6
            ETHERTYPE_IPV6: pre_parse_ipv6;
#endif // WITH_IPV6
            default: accept;
        }
    }

    state parse_vlan_tag {
        packet.extract(hdr.vlan_tag);
        transition select(hdr.vlan_tag.eth_type){
            ETHERTYPE_IPV4: pre_parse_ipv4;
#ifdef WITH_IPV6
            ETHERTYPE_IPV6: pre_parse_ipv6;
#endif // WITH_IPV6
            ETHERTYPE_MPLS: parse_mpls;
#if defined(WITH_XCONNECT) || defined(WITH_BNG) || defined(WITH_DOUBLE_VLAN_TERMINATION)
            ETHERTYPE_VLAN: parse_inner_vlan_tag;
            ETHERTYPE_QINQ: parse_inner_vlan_tag;
            ETHERTYPE_QINQ_NON_STD: parse_inner_vlan_tag;
#endif // WITH_XCONNECT
            default: accept;
        }
    }

#if defined(WITH_XCONNECT) || defined(WITH_BNG) || defined(WITH_DOUBLE_VLAN_TERMINATION)
    state parse_inner_vlan_tag {
        packet.extract(hdr.inner_vlan_tag);
        transition select(hdr.inner_vlan_tag.eth_type){
            ETHERTYPE_IPV4: pre_parse_ipv4;
#ifdef WITH_IPV6
            ETHERTYPE_IPV6: pre_parse_ipv6;
#endif // WITH_IPV6
            ETHERTYPE_MPLS: parse_mpls;
#ifdef WITH_BNG
            ETHERTYPE_PPPOED: parse_pppoe;
            ETHERTYPE_PPPOES: parse_pppoe;
#endif // WITH_BNG
            default: accept;
        }
    }
#endif // WITH_XCONNECT || WITH_BNG || WITH_DOUBLE_VLAN_TERMINATION

#ifdef WITH_BNG
    state parse_pppoe {
        packet.extract(hdr.pppoe);
        transition select(hdr.pppoe.protocol) {
            PPPOE_PROTOCOL_MPLS: parse_mpls;
            PPPOE_PROTOCOL_IP4: pre_parse_ipv4;
#ifdef WITH_IPV6
            PPPOE_PROTOCOL_IP6: pre_parse_ipv6;
#endif // WITH_IPV6
            default: accept;
        }
    }
#endif // WITH_BNG

    state parse_mpls {
        packet.extract(hdr.mpls);
        fabric_metadata.is_mpls = _TRUE;
        fabric_metadata.mpls_label = hdr.mpls.label;
        fabric_metadata.mpls_ttl = hdr.mpls.ttl;
        // There is only one MPLS label for this fabric.
        // Assume header after MPLS header is IPv4/IPv6
        // Lookup first 4 bits for version
        transition select(packet.lookahead<bit<IP_VER_LENGTH>>()) {
            // The packet should be either IPv4 or IPv6.
            // If we have MPLS, go directly to parsing state without
            // moving to pre_ states, the packet is considered MPLS
            IP_VERSION_4: parse_ipv4;
#ifdef WITH_IPV6
            IP_VERSION_6: parse_ipv6;
#endif // WITH_IPV6
            default: parse_ethernet;
        }
    }
    ...
```
In all the states above, we add support to transition to either `parse_ipv6` or `pre_parse_ipv6` states if the WITH_IPV6 flag is defined.
<br/>

```
#ifdef WITH_IPV6
    // Intermediate state to set is_ipv6
    state pre_parse_ipv6 {
        fabric_metadata.is_ipv6 = _TRUE;
        transition parse_ipv6;
    }
    state parse_ipv6 {
        packet.extract(hdr.ipv6);
        fabric_metadata.ip_proto = hdr.ipv6.next_hdr;
        fabric_metadata.ip_eth_type = ETHERTYPE_IPV6;
        transition select(hdr.ipv6.next_hdr) {
            PROTO_TCP: parse_tcp;
            PROTO_UDP: parse_udp;
            PROTO_ICMPV6: parse_icmp;
            default: accept;
        }
    }
#endif // WITH_IPV6
```
Then we add the `pre_parse_ipv6` and `parse_ipv6` states in the Parser file to parse IPV6 headers.
<br />

```
control FabricDeparser(packet_out packet,in parsed_headers_t hdr) {

    apply {
        packet.emit(hdr.packet_in);
        ...
        packet.emit(hdr.ipv4);
#ifdef WITH_IPV6
        packet.emit(hdr.ipv6);
#endif // WITH_IPV6
        packet.emit(hdr.tcp);
        packet.emit(hdr.udp);
        packet.emit(hdr.icmp);
        ...
    }
}
```
Finally we add IPV6 emit statement to the deparser to include it back into the packet.
<br />
<br />

[Forwarding](https://github.com/OpenNetworkingFoundation/onos-sdran/blob/8e99a8827af956946fa395546328ec021e63f9d3/pipelines/fabric/src/main/resources/include/control/forwarding.p4#L115)
```
control Forwarding (inout parsed_headers_t hdr,
                    inout fabric_metadata_t fabric_metadata,
                    inout standard_metadata_t standard_metadata) {

    ...

#ifdef WITH_IPV6
    /*
     * IPv6 Routing Table.
     */
    direct_counter(CounterType.packets_and_bytes) routing_v6_counter;

    action set_next_id_routing_v6(next_id_t next_id) {
        set_next_id(next_id);
        routing_v6_counter.count();
    }

    table routing_v6 {
        key = {
            hdr.ipv6.dst_addr: lpm @name("ipv6_dst");
        }
        actions = {
            set_next_id_routing_v6;
            @defaultonly nop;
        }
        const default_action = nop();
        counters = routing_v6_counter;
        size = ROUTING_V6_TABLE_SIZE;
    }
#endif // WITH_IPV6

    apply {
        if (fabric_metadata.fwd_type == FWD_BRIDGING) bridging.apply();
        else if (fabric_metadata.fwd_type == FWD_MPLS) mpls.apply();
        else if (fabric_metadata.fwd_type == FWD_IPV4_UNICAST) routing_v4.apply();
#ifdef WITH_IPV6
        else if (fabric_metadata.fwd_type == FWD_IPV6_UNICAST) routing_v6.apply();
#endif // WITH_IPV6
    }
}
```
Here they add IPV6 routing table if the program is built to support IPV6, then they add support ot reach that forwarding table in the apply block
<br/>
<br/>

[Next](https://github.com/OpenNetworkingFoundation/onos-sdran/blob/8e99a8827af956946fa395546328ec021e63f9d3/pipelines/fabric/src/main/resources/include/control/next.p4#L373)
```
control EgressNextControl (inout parsed_headers_t hdr,
                           inout fabric_metadata_t fabric_metadata,
                           inout standard_metadata_t standard_metadata) {
    ...

    apply {
        ...

        // TTL decrement and check.
        if (hdr.mpls.isValid()) {
            hdr.mpls.ttl = hdr.mpls.ttl - 1;
            if (hdr.mpls.ttl == 0) mark_to_drop(standard_metadata);
        } else {
            if(hdr.ipv4.isValid()) {
                hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
                if (hdr.ipv4.ttl == 0) mark_to_drop(standard_metadata);
            }
#ifdef WITH_IPV6
            else if (hdr.ipv6.isValid()) {
                hdr.ipv6.hop_limit = hdr.ipv6.hop_limit - 1;
                if (hdr.ipv6.hop_limit == 0) mark_to_drop(standard_metadata);
            }
#endif // WITH_IPV6
        }
    }
}
```
Here they add the check to see if the current pack is IPV6 iff the flag is defined.
<br/>
<br/>

[BNG](https://github.com/OpenNetworkingFoundation/onos-sdran/blob/8e99a8827af956946fa395546328ec021e63f9d3/pipelines/fabric/src/main/resources/include/bng.p4#L99)
```
control bng_ingress_upstream(
        inout parsed_headers_t hdr,
        inout fabric_metadata_t fmeta,
        inout standard_metadata_t smeta) {
    ...

    #ifdef WITH_IPV6
    action term_enabled_v6() {
        term_enabled(ETHERTYPE_IPV6);
    }

    // TODO: add match on hdr.ethernet.src_addr for antispoofing
    // Match on unmodified metadata field, taking into account that MAC src address
    // is modified by the Next control block when doing routing functionality.
    table t_pppoe_term_v6 {
        key = {
            fmeta.bng.line_id         : exact @name("line_id");
            hdr.ipv6.src_addr[127:64] : exact @name("ipv6_src_net_id");
            hdr.pppoe.session_id      : exact @name("pppoe_session_id");
        }
        actions = {
            term_enabled_v6;
            @defaultonly term_disabled;
        }
        size = BNG_MAX_SUBSC_NET;
        const default_action = term_disabled;
    }
#endif // WITH_IPV6

    apply {
        if(t_pppoe_cp.apply().hit) {
            return;
        }
        if (hdr.ipv4.isValid()) {
            switch(t_pppoe_term_v4.apply().action_run) {
                term_disabled: {
                    c_dropped.count(fmeta.bng.line_id);
                }
            }
        }
#ifdef WITH_IPV6
        else if (hdr.ipv6.isValid()) {
            switch(t_pppoe_term_v6.apply().action_run) {
               term_disabled: {
                   c_dropped.count(fmeta.bng.line_id);
               }
           }
        }
#endif // WITH_IPV6
    }
}
```
Here we can note that if IPV6 is supported, we add a new action block and a table that deals with IPV6 (setting VLAN tag's eth_type and fmeta's last_eth_type to ipv6, setting pppoe to invalid, and setting the c_terminated count).
Then right after we add support to invoke those IPV6 actions in the apply block.
<br/>

```
control bng_ingress_downstream(
        inout parsed_headers_t hdr,
        inout fabric_metadata_t fmeta,
        inout standard_metadata_t smeta) {

   ...

#ifdef WITH_IPV6
    table t_qos_v6 {
        key = {
            fmeta.bng.line_id      : ternary @name("line_id");
            hdr.ipv6.src_addr      : lpm     @name("ipv6_src");
            hdr.ipv6.traffic_class : ternary @name("ipv6_traffic_class");
        }
        actions = {
            qos_prio;
            qos_besteff;
        }
        size = 256;
        const default_action = qos_besteff;
    }
#endif // WITH_IPV6

    apply {
        // We are not sure the pkt is a BNG downstream one, first we need to
        // verify the line_id matches the one of a subscriber...

        // IPv4
        if (t_line_session_map.apply().hit) {
            // Apply QoS only to subscriber traffic. This makes sense only
            // if the downstream ports are used to receive IP traffic NOT
            // destined to subscribers, e.g. to services in the compute
            // nodes.
            if (hdr.ipv4.isValid()) {
                switch (t_qos_v4.apply().action_run) {
                    qos_prio: {
                        m_prio.execute_meter(fmeta.bng.line_id, fmeta.bng.ds_meter_result);
                    }
                    qos_besteff: {
                        m_besteff.execute_meter(fmeta.bng.line_id, fmeta.bng.ds_meter_result);
                    }
                }
            }
#ifdef WITH_IPV6
            // IPv6
            else if (hdr.ipv6.isValid()) {
                switch (t_qos_v6.apply().action_run) {
                    qos_prio: {
                        m_prio.execute_meter(fmeta.bng.line_id, fmeta.bng.ds_meter_result);
                    }
                    qos_besteff: {
                        m_besteff.execute_meter(fmeta.bng.line_id, fmeta.bng.ds_meter_result);
                    }
                }
            }
#endif // WITH_IPV6
        }
    }
}
```
Similarly, in the ingress downstrea control block, we add support for IPV6 packets by adding a new table and calling to that table in the apply block.
<br/>

```
control bng_egress_downstream(
        inout parsed_headers_t hdr,
        inout fabric_metadata_t fmeta,
        inout standard_metadata_t smeta) {

    ...

#ifdef WITH_IPV6
    action encap_v6() {
        encap();
        hdr.pppoe.length = hdr.ipv6.payload_len + 16w42;
        hdr.pppoe.protocol = PPPOE_PROTOCOL_IP6;
    }
#endif // WITH_IPV6

    apply {
        if (hdr.ipv4.isValid()) {
            encap_v4();
        }
#ifdef WITH_IPV6
        // IPv6
        else if (hdr.ipv6.isValid()) {
            encap_v6();
        }
#endif // WITH_IPV6
    }
}
```
We add support for IPV6 in the egress downstream
<br/>