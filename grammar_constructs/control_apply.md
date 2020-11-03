###### 1.1
Source: https://github.com/breezestars/onos-barefoot/blob/440d67b3a97363ed9b15894162c4c8bac7ebff05/pipelines/fabric/src/main/resources/include/parser.p4
```
control FabricDeparser(packet_out packet,in parsed_headers_t hdr) {

    apply {
        packet.emit(hdr.packet_in);
        ...
#ifdef WITH_IPV6
        packet.emit(hdr.ipv6);
#endif // WITH_IPV6
        packet.emit(hdr.tcp);
        packet.emit(hdr.udp);
        packet.emit(hdr.icmp);
#ifdef WITH_INT
        packet.emit(hdr.intl4_shim);
        packet.emit(hdr.int_header);
#ifdef WITH_INT_TRANSIT
        packet.emit(hdr.int_switch_id);
        packet.emit(hdr.int_port_ids);
        packet.emit(hdr.int_hop_latency);
        packet.emit(hdr.int_q_occupancy);
        packet.emit(hdr.int_ingress_tstamp);
        packet.emit(hdr.int_egress_tstamp);
        packet.emit(hdr.int_q_congestion);
        packet.emit(hdr.int_egress_tx_util);
#endif // WITH_INT_TRANSIT
#ifdef WITH_INT_SINK
        packet.emit(hdr.int_data);
#endif // WITH_INT_SINK
        packet.emit(hdr.intl4_tail);
#endif // WITH_INT
    }
}
```
In this deparser control block, we add new emit statements to emit specific headers based on whether the feature is supported or not (eg: if IPV6 is supported, we emit the IPV6 headers). In the above code they have nested the `ifdef` directive so that we check WITH_INT_TRANSIT and WITH_INT_SINK are defined iff WITH_INT is supported/defined; if WITH_INT is not supported, then we don't keep the statements under WITH_INT_TRANSIT or WITH_INT_SINK
###### 1.2
Source: https://github.com/breezestars/onos-barefoot/blob/440d67b3a97363ed9b15894162c4c8bac7ebff05/pipelines/fabric/src/main/resources/fabric.p4
```
control FabricIngress (inout parsed_headers_t hdr,
                       inout fabric_metadata_t fabric_metadata,
                       inout standard_metadata_t standard_metadata) {

    PacketIoIngress() pkt_io_ingress;
    Filtering() filtering;
    Forwarding() forwarding;
    Acl() acl;
    Next() next;
#ifdef WITH_PORT_COUNTER
    PortCountersControl() port_counters_control;
#endif // WITH_PORT_COUNTER

    apply {
        _PRE_INGRESS
#ifdef WITH_SPGW
        spgw_normalizer.apply(hdr.gtpu.isValid(), hdr.gtpu_ipv4, hdr.gtpu_udp,
                              hdr.ipv4, hdr.udp, hdr.inner_ipv4, hdr.inner_udp);
#endif // WITH_SPGW
        pkt_io_ingress.apply(hdr, fabric_metadata, standard_metadata);
        filtering.apply(hdr, fabric_metadata, standard_metadata);
#ifdef WITH_SPGW
        spgw_ingress.apply(hdr.gtpu_ipv4, hdr.gtpu_udp, hdr.gtpu,
                           hdr.ipv4, hdr.udp, fabric_metadata, standard_metadata);
#endif // WITH_SPGW
        ...
    }
}

control FabricEgress (inout parsed_headers_t hdr,
                      inout fabric_metadata_t fabric_metadata,
                      inout standard_metadata_t standard_metadata) {

    PacketIoEgress() pkt_io_egress;
    EgressNextControl() egress_next;

    apply {
        _PRE_EGRESS
        pkt_io_egress.apply(hdr, fabric_metadata, standard_metadata);
        egress_next.apply(hdr, fabric_metadata, standard_metadata);
#ifdef WITH_SPGW
        spgw_egress.apply(hdr.ipv4, hdr.gtpu_ipv4, hdr.gtpu_udp, hdr.gtpu,
                          fabric_metadata, standard_metadata);
#endif // WITH_SPGW
        ...
    }
}
```
Here, in the Ingress control block, we add a new variable based on the flag. Plus inside the apply block, we invoke another control block's apply statement if that feature is supported

###### 2

Source: https://github.com/breezestars/onos-barefoot/blob/440d67b3a97363ed9b15894162c4c8bac7ebff05/pipelines/fabric/src/main/resources/include/control/next.p4
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
Towards the end this control block's apply method, they make sure either MPLS or IPV4 is valid and if it is it decrements the TTL. There, they conditionally add a new block inside the else block that if the program supports IPV6 protocol, so if IPV4 is not present and IPV6 is present (and supported by the program), we decrement the IPV6 header's hop_limit

###### 3

Source: https://github.com/breezestars/onos-barefoot/blob/440d67b3a97363ed9b15894162c4c8bac7ebff05/pipelines/fabric/src/main/resources/include/bng.p4
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
        // If table miss, line_id will be 0 (default metadata value).
        t_line_map.apply();

        if (t_pppoe_cp.apply().hit) {
            return;
        }

        if (hdr.ipv4.isValid()) {
            t_pppoe_term_v4.apply();
            if (drop == _TRUE) {
                c_dropped.count(fmeta.bng.line_id);
            }
        }
#ifdef WITH_IPV6
        else if (hdr.ipv6.isValid()) {
            t_pppoe_term_v6.apply();
            if (drop == _TRUE) {
                c_dropped.count(fmeta.bng.line_id);
             }
        }
#endif // WITH_IPV6
    }
}
```
Here we can see that in `bng_ingress_upstream` control block we add a table and an action specific to IPV6 protocol if it is supported. And inside the apply block, we include a conditional statement to handle IPV6 packets that calls the table iff IPV6 is supported

###### 4

Source: https://github.com/opennetworkinglab/onos/blob/f4add194ade23c003230e209e8fcc1b7dfc86af4/pipelines/fabric/impl/src/main/resources/include/control/forwarding.p4 
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
Similar to the last example, here we add a new action and a table pretaining to IPV6 protocol if the WITH_IPV6 flag is defined. Then in the apply block we call the IPV6 table based on the WITH_IPV6 flag

###### 5

Source:https://github.com/imec-idlab/onos-whisper/blob/9150fcb524ac756492b109c7f4ef5713069744f3/pipelines/fabric/src/main/resources/include/int/int_transit.p4
```
control process_int_transit (
    inout parsed_headers_t hdr,
    inout fabric_metadata_t fmeta,
    inout standard_metadata_t smeta) {

    action init_metadata(bit<32> switch_id) {
        fmeta.int_meta.transit = _TRUE;
#ifdef _INT_INIT_METADATA
        // Allow other targets to initialize INT metadata in their own way.
        _INT_INIT_METADATA
#else
        fmeta.int_meta.switch_id = switch_id;
#endif // _INT_INIT_METADATA
    }

#ifdef _INT_METADATA_ACTIONS
    _INT_METADATA_ACTIONS
#else
    // Switch ID.
    @hidden
    action int_set_header_0() {
        hdr.int_switch_id.setValid();
        hdr.int_switch_id.switch_id = fmeta.int_meta.switch_id;
    }
    ...
#endif // _INT_METADATA_ACTIONS

    // Actions to keep track of the new metadata added.
    @hidden
    action add_1() {
        fmeta.int_meta.new_words = fmeta.int_meta.new_words + 1;
        fmeta.int_meta.new_bytes = fmeta.int_meta.new_bytes + 4;
    }

    @hidden
    action add_2() {
        fmeta.int_meta.new_words = fmeta.int_meta.new_words + 2;
        fmeta.int_meta.new_bytes = fmeta.int_meta.new_bytes + 8;
    }
    ...
}
```
Here we can observe that inside an action block, the program gives the user the option to initialize their own INT metadata and add that in the action block. Similarly they also allow the user to define their own set of meta actions to set headers. If none of those are defined by the user, the program uses the default values.

###### 6

Source: https://github.com/xinshengzzy/AHashFlow/blob/59daaff50b92afc05675bdc1d05b83b914e61ca8/src/switch-8.2.0/p4src/multicast.p4

```
control process_outer_multicast_rpf {
#if !defined(OUTER_PIM_BIDIR_OPTIMIZATION)
    /* outer mutlicast RPF check - sparse and bidir */
    if (multicast_metadata.outer_mcast_route_hit == TRUE) {
        apply(outer_multicast_rpf);
    }
#endif /* !OUTER_PIM_BIDIR_OPTIMIZATION */
}
```
Sometimes they have the skeleton of a control block but it only has contents in it if the flag is defined

###### 7
Source: https://github.com/wyan-all/onos-satellite/blob/master/pipelines/fabric/src/main/resources/include/spgw.p4
```
control update_gtpu_checksum(
        ...
    )
    apply {
        ...
#ifdef WITH_SPGW_UDP_CSUM_UPDATE
        // Compute outer UDP checksum.
        update_checksum_with_payload(gtpu_udp.isValid(),
            {
                gtpu_ipv4.src_addr,
                gtpu_ipv4.dst_addr,
                8w0,
                gtpu_ipv4.protocol,
                gtpu_udp.len,
                gtpu_udp.sport,
                gtpu_udp.dport,
                gtpu_udp.len,
                gtpu,
                ipv4,
                // FIXME: we are assuming only UDP for downlink packets
                // How to conditionally switch between UDP/TCP/ICMP?
                udp
            },
            gtpu_udp.checksum,
            HashAlgorithm.csum16
        );
#endif // WITH_SPGW_UDP_CSUM_UPDATE
    }
}
```
Here they add a new instantiation called update_checksum_with_payload to the apply block if the feature to update checksum with payload is supported

###### 8
Source: https://github.com/wyan-all/onos-satellite/blob/master/pipelines/fabric/src/main/resources/include/bng.p4
```
control bng_ingress_downstream(
        inout parsed_headers_t hdr,
        inout fabric_metadata_t fmeta,
        inout standard_metadata_t smeta) {

    ...
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
Like seen in [#2](#2), conditional statements are added to the apply block based on the features supported. Here we add the `else if` conditional block iff the program supports IPV6 protocol