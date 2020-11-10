###### 1.1
Source: https://github.com/breezestars/onos-barefoot/blob/440d67b3a97363ed9b15894162c4c8bac7ebff05/pipelines/fabric/src/main/resources/include/header.p4#L152
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
Here they add new typedef and constant declarations along with a new struct if the BNG feature is defined.
Similar code found at: 
* https://github.com/ayman014/onos-mosaic/blob/f8e15abe44b3ec5d702e67287f4acdbde3a473a8/pipelines/fabric/impl/src/main/resources/include/header.p4#L157
* https://github.com/epicLevi/onos-ai-docker/blob/e2aefa79fbe7464671bbfd5e4e3b2ffa609bc099/pipelines/fabric/impl/src/main/resources/include/header.p4#L157
###### 2.1
Source: https://github.com/andreyqg/p4c/blob/67666e19f4e090e16eaf55cb4307c61676ab8edf/p4include/psa.p4#L28 (The specific files seems to be derived from a Barefoot Network repo)
```
/**
 * These types need to be defined before including the architecture file
 * and the macro protecting them should be defined.
 */
#define PSA_ON_BMV2_CORE_TYPES
#ifdef PSA_ON_BMV2_CORE_TYPES
/* The bit widths shown below are specific to the BMv2 psa_switch
 * target.  These types do _not_ dictate what sizes these types should
 * have for any other implementation of PSA.  Each PSA implementation
 * is free to use its own custom width in bits for those types that
 * are bit<W> for some W.  One reason they are here is to support the
 * implementation of PSA on BMv2.  Another is so that we can easily
 * compile this file, and example PSA P4 programs that include it.
 *
 * The bit widths for BMv2 psa_switch have been chosen to be the same
 * as the corresponding InHeader types later.  This simplifies the
 * implementation of P4Runtime for BMv2 psa_switch. */

/* These are defined using `typedef`, not `type`, so they are truly
 * just different names for the type bit<W> for the particular width W
 * shown.  Unlike the `type` definitions below, values declared with
 * the `typedef` type names can be freely mingled in expressions, just
 * as any value declared with type bit<W> can.  Values declared with
 * one of the `type` names below _cannot_ be so freely mingled, unless
 * you first cast them to the corresponding `typedef` type.  While
 * that may be inconvenient when you need to do arithmetic on such
 * values, it is the price to pay for having all occurrences of values
 * of the `type` types marked as such in the automatically generated
 * control plane API.
 *
 * Note that the width of typedef <name>Uint_t will always be the same
 * as the width of type <name>_t. */
typedef bit<32> PortIdUint_t;
typedef bit<32> MulticastGroupUint_t;
typedef bit<16> CloneSessionIdUint_t;
typedef bit<8>  ClassOfServiceUint_t;
typedef bit<16> PacketLengthUint_t;
typedef bit<16> EgressInstanceUint_t;
typedef bit<64> TimestampUint_t;

/* Note: clone_spec in BMv2 simple_switch v1model is 32 bits wide, but
 * it is used such that 16 of its bits contain a clone/mirror session
 * id, and 16 bits contain the numeric id of a field_list.  Only the
 * 16 bits of clone/mirror session id are comparable to the type
 * CloneSessionIdUint_t here.  See occurrences of clone_spec in this
 * file for details:
 * https://github.com/p4lang/behavioral-model/blob/master/targets/simple_switch/simple_switch.cpp
 */

@p4runtime_translation("p4.org/psa/v1/PortId_t", 32)
type PortIdUint_t         PortId_t;
@p4runtime_translation("p4.org/psa/v1/MulticastGroup_t", 32)
type MulticastGroupUint_t MulticastGroup_t;
@p4runtime_translation("p4.org/psa/v1/CloneSessionId_t", 16)
type CloneSessionIdUint_t CloneSessionId_t;
@p4runtime_translation("p4.org/psa/v1/ClassOfService_t", 8)
type ClassOfServiceUint_t ClassOfService_t;
@p4runtime_translation("p4.org/psa/v1/PacketLength_t", 16)
type PacketLengthUint_t   PacketLength_t;
@p4runtime_translation("p4.org/psa/v1/EgressInstance_t", 16)
type EgressInstanceUint_t EgressInstance_t;
@p4runtime_translation("p4.org/psa/v1/Timestamp_t", 64)
type TimestampUint_t      Timestamp_t;
typedef error   ParserError_t;

const PortId_t PSA_PORT_RECIRCULATE = (PortId_t) 0xfffffffa;
const PortId_t PSA_PORT_CPU = (PortId_t) 0xfffffffd;

const CloneSessionId_t PSA_CLONE_SESSION_TO_CPU = (CloneSessionId_t) 0;

#endif  // PSA_ON_BMV2_CORE_TYPES
```
In this block of code we can observe that a list of typedefs and a list of type declarations based on those newly declared typedef variables are declared based on a flag. On top of that, `p4runtime_translation` is used to tell the compiler that some of those type declarations should be used rather than the target-specific bit width.

Similar code found at:
* https://github.com/cornell-netlab/petr4/blob/fd48ab12d4fb7855a4f6c22dd19e24eaa7e919bb/examples/psa.p4#L28

###### 3.1
Source: https://github.com/cornell-netlab/MicroP4/blob/02cba4794064115d16519103b41c00865c2af062/p4include/psa.p4#L96
```
#ifndef __PSA_P4__
#define __PSA_P4__

#include<core.p4>

#ifndef _PORTABLE_SWITCH_ARCHITECTURE_P4_
#define _PORTABLE_SWITCH_ARCHITECTURE_P4_

...
#define PSA_CORE_TYPES
#ifdef PSA_CORE_TYPES
/* The bit widths shown below are only examples.  Each PSA
 * implementation is free to use its own custom width in bits for
 * those types that are bit<W> for some W.  The only reason that there
 * are example numerical widths in this file is so that we can easily
 * compile this file, and example PSA P4 programs that include it. */

typedef bit<10> PortId_t;
typedef bit<10> MulticastGroup_t;
typedef bit<10> CloneSessionId_t;
typedef bit<3>  ClassOfService_t;
typedef bit<14> PacketLength_t;
typedef bit<16> EgressInstance_t;
typedef bit<48> Timestamp_t;
typedef error   ParserError_t;

const   PortId_t         PSA_PORT_RECIRCULATE = 254;
const   PortId_t         PSA_PORT_CPU = 255;

const   CloneSessionId_t PSA_CLONE_SESSION_TO_CPU = 0;
#endif  // PSA_CORE_TYPES
...
#endif  /* _PORTABLE_SWITCH_ARCHITECTURE_P4_ */

#endif   // __PSA_P4__   
```
Similar to the last example, here we can see that typedef and const variables are declared iff the PSA_CORE_TYPES flag is defined