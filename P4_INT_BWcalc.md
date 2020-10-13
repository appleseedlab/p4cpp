Note: This is student created repo
Source: https://github.com/Brew8it/p4_INT_BWcalc/blob/5de3cdf79c32fd06070e1ca1f6749ddc8f8208d0/p4src/includes/p4features.h
```
...
#define PIM_BIDIR_OPTIMIZATION
#define SFLOW_ENABLE
#define EGRESS_ACL_ENABLE

#ifdef MULTICAST_DISABLE
#define L2_MULTICAST_DISABLE
#define L3_MULTICAST_DISABLE
#endif

// Defines for switchapi library
#ifdef URPF_DISABLE
#define P4_URPF_DISABLE
#endif
...
```
They use a file to define flags that indicate whether a feature is supported or not. They also check if a specific feature is disabled, and if it is, they also disable related/child features as shown above

***

Source: https://github.com/Brew8it/p4_INT_BWcalc/blob/5de3cdf79c32fd06070e1ca1f6749ddc8f8208d0/p4src/includes/defines.p4
```
/* Boolean */
#define FALSE                                  0
#define TRUE                                   1

/* Packet types */
#define L2_UNICAST                             1
#define L2_MULTICAST                           2
#define L2_BROADCAST                           4
...
```
Using a file just to have all global defines/macros. Looks like they defined FALSE AND TRUE as 0 and 1 to act as boolean values (but looks like P4<sub>16</sub> supports true and boolean literals)

***

Source: https://github.com/Brew8it/p4_INT_BWcalc/blob/5de3cdf79c32fd06070e1ca1f6749ddc8f8208d0/p4factory-master/targets/l2_switch/p4src/l2_switch.p4
```
//uncomment to enable openflow
//#define OPENFLOW_ENABLE

#ifdef OPENFLOW_ENABLE
    #include "openflow.p4"
#endif /* OPENFLOW_ENABLE */
```
Using `ifdef` to check if openflow is enabled, and if so, `include` openflow file
<br>
<br>

```
parser start {
    return parse_ethernet;
}

...
parser parse_ethernet {
    extract(ethernet);
#ifdef OPENFLOW_ENABLE
    return select(latest.etherType) {
        ETHERTYPE_BF_FABRIC : parse_fabric_header;
        default : ingress;
    }
#else
    return ingress;
#endif /* OPENFLOW_ENABLE */
}
```
Using `ifdef` to see if Openflow is enabled, and if so, it inserts a select statement inside to parse a specific header, else defaults to ingress control block
```
control ingress {
#ifdef OPENFLOW_ENABLE
    apply(packet_out) {
        nop {
#endif /* OPENFLOW_ENABLE */
            apply(smac);
            apply(dmac);
#ifdef OPENFLOW_ENABLE
        }
    }

    process_ofpat_ingress ();
#endif /* OPENFLOW_ENABLE */
}
```
And in the ingress control they again check if Openflow is enabled and insert new statements to the apple block depending on that
