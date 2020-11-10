###### 1.1
Definition Source: https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/basic/src/main/resources/include/int_definitions.p4#L58
Macro Functional Definition: `IS_I2E_CLONE(smeta) (smeta.instance_type == BMV2_V1MODEL_INSTANCE_TYPE_INGRESS_CLONE)`
Usage Source: https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/basic/src/main/resources/int.p4#L72 
```
control egress (
    inout headers_t hdr,
    inout local_metadata_t local_metadata,
    inout standard_metadata_t standard_metadata) {

    apply {
        if(hdr.int_header.isValid()) {
            process_int_transit.apply(hdr, local_metadata, standard_metadata);

            #ifdef TARGET_BMV2
            if (IS_I2E_CLONE(standard_metadata)) {
                /* send int report */
                process_int_report.apply(hdr, local_metadata, standard_metadata);
            }

            if (local_metadata.int_meta.sink == _TRUE && !IS_I2E_CLONE(standard_metadata)) {
            #else
            if (local_metadata.int_meta.sink == _TRUE) {
            #endif // TARGET_BMV2
                process_int_sink.apply(hdr, local_metadata, standard_metadata);
             }
        }
        port_counters_egress.apply(hdr, standard_metadata);
        packetio_egress.apply(hdr, standard_metadata);
    }
}
```
If the TARGET_BMV2 flag is defined, they add a conditional block inside the apply block that checks if the standard_metadata's instance type is `BMV2_V1MODEL_INSTANCE_TYPE_INGRESS_CLONE` and [perform some actions](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/basic/src/main/resources/include/int_report.p4#L106) if it is.