Source: https://github.com/opennetworkinglab/onos/blob/master/pipelines/fabric/impl/src/main/resources/fabric.p4
```
#ifdef WITH_PORT_COUNTER
#include "include/control/port_counter.p4"
#endif // WITH_PORT_COUNTER
```
Using `ifdef` to include header files based on the flags
<br>
<br>

```
control FabricIngress (inout parsed_headers_t hdr,
                       inout fabric_metadata_t fabric_metadata,
                       inout standard_metadata_t standard_metadata) {

    PacketIoIngress() pkt_io_ingress;
    Filtering() filtering;
    Forwarding() forwarding;
    Acl() acl;
    Next() next;
#ifdef WITH_PORT_COUNTER
    PortCountersControl() port_counters_control;
#endif // WITH_PORT_COUNTER
    ...
```
Using `ifdef` to include control block variables
<br>
<br>

```
    apply {
        _PRE_INGRESS
        pkt_io_ingress.apply(hdr, fabric_metadata, standard_metadata);
#ifdef WITH_SPGW
        spgw_ingress.apply(hdr, fabric_metadata, standard_metadata);
#endif // WITH_SPGW
        ...
        }
    }
```
Using `ifdef` to add statements to the apply block
<br>
<br>