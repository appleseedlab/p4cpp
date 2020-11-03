###### 1.1
Source: https://github.com/breezestars/onos-barefoot/blob/440d67b3a97363ed9b15894162c4c8bac7ebff05/pipelines/fabric/src/main/resources/include/header.p4
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
Here they are adding declaring new header types if the WITH_SPGW type is supported. And they also add a supporting metadata struct inside which specific variables are defined if WITH_SPGW_PCC_GATING is declared somewhere.
###### 1.2
```
#ifdef WITH_BNG

typedef bit<2> bng_type_t;
const bng_type_t BNG_TYPE_INVALID = 2w0x0;
const bng_type_t BNG_TYPE_UPSTREAM = 2w0x1;
const bng_type_t BNG_TYPE_DOWNSTREAM = 2w0x2;;

struct bng_meta_t {
    bit<2>  type; // upstream or downstream
    bit<32> line_id; // subscriber line
    bit<32> ds_meter_result; // for downstream metering
}
#endif // WITH_BNG
```
Similarly, here they are adding new constants and a new struct if BNG (broadband network gateway) is supported
###### 1.3
```
struct parsed_headers_t {
    ethernet_t ethernet;
    vlan_tag_t vlan_tag;
#if defined(WITH_XCONNECT) || defined(WITH_BNG)
    vlan_tag_t inner_vlan_tag;
#endif // WITH_XCONNECT || WITH_BNG
#ifdef WITH_BNG
    pppoe_t pppoe;
#endif // WITH_BNG
    mpls_t mpls;
#ifdef WITH_SPGW
    ipv4_t gtpu_ipv4;
    udp_t gtpu_udp;
    gtpu_t gtpu;
    ipv4_t inner_ipv4;
    udp_t inner_udp;
#endif // WITH_SPGW
    ipv4_t ipv4;
#ifdef WITH_IPV6
    ipv6_t ipv6;
#endif // WITH_IPV6
    ...
}
```
Here new fields inside a struct are added based on what features are supported. <br>
Note: Some header structs are defined irrespective of whether that feature is supported or not (in the same file, you can see that `header ipv6_t` is defined whether or not IPV6 is defined)

###### 2.1

Source: https://github.com/opennetworkinglab/onos/blob/940332696982e212decfe08acb418d01457c0280/pipelines/fabric/impl/src/main/resources/include/int/int_header.p4

```
#ifdef WITH_INT_SINK
// Report Telemetry Headers
header report_fixed_header_t {
    bit<4>  ver;
    bit<4>  nproto;
    bit<1>  d;
    bit<1>  q;
    bit<1>  f;
    bit<15> rsvd;
    bit<6>  hw_id;
    bit<32> seq_no;
    bit<32> ingress_tstamp;
}

// Telemetry drop report header
header drop_report_header_t {
    bit<32> switch_id;
    bit<16> ingress_port_id;
    bit<16> egress_port_id;
    bit<8>  queue_id;
    bit<8>  drop_reason;
    bit<16> pad;
}

// Switch Local Report Header
header local_report_header_t {
    bit<32> switch_id;
    bit<16> ingress_port_id;
    bit<16> egress_port_id;
    bit<8>  queue_id;
    bit<24> queue_occupancy;
    bit<32> egress_tstamp;
}

header_union local_report_t {
    drop_report_header_t drop_report_header;
    local_report_header_t local_report_header;
}
#endif // WITH_INT_SINK
...
```
Here we can see that if WITH_INT_SINK is defined, we add new header structs and a header_union struct to let the P4 compiler that only one of those two will be used.