Source: https://github.com/flexmesh/FlexMesh/blob/c87a999f6a33c393f17734b9256288a95706756f/src/flex4.p4
```
#if INGRESS_MODULE_NUM > 14
#ifdef INGRESS_MODULE_14
    INGRESS_CHECK(0x4000) {
        INGRESS_MODULE_14;
    }
#endif
#endif

...

#if EGRESS_MODULE_NUM > 2
#ifdef ENGRESS_MODULE_2
    EGRESS_CHECK(0x2) {
        ENGRESS_MODULE_2;
    }
#endif
#endif
```
Using `#if` statements to check how many ingress and egress modules are present. And then uses `#ifdef` to check if the corresponding flag is set, and if so, declares a new ingress or egress check function using the predfined macro function

***

Source: https://github.com/flexmesh/FlexMesh/blob/c87a999f6a33c393f17734b9256288a95706756f/src/core/macro.p4

```
#define FLEX_EGRESS_BITMAP  flex_metadata.egress_bitmap
#define INGRESS_CHECK(X) if(( FLEX_INGRESS_BITMAP & (X)) == 0)
#define EGRESS_CHECK(X) if(( FLEX_EGRESS_BITMAP & (X)) == 0)
```
Definition for INGRESS_CHECK and EGRESS_CHECK
`flex_metadata` is a custom defined metadata for the program
