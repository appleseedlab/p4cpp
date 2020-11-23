[Makefile](https://github.com/ayman014/onos-mosaic/blob/f8e15abe44b3ec5d702e67287f4acdbde3a473a8/pipelines/fabric/impl/src/main/resources/Makefile#L6)
```
fabric-spgw:
	@./bmv2-compile.sh "fabric-spgw" "-DWITH_SPGW"

fabric-spgw-int:
	@./bmv2-compile.sh "fabric-spgw-int" "-DWITH_SPGW -DWITH_INT_SOURCE -DWITH_INT_TRANSIT"

fabric-full:
	@./bmv2-compile.sh "fabric-full" " -DWITH_MULTICAST -DWITH_IPV6 \
		-DWITH_SIMPLE_NEXT -DWITH_HASHED_NEXT -DWITH_BNG -DWITH_SPGW \
		-DWITH_INT_SOURCE -DWITH_INT_TRANSIT -DWITH_INT_SINK"
```
These three make commands define the `WITH_SPGW` flag when compiling the program
<br/>
<br/>

[Header](https://github.com/ayman014/onos-mosaic/blob/f8e15abe44b3ec5d702e67287f4acdbde3a473a8/pipelines/fabric/impl/src/main/resources/include/header.p4#L127)
```
#ifdef WITH_SPGW
// GTPU v1
header gtpu_t {
    bit<3>  version;    /* version */
    bit<1>  pt;         /* protocol type */
    bit<1>  spare;      /* reserved */
    bit<1>  ex_flag;    /* next extension hdr present? */
    bit<1>  seq_flag;   /* sequence no. */
    bit<1>  npdu_flag;  /* n-pdn number present ? */
    bit<8>  msgtype;    /* message type */
    bit<16> msglen;     /* message length */
    bit<32> teid;       /* tunnel endpoint id */
}

struct spgw_meta_t {
    direction_t       direction;
    bit<16>           ipv4_len;
    bit<32>           teid;
    bit<32>           s1u_enb_addr;
    bit<32>           s1u_sgw_addr;
#ifdef WITH_SPGW_PCC_GATING
    bit<16>           l4_sport;
    bit<16>           l4_dport;
    pcc_gate_status_t pcc_gate_status;
    sdf_rule_id_t     sdf_rule_id;
    pcc_rule_id_t     pcc_rule_id;
#endif // WITH_SPGW_PCC_GATING
}
#endif // WITH_SPGW
```
Including new header fields and struct if the `WITH_SPGW` flag is defined
```
//Custom metadata definition
struct fabric_metadata_t {
    ...
#ifdef WITH_SPGW
    spgw_meta_t   spgw;
#endif // WITH_SPGW
    ...
}
struct parsed_headers_t {
    ethernet_t ethernet;
    vlan_tag_t vlan_tag;
    ...
#ifdef WITH_SPGW
    ipv4_t gtpu_ipv4;
    udp_t gtpu_udp;
    gtpu_t gtpu;
    ipv4_t inner_ipv4;
    udp_t inner_udp;
#endif // WITH_SPGW
    ...
}
```
Adding new fields to the structs if the `WITH_SPGW` flag is defined
<br/>
<br/>

[Parser](https://github.com/ayman014/onos-mosaic/blob/f8e15abe44b3ec5d702e67287f4acdbde3a473a8/pipelines/fabric/impl/src/main/resources/include/parser.p4#L168)
```
parser FabricParser (packet_in packet,
                     out parsed_headers_t hdr,
                     inout fabric_metadata_t fabric_metadata,
                     inout standard_metadata_t standard_metadata) {

    bit<6> last_ipv4_dscp = 0;

    ...

    state parse_udp {
        packet.extract(hdr.udp);
        fabric_metadata.l4_sport = hdr.udp.sport;
        fabric_metadata.l4_dport = hdr.udp.dport;
        transition select(hdr.udp.dport) {
#ifdef WITH_SPGW
            UDP_PORT_GTPU: parse_gtpu;
#endif // WITH_SPGW
#ifdef WITH_INT
            default: parse_int;
#else
            default: accept;
#endif // WITH_INT
        }
    }
```
Adding new struct fields to a struct if SPGW feature is supported
<br/>

```
#ifdef WITH_SPGW
    state parse_gtpu {
        transition select(hdr.ipv4.dst_addr[31:32-S1U_SGW_PREFIX_LEN]) {
            // Avoid parsing GTP and inner headers if we know this GTP packet
            // is not to be processed by this switch.
            // FIXME: use parser value sets when support is ready in ONOS.
            // To set the S1U_SGW_PREFIX value at runtime.
            S1U_SGW_PREFIX[31:32-S1U_SGW_PREFIX_LEN]: do_parse_gtpu;
            default: accept;
        }
    }

    state do_parse_gtpu {
        packet.extract(hdr.gtpu);
        transition parse_inner_ipv4;
    }

    state parse_inner_ipv4 {
        packet.extract(hdr.inner_ipv4);
        last_ipv4_dscp = hdr.inner_ipv4.dscp;
        transition select(hdr.inner_ipv4.protocol) {
            PROTO_TCP: parse_tcp;
            PROTO_UDP: parse_inner_udp;
            PROTO_ICMP: parse_icmp;
            default: accept;
        }
    }

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
#endif // WITH_SPGW
```
Adding new parser states if the SPGW feature is enabled
<br/>

```
control FabricDeparser(packet_out packet,in parsed_headers_t hdr) {

    apply {
        packet.emit(hdr.packet_in);
        ...
#ifdef WITH_SPGW
        packet.emit(hdr.gtpu_ipv4);
        packet.emit(hdr.gtpu_udp);
        packet.emit(hdr.gtpu);
#endif // WITH_SPGW
        packet.emit(hdr.ipv4);
        ...
    }
}
```
Making sure to emit the respective fields into the packet if the SPGW feature is enabled
<br/>
<br/>

[Fabric](https://github.com/ayman014/onos-mosaic/blob/f8e15abe44b3ec5d702e67287f4acdbde3a473a8/pipelines/fabric/impl/src/main/resources/fabric.p4#L34)
```
#ifdef WITH_SPGW
#include "include/spgw.p4"
#endif // WITH_SPGW
```
If the SPGW feature is enabled, we include the SPGW file
<br/>

```
control FabricIngress (inout parsed_headers_t hdr,
                       inout fabric_metadata_t fabric_metadata,
                       inout standard_metadata_t standard_metadata) {

    PacketIoIngress() pkt_io_ingress;
    Filtering() filtering;
    Forwarding() forwarding;
    Acl() acl;
    Next() next;
    ...

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
        ...

    }
}
```
Adding statements to the ingress apply block to call SPGW functions iff the SPGW functionality is enabled
<br/>

```
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
Similarly, adding statements to the egress apply block to call SPGW functions iff the SPGW functionality is enabled
<br/>
<br/>

[Checksum](https://github.com/ayman014/onos-mosaic/blob/f8e15abe44b3ec5d702e67287f4acdbde3a473a8/pipelines/fabric/impl/src/main/resources/include/checksum.p4#L20)
```
#ifdef WITH_SPGW
#include "spgw.p4"
#endif // WITH_SPGW
```
If the SPGW feature is enabled, we include the SPGW file
<br/>

```
control FabricComputeChecksum(inout parsed_headers_t hdr,
                              inout fabric_metadata_t meta)
{
    apply {
        update_checksum(hdr.ipv4.isValid(),
            {
                ...
            },
            hdr.ipv4.hdr_checksum,
            HashAlgorithm.csum16
        );
#ifdef WITH_SPGW
        update_gtpu_checksum.apply(hdr.gtpu_ipv4, hdr.gtpu_udp, hdr.gtpu,
                                   hdr.ipv4, hdr.udp);
#endif // WITH_SPGW
    }
}
```
If SPGW feature is enabled, we call the update_gtpu_checksum's apply function when computing the checksum