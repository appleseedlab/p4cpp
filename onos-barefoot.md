Source: https://github.com/breezestars/onos-barefoot/blob/440d67b3a97363ed9b15894162c4c8bac7ebff05/pipelines/fabric/src/main/resources/include/parser.p4

```
#ifndef __PARSER__
#define __PARSER__
...
#endif
```
Using `ifdef` to check if the current file has already been included or not, and include it if it isn't already defined
<br>
<br>

```
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
Here `ifdef`is being to check whether IPV6 protocol is supported, if so, then we include that field as a transition state. Similarly the `if` directive is being used to check whether WITH_XCONNECT or WITH_BNG is defined, and if so, we add another transition state.
<br>
<br>

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
        ...
    }
}
```
Here they use `ifdef` if the IPV6 protocol is supported, and if so, we insert code to emit that field in the deparser.
<br>
<br>