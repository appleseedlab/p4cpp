Source: https://github.com/xinshengzzy/AHashFlow/blob/59daaff50b92afc05675bdc1d05b83b914e61ca8/src/switch-8.2.0/p4src/acl.p4
```
#define EGRESS_IPV6_ACL_KEY \
        EGRESS_ACL_KEY_PORT_LABEL       : ternary; \
        EGRESS_ACL_KEY_BD_LABEL         : ternary; \
        EGRESS_ACL_KEY_IPV6_SA          : ternary; \
        EGRESS_ACL_KEY_IPV6_DA          : ternary; \
        EGRESS_ACL_KEY_IPV6_PROTO       : ternary;
```
Using `define` to declare new set of variable - ternary?
<br>
<br>

```
#if defined(INT_ENABLE) || defined(POSTCARD_ENABLE)
@pragma pa_do_not_bridge egress i2e_metadata.mirror_session_id
#endif
```
Checking `if` INT_ENABLE or POSTCARD_ENABLE flags are defined, and if so, adds a pragma code
<br>
<br>

```
/*****************************************************************************/
/* Egress ACL l4 port range                                                  */
/*****************************************************************************/
...
control process_egress_l4port {
#ifdef EGRESS_ACL_ENABLE
    apply(egress_l4port_fields);
#ifndef EGRESS_ACL_RANGE_DISABLE
    apply(egress_l4_src_port);
    apply(egress_l4_dst_port);
#endif /* EGRESS_ACL_RANGE_DISABLE */
#endif /* EGRESS_ACL_ENABLE */
}
```
Using `ifdef` and `ifndef` to check if specific flags are set or not and depending on that, inserts apply respective apply statements in the control block

***

Source:https://github.com/xinshengzzy/AHashFlow/blob/59daaff50b92afc05675bdc1d05b83b914e61ca8/src/switch-8.2.0/p4src/switch.p4 
```
header_type egress_metadata_t {
    fields {
#ifdef PTP_ENABLE
        capture_tstamp_on_tx : 1;              /* request for packet departure time capture */
#endif
        bypass : 1;                            /* bypass egress pipeline */
        port_type : 2;                         /* egress port type */
        payload_length : 16;                   /* payload length for tunnels */
        ...
    }
}
```
Using `ifdef` to insert new fields in the egress_metadata_t header struct
<br>
<br>

```
#ifdef SFLOW_ENABLE
@pragma pa_atomic ingress ingress_metadata.sflow_take_sample
@pragma pa_solitary ingress ingress_metadata.sflow_take_sample
#endif
```
Using `ifdef` to insert `pragma` statements
<br>
<br>

```
control ingress {
    /* input mapping - derive an ifindex */
    process_ingress_port_mapping();

    /* read and apply system configuration parametes */
    process_global_params();
#ifdef PKTGEN_ENABLE
    if (VALID_PKTGEN_PACKET) {
        /* process pkt_gen generated packets */
        process_pktgen();
    } else {
#endif /* PKTGEN_ENABLE */
/* process outer packet headers */
    process_validate_outer_header();
    ...
    }
    ...
}
```
Using `ifdef` to insert control block statements, which potentially change the scope of some lines in this case because an `if` and `else` conditions are added if the flag is defined
<br>
<br>
```
    if (l2_metadata.lkp_pkt_type == L2_UNICAST) {
#if defined(L2_DISABLE) && defined(L2_MULTICAST_DISABLE) && defined(L3_MULTICAST_DISABLE)
        {
            {
#else
        apply(rmac) {
            rmac_hit {
#endif /* L2_DISABLE && L2_MULTICAST_DISABLE && L3_MULTICAST_DISABLE */
                if (DO_LOOKUP(L3)) {
                    if ((l3_metadata.lkp_ip_type == IPTYPE_IPV4) and
                        (ipv4_metadata.ipv4_unicast_enabled == TRUE)) {
                            /* router ACL/PBR */
#ifndef RACL_SWAP
                            process_ipv4_racl();
#endif /* !RACL_SWAP */
                            process_ipv4_urpf();
                            process_ipv4_fib();
#ifdef RACL_SWAP
                            process_ipv4_racl();
#endif /* RACL_SWAP */

#ifdef IPV6_DISABLE
		    }
#else
                    } else {
                        if ((l3_metadata.lkp_ip_type == IPTYPE_IPV6) and
                            (ipv6_metadata.ipv6_unicast_enabled == TRUE)) {
                            /* router ACL/PBR */
#ifndef RACL_SWAP
                            process_ipv6_racl();
#endif /* !RACL_SWAP */
                            process_ipv6_urpf();
                            process_ipv6_fib();
#ifdef RACL_SWAP
                            process_ipv6_racl();
#endif /* RACL_SWAP */
                        }
                    }
#endif /* IPV6_DISABLE */
                    process_urpf_bd();
                }
            }
        }
    } else {
        process_multicast();
    }
```
A big block of code whose scope is defined by the flags. `if`, `else`, and `else if` statements are added to the change how this block works when compiled. For instance, in the starting of this block, we either define this as a function or as a apply block based on the `L2_DISABLE, `L2_MULTICAST_DISABLE, `L3_MULTICAST_DISABLE` flags

***

Source: https://github.com/xinshengzzy/AHashFlow/blob/59daaff50b92afc05675bdc1d05b83b914e61ca8/src/switch-8.2.0/p4src/includes/p4features.h

```
...
/* Standard Model */
#define EGRESS_FILTER
#define FABRIC_ENABLE
#define EGRESS_ACL_ENABLE
...
```
All required flags being defined in a header file using `define`
***

Source: https://github.com/xinshengzzy/AHashFlow/blob/59daaff50b92afc05675bdc1d05b83b914e61ca8/src/switch-8.2.0/p4src/fabric.p4
```
header_type fabric_metadata_t {
    fields {
        packetType : 3;
        fabric_header_present : 1;
        reason_code : 16;              /* cpu reason code */

#ifdef FABRIC_ENABLE
        dst_device : 8;                /* destination device id */
        dst_port : 16;                 /* destination port id */
#endif /* FABRIC_ENABLE */
    }
}
```
Using `ifdef` to add new header fields depending on whether fabric mode is enabled or not
<br>
<br>

```
#ifdef FABRIC_ENABLE
action terminate_fabric_unicast_packet() {
    ...
}

action switch_fabric_unicast_packet() {
    ...
}
#ifndef MULTICAST_DISABLE
action terminate_fabric_multicast_packet() {
    ...
}

action switch_fabric_multicast_packet() {
    ...
}
#endif /* MULTICAST_DISABLE */
#endif /* FABRIC_ENABLE */
```
```
/*****************************************************************************/
/* Fabric header - source lookup                                             */
/*****************************************************************************/
#ifdef FABRIC_ENABLE
action set_ingress_ifindex_properties(l2xid) {
    modify_field(ig_intr_md_for_tm.level2_exclusion_id, l2xid);
}

table fabric_ingress_src_lkp {
    reads {
        fabric_header_multicast.ingressIfindex : exact;
    }
    actions {
        nop;
        set_ingress_ifindex_properties;
    }
    size : 1024;
}
#endif /* FABRIC_ENABLE */
```
Again checking if fabric mode is enabled and inserting new actions and tables related ot fabric mode depending on the flag
<br>
<br>

```
/*****************************************************************************/
/* Ingress fabric header processing                                          */
/*****************************************************************************/
control process_ingress_fabric {
    if (ingress_metadata.port_type != PORT_TYPE_NORMAL) {
        apply(fabric_ingress_dst_lkp);
#ifdef FABRIC_ENABLE
        if (ingress_metadata.port_type == PORT_TYPE_FABRIC) {
            if (valid(fabric_header_multicast)) {
                apply(fabric_ingress_src_lkp);
            }
            if (tunnel_metadata.tunnel_terminate == FALSE) {
                apply(native_packet_over_fabric);
            }
        }
#endif /* FABRIC_ENABLE */
    }
}
```
Another place where statements are inserted in a control block based on the FABRIC_ENABLED flag. Many more similar instances occur in this file that depend on whether the FABRIC_ENABLED flag is defined or not

***

Source:
```
header_type multicast_metadata_t {
    fields {
        ...
#ifdef FABRIC_ENABLE
        mcast_grp_a : 16;
        mcast_grp_b : 16;
        ingress_rid : 16;
        l1_exclusion_id : 16;
#endif /* FABRIC_ENABLE */
    }
}
```
Again checking for the FABRIC_ENABLE flag using `ifdef` and depending on that we have extra header fields

***

Source: https://github.com/xinshengzzy/AHashFlow/blob/59daaff50b92afc05675bdc1d05b83b914e61ca8/src/switch-8.2.0/p4src/ipv4.p4
```
table validate_outer_ipv4_packet {
    reads {
        ...
#if !defined(L2_MULTICAST_DISABLE) || !defined(L3_MULTICAST_DISABLE)
        ipv4.dstAddr mask 0xFFFFFF00 : ternary;
#endif /* !defined(L2_MULTICAST_DISABLE) || !defined(L3_MULTICAST_DISABLE) */
    }
    actions {
        set_valid_outer_ipv4_packet;
#if !defined(L2_MULTICAST_DISABLE) || !defined(L3_MULTICAST_DISABLE)
        set_valid_outer_ipv4_llmc_packet;
        set_valid_outer_ipv4_mc_packet;
#endif /* !defined(L2_MULTICAST_DISABLE) || !defined(L3_MULTICAST_DISABLE) */
        set_malformed_outer_ipv4_packet;
    }
    size : VALIDATE_PACKET_TABLE_SIZE;
}
#endif /* L3_DISABLE && IPV4_DISABLE */
```
Using `if` and `defined` to check if L2_MULTICAST_DISABLE and other flags are defined or not to insert statements into the table readers and actions
<br>
<br>

```
control process_ipv4_urpf {
#if !defined(L3_DISABLE) && !defined(IPV4_DISABLE) && !defined(URPF_DISABLE)
    /* unicast rpf lookup */
    if (ipv4_metadata.ipv4_urpf_mode != URPF_MODE_NONE) {
#ifdef IPV4_LOCAL_HOST_TABLE_SIZE
        apply(ipv4_local_hosts_urpf) {
            on_miss {
#endif
                apply(ipv4_urpf) {
                    on_miss {
                        apply(ipv4_urpf_lpm);
                    }
                }
#ifdef IPV4_LOCAL_HOST_TABLE_SIZE
            }
        }
#endif
    }
#endif /* L3_DISABLE && IPV4_DISABLE && URPF_DISABLE */
}
```
In this case, they have a control block fille with statement iff either of those three flags are not defined, and an outer apply statement is added iff the IPV4_LOCAL_HOST_TABLE_SIZE is defined. So control block and apply blocks are added based on certain flags, which are checked using `if`, `ifdef`, and `defined`