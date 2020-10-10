Source: https://github.com/mc36/freeRouter/blob/a5619d78d1b563e0a5dd8fa31fcd7ccc0c42262f/misc/p4bf/include/hdr_ig_headers.p4
```
#undef NEED_UDP2

#ifdef HAVE_L2TP
#define NEED_UDP2
#endif

#ifdef HAVE_VXLAN
#define NEED_UDP2
#endif

#undef NEED_ETH4

#ifdef HAVE_TAP
#define NEED_ETH4
#endif
```
Using `ifdef` to see if specific protocols or tools are being used in the current deployment, if so, then it declares a new flag using `define` to indicate requirements
<br>

```
struct headers {
    cpu_header_t cpu;
    ethernet_t ethernet;
    ...
#ifdef HAVE_GRE
    gre_t gre2;
#endif
#ifdef NEED_UDP2
    udp_t udp2;
#endif
#ifdef HAVE_L2TP
    l2tp_t l2tp2;
#endif
#ifdef HAVE_VXLAN
    vxlan_t vxlan2;
#endif
#ifdef NEED_ETH4
    ethernet_t eth4;
#endif
    ...
}
```
Using `ifdef` to see which header fields are required/defined, and based on that they are adding fields to the header struct.
<br>

***

Source: https://github.com/mc36/freeRouter/blob/174f2f3b9e2ce0338b565ac2db723de5fd2872bd/misc/p4bf/include/ig_prs_main.p4
```
#ifndef _INGRESS_PARSER_P4_
#define _INGRESS_PARSER_P4_

/*------------------ I N G R E S S   P A R S E R -----------------------------*/

parser ig_prs_main(packet_in pkt,
                   /* User */
                   out headers hdr, out ingress_metadata_t ig_md,
                   /* Intrinsic */
                   out ingress_intrinsic_metadata_t ig_intr_md)
{

    Checksum()ipv4_checksum;
#ifdef HAVE_NAT
    Checksum() tcp_checksum;
    Checksum() udp_checksum;
#endif

    state start {
        pkt.extract(ig_intr_md);
        pkt.advance(PORT_METADATA_SIZE);
        ig_md.always_zero = 0;
    #ifdef HAVE_PPPOE
        ig_md.pppoe_ctrl_valid = 0;
        ig_md.pppoe_data_valid = 0;
    #endif
        ...
    }
    ...
}
```
Using `ifndef` to make sure a main flag isn't being redefined.
Using `ifdef` to see if NAT is being used, and if so, checksum fields for TCP and UDP are included; also used to see if a flag is defined and if so, a corresponding new field in the start state is added.
<br>

```
#ifdef HAVE_PPPOE
    state prs_pppoeCtrl {
        pkt.extract(hdr.pppoeC);
        ig_md.pppoe_ctrl_valid = 1;
        transition accept;
    }

    state prs_pppoeData {
        pkt.extract(hdr.pppoeD);
        ig_md.pppoe_data_valid = 1;
        transition select(hdr.pppoeD.ppptyp) {
PPPTYPE_IPV4:
            prs_ipv4;
PPPTYPE_IPV6:
            prs_ipv6;
#ifdef HAVE_MPLS
PPPTYPE_MPLS_UCAST:
            prs_mpls0;
#endif
        default:
            prs_pppoeDataCtrl;
        }
    }

    state prs_pppoeDataCtrl {
        ig_md.pppoe_ctrl_valid = 1;
        transition accept;
    }
#endif

#ifdef HAVE_TAP
    state prs_eth6 {
        pkt.extract(hdr.eth6);
        transition accept;
    }
#endif
```
`ifdef` flags being used to define new states and to add new cases/matches to transitions select blocks
<br>

```
state prs_ipv4 {
        pkt.extract(hdr.ipv4);
        ipv4_checksum.add(hdr.ipv4);
        ig_md.ipv4_valid = 1;
        ig_md.l4_lookup = pkt.lookahead<l4_lookup_t>();
#ifdef HAVE_NAT
        tcp_checksum.subtract({hdr.ipv4.src_addr});
        tcp_checksum.subtract({hdr.ipv4.dst_addr});
        udp_checksum.subtract({hdr.ipv4.src_addr});
        udp_checksum.subtract({hdr.ipv4.dst_addr});
#endif
        ...
}
```
`ifdef` being used to do secific calculations inside a state

***

Source: https://github.com/mc36/freeRouter/blob/56a13cf8828b1f159a7d04280f4dd18648f64a7d/misc/p4bf/include/ig_ctl.p4
```
control ig_ctl(inout headers hdr, inout ingress_metadata_t ig_md,
               in ingress_intrinsic_metadata_t ig_intr_md,
               in ingress_intrinsic_metadata_from_parser_t ig_prsr_md,
               inout ingress_intrinsic_metadata_for_deparser_t ig_dprsr_md,
               inout ingress_intrinsic_metadata_for_tm_t ig_tm_md)
{
#ifdef HAVE_NOHW

    apply {
        if (ig_intr_md.ingress_port == CPU_PORT) {
            ig_tm_md.ucast_egress_port =(PortId_t) hdr.cpu.port;
            ig_tm_md.bypass_egress = 1;
            hdr.cpu.setInvalid();
        } else {
            hdr.cpu.setValid();
            hdr.cpu.port = ig_md.ingress_id;
            ig_tm_md.ucast_egress_port = CPU_PORT;
            ig_tm_md.bypass_egress = 1;
        }
    }

#else
    IngressControlBundle() ig_ctl_bundle;
#ifdef HAVE_MPLS
    IngressControlMPLS()ig_ctl_mpls;
#endif
    ...
    apply {
        ...
    }
#endif
}
```
Here the `ifdef` and `else` directives are used to evaluate the flag and depending on that it either inserts an apply block or instantiate predefined control blocks and an apply block

***

Source: https://github.com/mc36/freeRouter/blob/2d24fc554f916e043bfcee29b8318c8e7b2e1b80/misc/p4bf/include/ig_ctl_tunnel.p4
```
#ifdef HAVE_TUN

control IngressControlTunnel(...)
{
#ifdef HAVE_L2TP
    SubIntId_t l2tp_hit;
#endif
    ...
}
#endif
```
Using `ifdef` to include the main control block in this file and to define new variables based on the respective flags
<br>

```
#ifdef HAVE_GRE
    action act_tunnel_gre(SubIntId_t port) {
        hdr.ethernet.ethertype = hdr.gre.gretyp;
        ig_md.source_id = port;
        ...
    }
#endif
```
Using `ifdef` to define actions