## Repo: Onos Satellite

If INT features are to be enabled, both `WITH_INT_SOURCE` and `WITH_INT_TRANSIT` have to be defined.


When trying to test P4 file (with `p4test`) with just `WITH_INT_SOURCE`, the following error is thrown:
```
include/control/../header.p4(244):syntax error, unexpected IDENTIFIER "int_switch_id_t"
    int_switch_id_t
    ^^^^^^^^^^^^^^^
[--Werror=overlimit] error: 1 errors encountered, aborting compilation
```
`header.p4`:
```
#include "int/int_header.p4"
...
struct parsed_headers_t {
    ...
#ifdef WITH_INT
    // INT specific headers
    intl4_shim_t intl4_shim;
    int_header_t int_header;
    int_switch_id_t int_switch_id; //Line 244
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
`int/int_header.p4` defining `int_switch_id_t` header type:
```
#ifdef WITH_INT_TRANSIT
// INT meta-value headers - 4 bytes each
// Different header for each value type
header int_switch_id_t {
    bit<32> switch_id;
}
...
#endif
```
If either `WITH_INT_SOURCE` or `WITH_INT_TRANSIT` is present, `WITH_INT` is defined automatically. So, `header.p4` tries to declare a struct variable of a type that's defined iff is declared `WITH_INT_TRANSIT`. So, `WITH_INT_SOURCE` and `WITH_INT_TRANSIT` have to be used together


Note: No errors occur if just `WITH_INT_TRANSIT` is passed in as an argument

***

Unable to compile with just `WITH_BNG` or just `WITHOUT_XCONNECT` flags. Throws the following error
```
include/bng.p4(345): [--Werror=type-error] error: Field inner_vlan_id is not a member of structure struct fabric_metadata_t
                c_tag = fmeta.inner_vlan_id;
                              ^^^^^^^^^^^^^
include/control/../header.p4(168)
struct fabric_metadata_t {
       ^^^^^^^^^^^^^^^^^
```

Needs the `WITH_DOUBLE_VLAN_TERMINATION` flag to include that member in the fabric_metadata_t structure