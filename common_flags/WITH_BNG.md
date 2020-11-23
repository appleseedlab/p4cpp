[Including the WITH_BNG flag](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/Makefile#L9)
```
fabric-bng:
	@./bmv2-compile.sh "fabric-bng" "-DWITH_BNG -DWITH_DOUBLE_VLAN_TERMINATION -DWITHOUT_XCONNECT"
```
This make command adds the `WITH_BNG` flag to the program whem compiling
<br/>
<br/>

[Header File](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/header.p4#L152)
```
#ifdef WITH_BNG

typedef bit<2> bng_type_t;
const bng_type_t BNG_TYPE_INVALID = 2w0x0;
const bng_type_t BNG_TYPE_UPSTREAM = 2w0x1;
const bng_type_t BNG_TYPE_DOWNSTREAM = 2w0x2;;

struct bng_meta_t {
    bit<2>  type; // upstream or downstream
    bit<32> line_id; // subscriber line
    bit<16> pppoe_session_id;
    bit<32> ds_meter_result; // for downstream metering
}
#endif // WITH_BNG
```
Adding new header fields and a new header struct related to the BNG feature

```
//Custom metadata definition
struct fabric_metadata_t {
    ...
#ifdef WITH_BNG
    bng_meta_t    bng;
#endif // WITH_BNG
#ifdef WITH_INT
    int_metadata_t int_meta;
#endif // WITH_INT
}

struct parsed_headers_t {
    ethernet_t ethernet;
    vlan_tag_t vlan_tag;
#if defined(WITH_XCONNECT) || defined(WITH_BNG) || defined(WITH_DOUBLE_VLAN_TERMINATION)
    vlan_tag_t inner_vlan_tag;
#endif // WITH_XCONNECT || WITH_BNG || WITH_DOUBLE_VLAN_TERMINATION
#ifdef WITH_BNG
    pppoe_t pppoe;
#endif // WITH_BNG
    mpls_t mpls;
    ...
}
```
Then they add fields to two other structs, the custom fabric metadata struct and parsed headers struct, that are related to the BNG feature.
<br/>
<br/>

[Parser](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/parser.p4#L66)
```
parser FabricParser (packet_in packet,
                     out parsed_headers_t hdr,
                     inout fabric_metadata_t fabric_metadata,
                     inout standard_metadata_t standard_metadata) {

    ...

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
    ...
```
When parsing the vlang tag, we also make sure to parse the inner_vlan_tag field that we added to the `parsed_headers_t` struct in header.p4 file if the `WITH_BNG` flag was defined. We also add the parse_inner_vlan_tag right after that if the `WITH_BNG` was defined (or if one of the other flags were defined as well).
In the parse_inner_vlan_tag state, we also parse the pppoe field if BNG is defined. If you recall, we also added the pppoe field to the `parsed_headers_t` struct if the `WITH_BNG` flag was defined.

```
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
```
Adding the `parse_pppoe` state if the `WITH_BNG` flag is defined. This is the state that's called in `parse_inner_vlan_tag` state.


```
control FabricDeparser(packet_out packet,in parsed_headers_t hdr) {

    apply {
        packet.emit(hdr.packet_in);
        ...
        packet.emit(hdr.vlan_tag);
#if defined(WITH_XCONNECT) || defined(WITH_BNG) || defined(WITH_DOUBLE_VLAN_TERMINATION)
        packet.emit(hdr.inner_vlan_tag);
#endif // WITH_XCONNECT || WITH_BNG || WITH_DOUBLE_VLAN_TERMINATION
#ifdef WITH_BNG
        packet.emit(hdr.pppoe);
#endif // WITH_BNG
        packet.emit(hdr.mpls);
    ...
    }
}
```
Here we make sure to emit the extra packets that we parse if `WITH_BNG` is defined.
<br/>
<br/>

[Size](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/size.p4#L8)
```
// Default sizes when building for BMv2.
#define BNG_MAX_SUBSC 8192
#define BNG_MAX_NET_PER_SUBSC 4
#define BNG_MAX_SUBSC_NET BNG_MAX_NET_PER_SUBSC * BNG_MAX_SUBSC
#ifdef WITH_BNG
    #define PORT_VLAN_TABLE_SIZE BNG_MAX_SUBSC
#else
    #define PORT_VLAN_TABLE_SIZE 1024
#endif // WITH_BNG
#define FWD_CLASSIFIER_TABLE_SIZE 1024
#define BRIDGING_TABLE_SIZE 1024
...
```
If BNG feature is supported, we defined the `PORT_VLAN_TABLE_SIZE` with the size as `BNG_MAX_SUBSC`, else `PORT_VLAN_TABLE_SIZE` is set to 1024
<br/>
<br/>

[Fabric](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/fabric.p4#L38)

```
#ifdef WITH_BNG
#include "include/bng.p4"
#endif // WITH_BNG
```
We include the BNG file if the feature is supported
<br/>

```
control FabricIngress (inout parsed_headers_t hdr,
                       inout fabric_metadata_t fabric_metadata,
                       inout standard_metadata_t standard_metadata) {

    ...

    apply {
        ...
#ifdef WITH_BNG
        bng_ingress.apply(hdr, fabric_metadata, standard_metadata);
#endif // WITH_BNG

    }
}
```
We make sure to invoke the apply block of bng_ingress with the data passed in if the BNG feature is supported.
<br/>

```
control FabricEgress (inout parsed_headers_t hdr,
                      inout fabric_metadata_t fabric_metadata,
                      inout standard_metadata_t standard_metadata) {

    ...

    apply {
        ...
#ifdef WITH_BNG
        bng_egress.apply(hdr, fabric_metadata, standard_metadata);
#endif // WITH_BNG
#ifdef WITH_INT
        process_int_main.apply(hdr, fabric_metadata, standard_metadata);
#endif
    }
}
```
Similar to to the FabricIngress, in the FabricEgress's apply block we make sure to call the apply block of bng_egress with the correct data passed in if the BNG feature is supported.
<br/>
<br/>

[Filtering](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/control/filtering.p4#L134)
```
control Filtering (inout parsed_headers_t hdr,
                   inout fabric_metadata_t fabric_metadata,
                   inout standard_metadata_t standard_metadata) {

        ...
        apply {
        ...

        // Set last_eth_type checking the validity of the L2.5 headers
        if (hdr.mpls.isValid()) {
            fabric_metadata.last_eth_type = ETHERTYPE_MPLS;
        } else {
            if (hdr.vlan_tag.isValid()) {
#if defined(WITH_XCONNECT) || defined(WITH_BNG) || defined(WITH_DOUBLE_VLAN_TERMINATION)
                if(hdr.inner_vlan_tag.isValid()) {
                    fabric_metadata.last_eth_type = hdr.inner_vlan_tag.eth_type;
                } else
#endif //  WITH_XCONNECT || WITH_BNG || WITH_DOUBLE_VLAN_TERMINATION
                    fabric_metadata.last_eth_type = hdr.vlan_tag.eth_type;
            } else {
                fabric_metadata.last_eth_type = hdr.ethernet.eth_type;
            }
        }

        ...
    }
}
```
If BNG feature is enabled, we check if inner_vlang_tag is valid, and if it is we set the fabric_metadata.last_eth_type field to the inner_vlang_tag's ether_type field