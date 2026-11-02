# Photoshop network-change crash and a local mutex reentrancy experiment

Target discussion:
https://community.adobe.com/bug-reports-711/photoshop-crashes-on-pc-wake-1641867

I have additional observations from Photoshop 27.8.0.13 on Windows 26H2, OS build 26340.9843, using an ASIX USB Ethernet adapter. A controlled network adapter reconnect reproduced a C++ exception involving netprofm, GetConnectivity and reentrant COM/RPC calls.

In my dump, the local _Throw_Cpp_error argument was 5, corresponding to RESOURCE_DEADLOCK_WOULD_OCCUR. This is a different argument from the value described in the original report.

The stack and disassembly indicate that Photoshop queries Network List Manager while holding a non-recursive mutex in uxtech::network::ReachabilityImpl. COM STA reentrancy enters the same network query before the outer call completes, and the second lock attempt throws. The dump does not contain the relevant heap bytes, so runtime mutex-owner fields were not directly inspected.

Turning off the Home Screen and reducing NIC power transitions did not eliminate reproduction. After backing up and checking the exact executable, a local, single-byte compatibility experiment changed only this object's mutex initialization flag from 0x2 to 0x102, permitting same-thread recursion while retaining normal lock/unlock operations.

Results on this exact binary:

- Five consecutive adapter reconnects passed in one session.
- A sixth reconnect after restarting Photoshop passed.
- Photoshop remained responsive and the unsaved fixture modification survived.
- AutoRecover, a fixture-only forced-exit recovery, and normal application shutdown also passed after the change.

This is an exact-binary experiment on an installation that already had local modifications, not a general patch validated across official builds. Real sleep/resume and long-duration idle behavior were not retested, and other potential reentrancy consequences remain outside these short tests. The result seems worth investigating as a locking/reentrancy defect in the connectivity callback path.

Microsoft STL references used to interpret the mutex flags and error value:
https://raw.githubusercontent.com/microsoft/STL/main/stl/src/mutex.cpp
https://raw.githubusercontent.com/microsoft/STL/main/stl/inc/xthreads.h
