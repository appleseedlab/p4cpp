###### 1.1
Source: https://github.com/breezestars/onos-barefoot/blob/440d67b3a97363ed9b15894162c4c8bac7ebff05/pipelines/fabric/src/main/resources/include/parser.p4#L22
```
parser FabricParser (packet_in packet,
                     out parsed_headers_t hdr,
                     inout fabric_metadata_t fabric_metadata,
                     inout standard_metadata_t standard_metadata) {
    ...
#ifdef WITH_BNG
    state parse_pppoe {
        packet.extract(hdr.pppoe);
        transition select(hdr.pppoe.protocol) {
            PPPOE_PROTOCOL_MPLS: parse_mpls;
            PPPOE_PROTOCOL_IP4: parse_ipv4;
#ifdef WITH_IPV6
            PPPOE_PROTOCOL_IP6: parse_ipv6;
#endif // WITH_IPV6
            default: accept;
        }
    }
#endif // WITH_BNG
...
#ifdef WITH_IPV6
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
}
```
Here they insert new states into the parser based on the features supported by the program (eg: if IPV6 is supported, we add a new state into the parser to extract IPV6 headers)

###### 2.1

Source: https://github.com/imec-idlab/onos-whisper/blob/9150fcb524ac756492b109c7f4ef5713069744f3/pipelines/fabric/src/main/resources/include/parser.p4#L22
```
parser FabricParser (packet_in packet,
                     out parsed_headers_t hdr,
                     inout fabric_metadata_t fabric_metadata,
                     inout standard_metadata_t standard_metadata) {
    ...
#ifdef WITH_IPV6
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
}
```
Here a whole new state is added to parse IPV6 headers iff the program is set to support IPV6. And in the second block we see that deciding whether to transition to accept state or to another state depends on the WITH_INT flag.
Very similar content found in: 
* https://github.com/breezestars/vomci-onos/blob/848a7c61cddcce0db50877af5b6af1eae3edc22e/pipelines/fabric/src/main/resources/include/parser.p4
* https://github.com/breezestars/onos-barefoot/blob/440d67b3a97363ed9b15894162c4c8bac7ebff05/pipelines/fabric/src/main/resources/include/parser.p4
* https://github.com/dnosproject/onos/blob/cb9ea2d617dd1320e1c3ab45c6118af790944dc1/pipelines/fabric/src/main/resources/include/parser.p4

###### 2.2

```
#ifdef WITH_INT
    state parse_int {
        transition select(last_ipv4_dscp) {
            INT_DSCP &&& INT_DSCP: parse_intl4_shim;
            default: accept;
        }
    }

    state parse_intl4_shim {
        packet.extract(hdr.intl4_shim);
        transition parse_int_header;
    }

    state parse_int_header {
        packet.extract(hdr.int_header);
        // If there is no INT metadata but the INT header (plus shim and tail)
        // exists, default value of length field in shim header should be
        // INT_HEADER_LEN_WORDS.
        transition select (hdr.intl4_shim.len_words) {
            INT_HEADER_LEN_WORDS: parse_intl4_tail;
            default: parse_int_data;
        }
    }

    state parse_int_data {
#ifdef WITH_INT_SINK
        // Parse INT metadata stack, but not tail
        packet.extract(hdr.int_data, (bit<32>) (hdr.intl4_shim.len_words - INT_HEADER_LEN_WORDS) << 5);
        transition parse_intl4_tail;
#else // not interested in INT data
        transition accept;
#endif // WITH_INT_SINK
    }

    state parse_intl4_tail {
        packet.extract(hdr.intl4_tail);
        transition accept;
    }
#endif // WITH_INT
}
```

In the same file, you can observe that if a single feature is enabled for the program, then we include multiple parser states, and inside some of the states, we include new parser and transition statements based on additional sub flags
