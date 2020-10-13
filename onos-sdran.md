Source: https://github.com/OpenNetworkingFoundation/onos-sdran/blob/8e99a8827af956946fa395546328ec021e63f9d3/pipelines/fabric/src/main/resources/include/parser.p4
```
#ifndef __PARSER__
#define __PARSER__
...
#endif
```
`ifndef` being used to make sure the file isn't being included twice
<br>
<br>

```
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
```
Inserting transition state to parse IPV6 packets if the program supports IPV6. Checking if the flag is set using `ifdef`
<br>
<br>

```
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
```
Adding new state if any of the flags (WITH_XCONNECT, WITH_BNG, WITH_DOUBLE_VLAN_TERMINATION) are defined. Checking using `if` and `defined` function
<br>
<br>

```
control FabricDeparser(packet_out packet,in parsed_headers_t hdr) {

    apply {
        ...
#if defined(WITH_XCONNECT) || defined(WITH_BNG) || defined(WITH_DOUBLE_VLAN_TERMINATION)
        packet.emit(hdr.inner_vlan_tag);
#endif // WITH_XCONNECT || WITH_BNG || WITH_DOUBLE_VLAN_TERMINATION
#ifdef WITH_BNG
        packet.emit(hdr.pppoe);
#endif // WITH_BNG
        packet.emit(hdr.mpls);
        ...
        packet.emit(hdr.ipv4);
#ifdef WITH_IPV6
        packet.emit(hdr.ipv6);
#endif // WITH_IPV6
        packet.emit(hdr.tcp);
        packet.emit(hdr.udp);
        packet.emit(hdr.icmp);
#ifdef WITH_INT
        ...
#ifdef WITH_INT_TRANSIT
        ...
#endif // WITH_INT_TRANSIT
#ifdef WITH_INT_SINK
        packet.emit(hdr.int_data);
#endif // WITH_INT_SINK
        packet.emit(hdr.intl4_tail);
#endif // WITH_INT
    }
}

#endif
```
Adding new deparser statements to emit specific header fields if they are supported. Using `ifdef` and `if` to check if the corresponding flags are set, and if so, we insert those emit statements
<br>
<br>

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
```
Adding a new statement to parse the intl4_shim part in the header if the WITH_INT flag is defined. Otherwise the state transitions to accept by default.

***

Source: https://github.com/OpenNetworkingFoundation/onos-sdran/blob/8e99a8827af956946fa395546328ec021e63f9d3/pipelines/fabric/src/main/resources/include/size.p4
```
#define BNG_MAX_SUBSC_NET BNG_MAX_NET_PER_SUBSC * BNG_MAX_SUBSC
#ifdef WITH_BNG
    #define PORT_VLAN_TABLE_SIZE BNG_MAX_SUBSC
#else
    #define PORT_VLAN_TABLE_SIZE 1024
#endif // WITH_BNG
```
Using `define` to create a variable whose value is computed using two other variables. Also using `ifdef` to see if a flag is set and define a variable based on the flag (set the PORT_VLAN_TABLE_SIZE to 1024 by default or to BNG_MAX_SUBSC if BNG is defined)

***

Source: https://github.com/OpenNetworkingFoundation/onos-sdran/blob/8e99a8827af956946fa395546328ec021e63f9d3/pipelines/basic/src/main/resources/include/defines.p4
```
#ifndef __DEFINES__
#define __DEFINES__

#define ETH_TYPE_IPV4 0x0800
#define IP_PROTO_TCP 8w6
#define IP_PROTO_UDP 8w17
#define IP_VERSION_4 4w4
#define IPV4_IHL_MIN 4w5
#define MAX_PORTS 511

#ifndef _BOOL
#define _BOOL bool
#endif
#ifndef _TRUE
#define _TRUE true
#endif
#ifndef _FALSE
#define _FALSE false
#endif
...
```
Using the `define` directive to define variables with a value, and also uses `ifdef` to check if certain flags are not set, and if not, it sets them using `define`