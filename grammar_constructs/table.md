###### 1.1
Source: https://github.com/wyan-all/onos-satellite/blob/master/pipelines/fabric/src/main/resources/include/control/next.p4
```
control Next (inout parsed_headers_t hdr,
              inout fabric_metadata_t fabric_metadata,
              inout standard_metadata_t standard_metadata) {
    ...
    table next_vlan {
        key = {
            fabric_metadata.next_id: exact @name("next_id");
        }
        actions = {
            set_vlan;
#ifdef WITH_DOUBLE_VLAN_TERMINATION
            set_double_vlan;
#endif // WITH_DOUBLE_VLAN_TERMINATION
            @defaultonly nop;
        }
        const default_action = nop();
        counters = next_vlan_counter;
        size = NEXT_VLAN_TABLE_SIZE;
    }
...
}
```
Here they are adding new actionList values to the table's actions block based on the supported features