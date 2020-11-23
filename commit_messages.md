https://github.com/wyan-all/onos-satellite/commit/70e816b05ad7a592178f85221149b1855bc4f9ed
commit 70e816b05ad7a592178f85221149b1855bc4f9ed
Author: Carmelo Cascone <carmelo@opennetworking.org>
Date:   Tue Mar 19 16:15:47 2019 -0700

    Support compiling fabric.p4 with arbitrary table sizes
    
    By using preprocessor macros. Also, change expected location of tofino
    compiler outputs when building the pipeconf.
    
    Change-Id: I98ea95b61d57e725c88e52a3bfd95618f3c407cb

Main changes in this commit are changing all constant size values to a macro label that's defined in the size.p4 file (the size.p4 file was newly added in this commit)

https://github.com/gregorypaskar/p4/commit/72b4e9200755c5bc8d197b28bbd4561eb0e7d30d
commit 72b4e9200755c5bc8d197b28bbd4561eb0e7d30d
Author: Han Wang <hanw@users.noreply.github.com>
Date:   Tue Apr 17 10:47:19 2018 -0700

    remove macro for PSA_SWITCH in psa.p4, update testcase (#1231)

Looks like they removed a macro and substituted it with a function (issue number #1231?)

https://github.com/gregorypaskar/p4/commit/c02c4e85b7566c88a1cc0084c5d6a1f1e8adb7c3
commit c02c4e85b7566c88a1cc0084c5d6a1f1e8adb7c3
Author: Chris Dodd <chris@barefootnetworks.com>
Date:   Fri Aug 12 17:24:27 2016 -0700

    workaround for setbit/clrbit macros in some system headers
Maybe related to P4 - might be an issue
"some broken systems define a `setbit' macro in their system header files!" - so they check to see if 'setbit' is defined, and if so, they undefine it (similar for 'clrbit' flag)