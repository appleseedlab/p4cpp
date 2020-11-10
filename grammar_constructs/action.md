###### 1.1
Source: https://github.com/gregorypaskar/p4/blob/c50a6bec93d6ffbb249dd0a0d67a3ddf64ef0b8d/testdata/p4_16_samples/fabric_20190420/include/int/int_transit.p4#L20
```
control process_int_transit (
    inout parsed_headers_t hdr,
    inout fabric_metadata_t fmeta,
    inout standard_metadata_t smeta) {

    action init_metadata(bit<32> switch_id) {
        fmeta.int_meta.transit = _TRUE;
#ifdef _INT_INIT_METADATA
        // Allow other targets to initialize INT metadata in their own way.
        _INT_INIT_METADATA
#else
        fmeta.int_meta.switch_id = switch_id;
#endif // _INT_INIT_METADATA
    }
    ...
}
```
They are adding new blockStatement values inside the action block based on the flag