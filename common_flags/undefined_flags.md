#####  WITH_SPGW
* Gets defined in one of the profiles in [Makefile](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/Makefile#L7)
#####  WITHOUT_XCONNECT
* Gets defined in one of the profiles in [Makefile](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/Makefile#L10)
#####  WITH_BNG
* Gets defined in one of the profiles in [Makefile](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/Makefile#L10)
#####  WITH_DOUBLE_VLAN_TERMINATION
* Gets defined in one of the profiles in [Makefile](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/Makefile#L10)
#####  WITH_INT_SOURCE
* Gets defined in one of the profiles in [Makefile](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/Makefile#L13)
#####  WITH_IPV6
* Gets defined in one of the profiles in [Makefile](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/Makefile#L19)
#####  WITH_SIMPLE_NEXT
* Gets defined in one of the profiles in [Makefile](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/Makefile#L20)
#####  WITH_INT_SINK
* Gets defined in one of the profiles in [Makefile](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/Makefile#L21)
#####  WITH_INT_TRANSIT
* Gets defined in one of the profiles in [Makefile](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/Makefile#L21)
#####  WITH_INT
* Gets defined at [define.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/define.p4#L23) if either WITH_INT_SOURCE, WITH_INT_TRANSIT, or WITH_INT_SINK is defined
#####  WITH_PORT_COUNTER
* Gets defined at [bmv2-compile.sh](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/bmv2-compile.sh#L6)
#####  IP_VERSION_6
* Always gets defined with a value of 6 in [define.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/define.p4#L63)
    * If WITH_IPV6 flag is defined, this macro's value is to say which version of IPV6 is being used at [parser.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/parser.p4#L119)
#####  IP_VERSION_4
* Always gets defined with a value of 4 in [define.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/define.p4#L60)
    * This macro's value is to say which version of IPV4 is being used at [parser.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/parser.p4#L117)
        * Also at [spgw.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/spgw.p4#L192)
* Also gets defined as `4w4` in the basic pipeline implementation ([defines.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/basic/src/main/resources/include/defines.p4#L23))
    * Gets used at [int_report.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/basic/src/main/resources/include/int_report.p4#L62) to set the header's report_ipv4's version
#####  _TRUE
* A macro variable to represent true value (it is assigned as `true`). Defined at [define.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/basic/src/main/resources/include/defines.p4#L31)
    * Also defined in the fabric pipeline
#####  _BOOL
* A macro variable to represent bool value. Defined at [define.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/define.p4#L35)
    * Also defined in the basic pipeline
#####  _FALSE
* A macro variable to represent true value (it is assigned as `true`). Defined at [define.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/basic/src/main/resources/include/defines.p4#L34)
    * Also defined in the fabric pipeline
#####  _INT_METADATA_ACTIONS
* Not defined in any other file
* Used at [int_transit.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/int/int_transit.p4#L35) file
    * If this macro is defined, it replaces the macro with its value
    * Else it inserts a bunch of actions inside the control block to set header values of switch IDs, port IDs, hop latency, queue occupancy, ingress and egress timestamp, queue congestion, and egress port utilization
#####  _INT_INIT_METADATA
* Not defined in any other file
* Used at [int_transit.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/int/int_transit.p4#L29) file
    * Comment says the macro is used to [`Allow other targets to initialize INT metadata in their own way.`](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/int/int_transit.p4#L28)
    * If not defined, sets metadata's switch ID to switch ID passed into the action
#####  _PRE_INGRESS
* Gets defined at [define.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/define.p4#L49) and has no value associated with it
* Get used in [fabric.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/fabric.p4#L60) where the macro is replaced with its value (which is nothing...?)
#####  _ROUTING_V4_TABLE_ANNOT
* Defined at [forwarding.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/control/forwarding.p4#L99)
* Not used anywhere else
#####  __TARGET_BMV2__
* Used at [int_main.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/int/int_main.p4#L88)
* Not defined any where
#####  _PRE_EGRESS
* Defined at [define.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/define.p4#L53) and gets no value assigned to it
* Used at [fabric.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/fabric.p4#L101)
#####  _PKT_OUT_HDR_ANNOT
* Defined at [define.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/define.p4#L45) and gets no value assigned to it
* Used at [header.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/header.p4#L29)
#####  WITH_SPGW_UDP_CSUM_UPDATE
* Used at [spgw.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/spgw.p4#L262)
* Not defined anywhere
#####  WITH_SPGW_PCC_GATING
* Not defined anywhere
* Used at [header.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/header.p4#L142) and [spgw.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/spgw.p4#L94)
#####  IP_VER_LENGTH
* Gets defined with a value of 4 at [define.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/define.p4#L57)
* Used at [parser.p4](https://github.com/wyan-all/onos-satellite/blob/e38afb37544085321295cf6fc813ddc3101789ee/pipelines/fabric/src/main/resources/include/parser.p4) inside a transition select argument list

***

Vomci

##### \_PKT_OUT_HDR_ANNOT\_
* Gets defined at [define.p4](https://github.com/breezestars/vomci-onos/blob/848a7c61cddcce0db50877af5b6af1eae3edc22e/pipelines/fabric/src/main/resources/include/define.p4#L57) if it isn't already defined
* Used at [header.p4](https://github.com/breezestars/vomci-onos/blob/848a7c61cddcce0db50877af5b6af1eae3edc22e/pipelines/fabric/src/main/resources/include/header.p4#L28)
    * The macro gets replaced with its value (which is nothing if user hasn't defined it)