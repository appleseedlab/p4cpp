Note: WITH_INT is related to WITH_INT_SOURCE, WITH_INT_TRANSIT, and WITH_INT_SINK, so this file also includes their uses

[Defining the WITH_INT flag - Basic-Pro Pipeline](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/basic-pro/src/main/resources/include/define.p4#L22) | [Defining the WITH_INT flag - Fabric Pipeline](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/fabric/src/main/resources/include/define.p4#L22)
```
#if defined(WITH_INT_SOURCE) || defined(WITH_INT_TRANSIT) || defined(WITH_INT_SINK)
#define WITH_INT
#endif
```
The INT_FLAG is defined iff one of those three flags are defined by the user; the user defines those flags in this [Makefile - Basic-Pro Pipeline](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/basic-pro/src/main/resources/Makefile#L9):
```
fabric-int:
	@./bmv2-compile.sh "fabric-int" "-DWITH_INT_SOURCE -DWITH_INT_TRANSIT"

fabric-spgw-int:
	@./bmv2-compile.sh "fabric-spgw-int" "-DWITH_SPGW -DWITH_INT_SOURCE -DWITH_INT_TRANSIT"
```
[Makefile - Fabric pipeline](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/fabric/src/main/resources/Makefile#L12)
```
fabric-int:
	@./bmv2-compile.sh "fabric-int" "-DWITH_INT_SOURCE -DWITH_INT_TRANSIT"

fabric-spgw-int:
	@./bmv2-compile.sh "fabric-spgw-int" "-DWITH_SPGW -DWITH_INT_SOURCE -DWITH_INT_TRANSIT"
```
In most files, if the WITH_INT flag is defined, it includes the [int_main.p4](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/fabric/src/main/resources/include/int/int_main.p4), which then includes the files that correspond to the flags defined. Hence, the WITH_INT is used as a flag to indicate that at least one of the INT operation is enabled.
<br/>
<br/>

[Header - Basic-Pro Pipeline](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/basic-pro/src/main/resources/include/header.p4#L143) | Similar block of code for [Header - Fabric Pipeline](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/fabric/src/main/resources/include/header.p4#L167) 
```
//Custom metadata definition
struct fabric_metadata_t {
    bit<16>       eth_type;
    bit<16>       ip_eth_type;
    vlan_id_t     vlan_id;
    bit<3>        vlan_pri;
    bit<1>        vlan_cfi;
    mpls_label_t  mpls_label;
    bit<8>        mpls_ttl;
    _BOOL         skip_forwarding;
    _BOOL         skip_next;
    fwd_type_t    fwd_type;
    next_id_t     next_id;
    _BOOL         is_multicast;
    _BOOL         is_controller_packet_out;
    _BOOL         clone_to_cpu;
    bit<8>        ip_proto;
    bit<16>       l4_sport;
    bit<16>       l4_dport;
#ifdef WITH_SPGW
    spgw_meta_t   spgw;
#endif // WITH_SPGW
#ifdef WITH_INT
    int_metadata_t int_meta;
#endif // WITH_INT
}
```
Adding new struct field for INT data if the WITH_INT flag is defined
</br>
```
struct parsed_headers_t {
    ...
#ifdef WITH_INT_SINK
    // INT Report encap
    ethernet_t report_ethernet;
    ipv4_t report_ipv4;
    udp_t report_udp;
    // INT Report header (support only fixed)
    report_fixed_header_t report_fixed_header;
    // local_report_t report_local;
#endif // WITH_INT_SINK
#ifdef WITH_INT
    // INT specific headers
    intl4_shim_t intl4_shim;
    int_header_t int_header;
    int_switch_id_t int_switch_id;
    int_port_ids_t int_port_ids;
    int_hop_latency_t int_hop_latency;
    int_q_occupancy_t int_q_occupancy;
    int_ingress_tstamp_t int_ingress_tstamp;
    int_egress_tstamp_t int_egress_tstamp;
    int_q_congestion_t int_q_congestion;
    int_egress_port_tx_util_t int_egress_tx_util;
#ifdef WITH_INT_SINK
    int_data_t int_data;
#endif // WITH_INT_SINK
    intl4_tail_t intl4_tail;
#endif //WITH_INT
}
```
Again adding new struct fields if the WITH_INT_SINK and/or WITH_INT flags are defined. If WITH_INT_SINK is defined, then it means that WITH_INT is also defined, so that whole block of struct fields are added. But if one of the other three INT flags are defined, then just the fields inside the WITH_INT macro block are defined.
<br/>
<br/>

[Fabric](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/fabric/src/main/resources/fabric.p4#L42)
```
#ifdef WITH_INT
#include "include/int/int_main.p4"
#endif // WITH_INT
```
We include the core file if WITH_INT is defined


```
control FabricIngress (inout parsed_headers_t hdr,
                       inout fabric_metadata_t fabric_metadata,
                       inout standard_metadata_t standard_metadata) {

    ...

    apply {
        ...
        if (fabric_metadata.skip_next == _FALSE) {
            next.apply(hdr, fabric_metadata, standard_metadata);
#ifdef WITH_PORT_COUNTER
            // FIXME: we're not counting pkts punted to cpu or forwarded via
            // multicast groups. Remove when gNMI support will be there.
            port_counters_control.apply(hdr, fabric_metadata, standard_metadata);
#endif // WITH_PORT_COUNTER
#if defined(WITH_INT_SOURCE) || defined(WITH_INT_SINK)
            process_set_source_sink.apply(hdr, fabric_metadata, standard_metadata);
#endif
        }
    }
}

control FabricEgress (inout parsed_headers_t hdr,
                      inout fabric_metadata_t fabric_metadata,
                      inout standard_metadata_t standard_metadata) {

    PacketIoEgress() pkt_io_egress;
    EgressNextControl() egress_next;

    apply {
        ...
#ifdef WITH_INT
        process_int_main.apply(hdr, fabric_metadata, standard_metadata);
#endif
    }
}
```
Adding new apply statements pretaining to INT feature
<br/>
<br/>

[Parser](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/fabric/src/main/resources/include/parser.p4#L148)
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
    ...
    #ifdef WITH_SPGW
    state parse_gtpu {
        ...
    }

    state do_parse_gtpu {
       ...
    }

    state parse_inner_ipv4 {
        ...
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
Here the last transition statement or default transition select statement is dependent on whether the WITH_INT flag is defined. If it is, then we transition to the parse_int state, else we transition to accept.
<br>

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
And here they are adding new states related to INT features to which the some of the other states transition to (as seen in the previous code block). And in the `parse_int_data` state we add new statements based on whether some sub INT functionalities are defined.
<br/>
<br/>

[Deparser](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/fabric/src/main/resources/include/parser.p4#L261)
```
control FabricDeparser(packet_out packet,in parsed_headers_t hdr) {

    apply {
        packet.emit(hdr.packet_in);
#ifdef WITH_INT_SINK
        packet.emit(hdr.report_ethernet);
        packet.emit(hdr.report_ipv4);
        packet.emit(hdr.report_udp);
        packet.emit(hdr.report_fixed_header);
#endif // WITH_INT_SINK
        ...
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
Based on what features are supported, we make sure to emit those back into the packet in the deparser stage.
<br/>
<br/>

[INT Header](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/fabric/src/main/resources/include/int/int_header.p4#L65)
```
#ifdef WITH_INT_SINK
header int_data_t {
    // Maximum int metadata stack size in bits:
    // (0xFF -4) * 32 (excluding INT shim header, tail header and INT header)
    varbit<8032> data;
}
#endif // WITH_INT_SINK
```
```
#ifdef WITH_INT_TRANSIT
// INT meta-value headers - 4 bytes each
// Different header for each value type
header int_switch_id_t {
    bit<32> switch_id;
}
header int_port_ids_t {
    bit<16> ingress_port_id;
    bit<16> egress_port_id;
}
header int_hop_latency_t {
    bit<32> hop_latency;
}
header int_q_occupancy_t {
    bit<8> q_id;
    bit<24> q_occupancy;
}
header int_ingress_tstamp_t {
    bit<32> ingress_tstamp;
}
header int_egress_tstamp_t {
    bit<32> egress_tstamp;
}
header int_q_congestion_t {
    bit<8> q_id;
    bit<24> q_congestion;
}
header int_egress_port_tx_util_t {
    bit<32> egress_port_tx_util;
}
#endif // WITH_INT_TRANSIT
```
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
```
In the file that defines all header "structs" related to INT, specific header structs are included on an as-need basis.
<br/>
<br/>

[INT_Main](https://github.com/breezestars/onos-barefoot/blob/master/pipelines/fabric/src/main/resources/include/int/int_main.p4#L21)
```
#ifdef WITH_INT_SOURCE
#include "int_source.p4"
#endif // WITH_INT_SOURCE

#ifdef WITH_INT_TRANSIT
#include "int_transit.p4"
#endif // WITH_INT_TRANSIT

#ifdef WITH_INT_SINK
#include "int_sink.p4"
#include "int_report.p4"
#endif // WITH_INT_SINK
```
Based on what flags are defined, the respective support files are added to the main file (the file that's included in other files when WITH_INT is defined).
<br>

```
control process_set_source_sink (
    inout parsed_headers_t hdr,
    inout fabric_metadata_t fabric_metadata,
    inout standard_metadata_t standard_metadata) {

    ...

#ifdef WITH_INT_SINK
    direct_counter(CounterType.packets_and_bytes) counter_set_sink;

    action int_set_sink () {
        fabric_metadata.int_meta.sink = _TRUE;
        counter_set_sink.count();
    }

    table tb_set_sink {
        key = {
            standard_metadata.egress_spec: exact @name("eg_spec");
        }
        actions = {
            int_set_sink;
            @defaultonly nop();
        }
        const default_action = nop();
        counters = counter_set_sink;
        size = MAX_PORTS;
    }
#endif // WITH_INT_SINK

    apply {
        tb_set_source.apply();

#ifdef WITH_INT_SINK
        tb_set_sink.apply();
        if(fabric_metadata.int_meta.sink == _TRUE) {
            // FIXME: this works only on BMv2
            #ifdef __TARGET_BMV2__
            clone(CloneType.I2E, REPORT_MIRROR_SESSION_ID);
            #endif
        }
#endif // WITH_INT_SINK
    }
}
```
Here we add a new table and corresponding action if the WITH_INT_SINK flag is defined, and to reach that code we add calls to that table in the apply block.
<br/>

```
control process_int_main (
    inout parsed_headers_t hdr,
    inout fabric_metadata_t fabric_metadata,
    inout standard_metadata_t standard_metadata) {

    apply {
        if (standard_metadata.ingress_port != CPU_PORT &&
            standard_metadata.egress_port != CPU_PORT &&
            (hdr.udp.isValid() || hdr.tcp.isValid())) {
#ifdef WITH_INT_SOURCE
            if (fabric_metadata.int_meta.source == _TRUE) {
                process_int_source.apply(hdr, fabric_metadata, standard_metadata);
            }
#endif // WITH_INT_SOURCE
            if(hdr.int_header.isValid()) {
#ifdef WITH_INT_TRANSIT
                process_int_transit.apply(hdr, fabric_metadata, standard_metadata);
#endif // WITH_INT_TRANSIT
#ifdef WITH_INT_SINK
                if (standard_metadata.instance_type == PKT_INSTANCE_TYPE_INGRESS_CLONE) {
                    /* send int report */
                    process_int_report.apply(hdr, fabric_metadata, standard_metadata);
                }
                if (fabric_metadata.int_meta.sink == _TRUE) {
                    // int sink
                    process_int_sink.apply(hdr, fabric_metadata);
                }
#endif // WITH_INT_SINK
            }
        }
    }
}
```
Here is a skeleton control block that will retain only the information relevant to the activated features. If none of those features (WITH_INT_SOURCE, WITH_INT_TRANSIT, WITH_INT_SINK) weren't defined/active, which isn't possible in this case since this file is included iff at least one of them is active, then this control block wouldn't much in it.