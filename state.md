Source: https://github.com/breezestars/onos-barefoot/blob/440d67b3a97363ed9b15894162c4c8bac7ebff05/pipelines/fabric/src/main/resources/include/parser.p4
```
    state parse_tcp {
        packet.extract(hdr.tcp);
        fabric_metadata.l4_sport = hdr.tcp.sport;
        fabric_metadata.l4_dport = hdr.tcp.dport;
#ifdef WITH_INT
        transition parse_int;
#else
        transition accept;
#endif // WITH_INT
    }
    ...
    state parse_inner_udp {
        packet.extract(hdr.inner_udp);
        fabric_metadata.l4_sport = hdr.inner_udp.sport;
        fabric_metadata.l4_dport = hdr.inner_udp.dport;
#ifdef WITH_INT
        transition parse_int;
#else
        transition accept;
#endif // WITH_INT
    }
```
Here, inside each parse state, we either transition to accept or parse an extra field (INT - In-band network telemetry) if the program supports INT

```
#ifdef WITH_INT
    ...
    state parse_int_data {
#ifdef WITH_INT_SINK
        // Parse INT metadata stack, but not tail
        packet.extract(hdr.int_data, (bit<32>) (hdr.intl4_shim.len_words - INT_HEADER_LEN_WORDS) << 5);
        transition parse_intl4_tail;
#else // not interested in INT data
        transition accept;
#endif // WITH_INT_SINK
    }
#endif // WITH_INT
```
Here we include the parse_int_data state iff we support INT, and we extract the INT data iff WITH_INT_SINK is defined, otherwise we just transition to accept

***

transition block inside a state
```
state parse_ethernet {
        packet.extract(hdr.ethernet);
        fabric_metadata.eth_type = hdr.ethernet.eth_type;
        fabric_metadata.vlan_id = DEFAULT_VLAN_ID;
        transition select(hdr.ethernet.eth_type){
            ETHERTYPE_VLAN: parse_vlan_tag;
            ETHERTYPE_MPLS: parse_mpls;
            ETHERTYPE_IPV4: parse_ipv4;
#ifdef WITH_IPV6
            ETHERTYPE_IPV6: parse_ipv6;
#endif // WITH_IPV6
            default: accept;
        }
    }

    state parse_vlan_tag {
        packet.extract(hdr.vlan_tag);
        transition select(hdr.vlan_tag.eth_type){
            ETHERTYPE_IPV4: parse_ipv4;
#ifdef WITH_IPV6
            ETHERTYPE_IPV6: parse_ipv6;
#endif // WITH_IPV6
            ETHERTYPE_MPLS: parse_mpls;
#if defined(WITH_XCONNECT) || defined(WITH_BNG)
            ETHERTYPE_VLAN: parse_inner_vlan_tag;
#endif // WITH_XCONNECT
            default: accept;
        }
    }
```
In inside both of these states, we have a transition block to tell the program to which state to transition to at the end. If add new transition states based on what the program supports (eg: if IPV6 is supported, then we add an option to the transition block to go to parse_ipv6 state if matched)