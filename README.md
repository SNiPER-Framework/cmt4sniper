# CMT Wrapper for SNiPER v2

Recently we dropped CMT and moved to CMake in SNiPER. We will released SNiPER v2 soon. It's a big step to people who are still using CMT to build their own projects. This CMT wrapper is privided in the hope of making life easier.

## Usages

Suppose SNiPER has been built with CMake. We can build our own CMT project (such as JUNO offline) as before simply in 2 steps:

1. Put this wrapper to a `${CMTPROJECTPATH}` before building your own CMT project.
2. In file `${YOUR_PROJECT}/cmt/project.cmt`, replace

    ```use sniper```

    with

    ```use cmt4sniper```

That's it!

## Porting Code to SNiPER v2

I tried to keep the software interface be same as before. However, there are a few changes indeed. With a complete building of the JUNO offline software, I found the following incompatibilities:

1. The most obvious change is:
    ```c++
    //DataBuffer, a single file package, is merged into SniperKernel
    //#include "DataBuffer/DataBuffer.h"
    #include "SniperKernel/DataBuffer.h"
    ```
   Since most users do not use this header directly, the influence is almost negligible. In JUNO offline, only `EvtNavigator/NavBuffer.h` (YES, ONLY ONE FILE) should be modified according to this change.
2. The machanism of Property is changed internally, but the Python interface is not changed. In general, users will not notice the change. An exception is found in JUNO offline, where a TimeStamp is declared as a Property. The TimeStamp is originally taken from TTimeStamp in ROOT. There is an `operator<<` for TimeStamp, but the equivalent `operator>>` is missing. An error will raise while compiling in similar situations. I believe this is a rare case. And it is easy to fix by adding the missing operator(s):
    ```c++
    //this one already exist
    std::ostream& operator<<(std::ostream& os, const TimeStamp& vldts);
    //we can add the missing one
    std::istream& operator>>(std::istream& is, const TimeStamp& vldts);
    ```
3. In v2, a Property value is dumped to a JSON string during the data delivery between Python and C++. In fact it is better than v1. We can delivery any combination of `vector` and `map`. For example, a Property of `vector<vector>` in C++ can be assigned by `[[],...]` in Python now. It's hard to achieve this in v1. Such an example is also found in JUNO offline. In order to assigne a `vector<vector>` Property in v1, a customized `ThreeVector` is exported and `[ThreeVector,...]` is delivered in Python. Unfortunately the customized `ThreeVector` can not be dumped to JSON, and a Runtime Error is raised in v2. The solution is quite simple, use `[[],...]` directly.

Then I can build the entire JUNO offline software, and run simulation jobs successfully based on SNiPER v2. We can see the modifications are very few during code porting. Of cause a full validation should be done next.

## One More Big Change in v2

Previous contents are based on this commit [e98267c](https://github.com/SNiPER-Framework/sniper/commit/e98267c8b1b373db312588a62f1b4c9cc97faabe). Possibly we will introduce another big change before frozen v2.0. In code view:

```c++
//class
class Task : public DLElement {};

//and
Task* DLElement::getParent();
Task* DLElement::getRoot();
```

will be updated to:

```c++
//add a new class between the inheritance of Task and DLElement
class IExecUnit : public DLElement {};
class Task : public IExecUnit {};

//and
IExecUnit* DLElement::getParent();
IExecUnit* DLElement::getRoot();
```

In JUNO offline, 28 C++ files should be updated according to this change. Just simplely replace keyword Task by IExecUnit.
