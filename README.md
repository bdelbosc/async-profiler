# async-profiler

This project is a low overhead sampling profiler for Java
that does not suffer from [Safepoint bias problem](http://psy-lob-saw.blogspot.ru/2016/02/why-most-sampling-java-profilers-are.html).
It features HotSpot-specific APIs to collect stack traces
and to track memory allocations. The profiler works with
OpenJDK, Oracle JDK and other Java runtimes based on the HotSpot JVM.

async-profiler can trace the following kinds of events:
 - CPU cycles
 - Hardware and Software performance counters like cache misses, branch misses, page faults, context switches etc.
 - Allocations in Java Heap
 - Contented lock attempts, including both Java object monitors and ReentrantLocks

See our [Wiki](https://github.com/jvm-profiling-tools/async-profiler/wiki) or
[3 hours playlist](https://www.youtube.com/playlist?list=PLNCLTEx3B8h4Yo_WvKWdLvI9mj1XpTKBr)
to learn about all features. 

## Download

Current release (2.8.3):

 - Linux x64 (glibc): [async-profiler-2.8.3-linux-x64.tar.gz](https://github.com/jvm-profiling-tools/async-profiler/releases/download/v2.8.3/async-profiler-2.8.3-linux-x64.tar.gz)
 - Linux x64 (musl): [async-profiler-2.8.3-linux-musl-x64.tar.gz](https://github.com/jvm-profiling-tools/async-profiler/releases/download/v2.8.3/async-profiler-2.8.3-linux-musl-x64.tar.gz)
 - Linux arm64: [async-profiler-2.8.3-linux-arm64.tar.gz](https://github.com/jvm-profiling-tools/async-profiler/releases/download/v2.8.3/async-profiler-2.8.3-linux-arm64.tar.gz)
 - macOS x64/arm64: [async-profiler-2.8.3-macos.zip](https://github.com/jvm-profiling-tools/async-profiler/releases/download/v2.8.3/async-profiler-2.8.3-macos.zip)
 - Converters between profile formats: [converter.jar](https://github.com/jvm-profiling-tools/async-profiler/releases/download/v2.8.3/converter.jar)  
   (JFR to Flame Graph, JFR to FlameScope, collapsed stacks to Flame Graph)

[Previous releases](https://github.com/jvm-profiling-tools/async-profiler/releases)

Note: async-profiler also comes bundled with IntelliJ IDEA Ultimate 2018.3 and later.
For more information refer to [IntelliJ IDEA documentation](https://www.jetbrains.com/help/idea/cpu-and-allocation-profiling-basic-concepts.html).

## Supported platforms

 - **Linux** / x64 / x86 / arm64 / arm32 / ppc64le
 - **macOS** / x64 / arm64

### Community supported builds

 - **Windows** / x64 - <img src="https://upload.wikimedia.org/wikipedia/commons/9/9c/IntelliJ_IDEA_Icon.svg" width="16" height="16"/> [IntelliJ IDEA](https://www.jetbrains.com/idea/) 2021.2 and later

## CPU profiling

In this mode profiler collects stack trace samples that include **Java** methods,
**native** calls, **JVM** code and **kernel** functions.

The general approach is receiving call stacks generated by `perf_events`
and matching them up with call stacks generated by `AsyncGetCallTrace`,
in order to produce an accurate profile of both Java and native code.
Additionally, async-profiler provides a workaround to recover stack traces
in some [corner cases](https://bugs.openjdk.java.net/browse/JDK-8178287)
where `AsyncGetCallTrace` fails.

This approach has the following advantages compared to using `perf_events`
directly with a Java agent that translates addresses to Java method names:

* Works on older Java versions because it doesn't require
  `-XX:+PreserveFramePointer`, which is only available in JDK 8u60 and later.

* Does not introduce the performance overhead from `-XX:+PreserveFramePointer`,
  which can in rare cases be as high as 10%.

* Does not require generating a map file to map Java code addresses to method
  names.

* Works with interpreter frames.

* Does not require writing out a perf.data file for further processing in
  user space scripts.

If you wish to resolve frames within `libjvm`, the [debug symbols](#installing-debug-symbols) are required.

## ALLOCATION profiling

Instead of detecting CPU-consuming code, the profiler can be configured
to collect call sites where the largest amount of heap memory is allocated.

async-profiler does not use intrusive techniques like bytecode instrumentation
or expensive DTrace probes which have significant performance impact.
It also does not affect Escape Analysis or prevent from JIT optimizations
like allocation elimination. Only actual heap allocations are measured.

The profiler features TLAB-driven sampling. It relies on HotSpot-specific
callbacks to receive two kinds of notifications:
 - when an object is allocated in a newly created TLAB (aqua frames in a Flame Graph);
 - when an object is allocated on a slow path outside TLAB (brown frames).

This means not each allocation is counted, but only allocations every _N_ kB,
where _N_ is the average size of TLAB. This makes heap sampling very cheap
and suitable for production. On the other hand, the collected data
may be incomplete, though in practice it will often reflect the top allocation
sources.

Sampling interval can be adjusted with `--alloc` option.
For example, `--alloc 500k` will take one sample after 500 KB of allocated
space on average. However, intervals less than TLAB size will not take effect.

The minimum supported JDK version is 7u40 where the TLAB callbacks appeared.

### Installing Debug Symbols

The allocation profiler requires HotSpot debug symbols. Oracle JDK already has them
embedded in `libjvm.so`, but in OpenJDK builds they are typically shipped
in a separate package. For example, to install OpenJDK debug symbols on
Debian / Ubuntu, run:
```
# apt install openjdk-8-dbg
```
or for OpenJDK 11:
```
# apt install openjdk-11-dbg
```

On CentOS, RHEL and some other RPM-based distributions, this could be done with
[debuginfo-install](http://man7.org/linux/man-pages/man1/debuginfo-install.1.html) utility:
```
# debuginfo-install java-1.8.0-openjdk
```

On Gentoo the `icedtea` OpenJDK package can be built with the per-package setting
`FEATURES="nostrip"` to retain symbols.

The `gdb` tool can be used to verify if the debug symbols are properly installed for the `libjvm` library.
For example on Linux:
```
$ gdb $JAVA_HOME/lib/server/libjvm.so -ex 'info address UseG1GC'
```
This command's output will either contain `Symbol "UseG1GC" is at 0xxxxx`
or `No symbol "UseG1GC" in current context`.

## Wall-clock profiling

`-e wall` option tells async-profiler to sample all threads equally every given
period of time regardless of thread status: Running, Sleeping or Blocked.
For instance, this can be helpful when profiling application start-up time.

Wall-clock profiler is most useful in per-thread mode: `-t`.

Example: `./profiler.sh -e wall -t -i 5ms -f result.html 8983`

## Java method profiling

`-e ClassName.methodName` option instruments the given Java method
in order to record all invocations of this method with the stack traces.

Example: `-e java.util.Properties.getProperty` will profile all places
where `getProperty` method is called from.

Only non-native Java methods are supported. To profile a native method,
use hardware breakpoint event instead, e.g. `-e Java_java_lang_Throwable_fillInStackTrace`

**Be aware** that if you attach async-profiler at runtime, the first instrumentation
of a non-native Java method may cause the [deoptimization](https://github.com/openjdk/jdk/blob/bf2e9ee9d321ed289466b2410f12ad10504d01a2/src/hotspot/share/prims/jvmtiRedefineClasses.cpp#L4092-L4096)
of all compiled methods. The subsequent instrumentation flushes only the _dependent code_.

The massive CodeCache flush doesn't occur if attaching async-profiler as an agent.

Here are some useful native methods that you may want to profile:
* ```G1CollectedHeap::humongous_obj_allocate``` - trace the _humongous allocation_ of the G1 GC,
* ```JVM_StartThread``` - trace the new thread creation,
* ```Java_java_lang_ClassLoader_defineClass1``` - trace class loading.

## Building

Build status: [![Build Status](https://github.com/jvm-profiling-tools/async-profiler/actions/workflows/cpp.yml/badge.svg?branch=master)](https://github.com/jvm-profiling-tools/async-profiler/actions/workflows/cpp.yml)

Make sure the `JAVA_HOME` environment variable points to your JDK installation,
and then run `make`. GCC is required. After building, the profiler agent binary
will be in the `build` subdirectory. Additionally, a small application `jattach`
that can load the agent into the target process will also be compiled to the
`build` subdirectory. If the build fails due to
`Source option 7 is no longer supported. Use 8 or later.`, use `make JAVA_TARGET=8`.

## Basic Usage

As of Linux 4.6, capturing kernel call stacks using `perf_events` from a non-root
process requires setting two runtime variables. You can set them using
sysctl or as follows:

```
# sysctl kernel.perf_event_paranoid=1
# sysctl kernel.kptr_restrict=0
```

To run the agent and pass commands to it, the helper script `profiler.sh`
is provided. A typical workflow would be to launch your Java application,
attach the agent and start profiling, exercise your performance scenario, and
then stop profiling. The agent's output, including the profiling results, will
be displayed in the Java application's standard output.

Example:

```
$ jps
9234 Jps
8983 Computey
$ ./profiler.sh start 8983
$ ./profiler.sh stop 8983
```

The following may be used in lieu of the `pid` (8983):

 - The keyword `jps`, which will use the most recently launched Java process.
 - The application name as it appears in the `jps` output: e.g. `Computey`

Alternatively, you may specify `-d` (duration) argument to profile
the application for a fixed period of time with a single command.

```
$ ./profiler.sh -d 30 8983
```

By default, the profiling frequency is 100Hz (every 10ms of CPU time).
Here is a sample of the output printed to the Java application's terminal:

```
--- Execution profile ---
Total samples:           687
Unknown (native):        1 (0.15%)

--- 6790000000 (98.84%) ns, 679 samples
  [ 0] Primes.isPrime
  [ 1] Primes.primesThread
  [ 2] Primes.access$000
  [ 3] Primes$1.run
  [ 4] java.lang.Thread.run

... a lot of output omitted for brevity ...

          ns  percent  samples  top
  ----------  -------  -------  ---
  6790000000   98.84%      679  Primes.isPrime
    40000000    0.58%        4  __do_softirq

... more output omitted ...
```

This indicates that the hottest method was `Primes.isPrime`, and the hottest
call stack leading to it comes from `Primes.primesThread`.

## Launching as an Agent

If you need to profile some code as soon as the JVM starts up, instead of using the `profiler.sh` script,
it is possible to attach async-profiler as an agent on the command line. For example:

```
$ java -agentpath:/path/to/libasyncProfiler.so=start,event=cpu,file=profile.html ...
```

Agent library is configured through the JVMTI argument interface.
The format of the arguments string is described
[in the source code](https://github.com/jvm-profiling-tools/async-profiler/blob/v2.8.3/src/arguments.cpp#L52).
The `profiler.sh` script actually converts command line arguments to that format.

For instance, `-e wall` is converted to `event=wall`, `-f profile.html`
is converted to `file=profile.html`, and so on. However, some arguments are processed
directly by `profiler.sh` script. E.g. `-d 5` results in 3 actions:
attaching profiler agent with start command, sleeping for 5 seconds,
and then attaching the agent again with stop command.

## Multiple events

It is possible to profile CPU, allocations, and locks at the same time.
Or, instead of CPU, you may choose any other execution event: wall-clock,
perf event, tracepoint, Java method, etc.

The only output format that supports multiple events together is JFR.
The recording will contain the following event types:
 - `jdk.ExecutionSample`
 - `jdk.ObjectAllocationInNewTLAB` (alloc)
 - `jdk.ObjectAllocationOutsideTLAB` (alloc)
 - `jdk.JavaMonitorEnter` (lock)
 - `jdk.ThreadPark` (lock)

To start profiling cpu + allocations + locks together, specify
```
./profiler.sh -e cpu,alloc,lock -f profile.jfr ...
```
or use `--alloc` and `--lock` parameters with the desired threshold:
```
./profiler.sh -e cpu --alloc 2m --lock 10ms -f profile.jfr ...
```
The same, when starting profiler as an agent:
```
-agentpath:/path/to/libasyncProfiler.so=start,event=cpu,alloc=2m,lock=10ms,file=profile.jfr
```

## Flame Graph visualization

async-profiler provides out-of-the-box [Flame Graph](https://github.com/BrendanGregg/FlameGraph) support.
Specify `-o flamegraph` argument to dump profiling results as an interactive HTML Flame Graph.
Also, Flame Graph output format will be chosen automatically if the target filename ends with `.html`.

```
$ jps
9234 Jps
8983 Computey
$ ./profiler.sh -d 30 -f /tmp/flamegraph.html 8983
```

[![Example](https://github.com/jvm-profiling-tools/async-profiler/blob/master/demo/flamegraph.png)](https://htmlpreview.github.io/?https://github.com/jvm-profiling-tools/async-profiler/blob/master/demo/flamegraph.html)

## Profiler Options

The following is a complete list of the command-line options accepted by
`profiler.sh` script.

* `start` - starts profiling in semi-automatic mode, i.e. profiler will run
  until `stop` command is explicitly called.

* `resume` - starts or resumes earlier profiling session that has been stopped.
  All the collected data remains valid. The profiling options are not preserved
  between sessions, and should be specified again.

* `stop` - stops profiling and prints the report.

* `dump` - dump collected data without stopping profiling session.

* `check` - check if the specified profiling event is available.

* `status` - prints profiling status: whether profiler is active and
  for how long.

* `meminfo` - prints used memory statistics.

* `list` - show the list of profiling events available for the target process
  (if PID is specified) or for the default JVM.

* `-d N` - the profiling duration, in seconds. If no `start`, `resume`, `stop`
  or `status` option is given, the profiler will run for the specified period
  of time and then automatically stop.  
  Example: `./profiler.sh -d 30 8983`

* `-e event` - the profiling event: `cpu`, `alloc`, `lock`, `cache-misses` etc.
  Use `list` to see the complete list of available events.

  In allocation profiling mode the top frame of every call trace is the class
  of the allocated object, and the counter is the heap pressure (the total size
  of allocated TLABs or objects outside TLAB).

  In lock profiling mode the top frame is the class of lock/monitor, and
  the counter is number of nanoseconds it took to enter this lock/monitor.

  Two special event types are supported on Linux: hardware breakpoints
  and kernel tracepoints:
    - `-e mem:<func>[:rwx]` sets read/write/exec breakpoint at function
      `<func>`. The format of `mem` event is the same as in `perf-record`.
      Execution breakpoints can be also specified by the function name,
      e.g. `-e malloc` will trace all calls of native `malloc` function.
    - `-e trace:<id>` sets a kernel tracepoint. It is possible to specify
      tracepoint symbolic name, e.g. `-e syscalls:sys_enter_open` will trace
      all `open` syscalls.

* `-i N` - sets the profiling interval in nanoseconds or in other units,
  if N is followed by `ms` (for milliseconds), `us` (for microseconds),
  or `s` (for seconds). Only CPU active time is counted. No samples
  are collected while CPU is idle. The default is 10000000 (10ms).  
  Example: `./profiler.sh -i 500us 8983`

* `--alloc N` - allocation profiling interval in bytes or in other units,
  if N is followed by `k` (kilobytes), `m` (megabytes), or `g` (gigabytes).

* `--live` - retain allocation samples with live objects only
  (object that have not been collected by the end of profiling session).
  Useful for finding Java heap memory leaks.

* `--lock N` - lock profiling threshold in nanoseconds (or other units).
  In lock profiling mode, record contended locks that the JVM has waited for
  longer than the specified duration.

* `-j N` - sets the Java stack profiling depth. This option will be ignored if N is greater
  than default 2048.  
  Example: `./profiler.sh -j 30 8983`

* `-t` - profile threads separately. Each stack trace will end with a frame
  that denotes a single thread.  
  Example: `./profiler.sh -t 8983`

* `-s` - print simple class names instead of FQN.

* `-g` - print method signatures.

* `-a` - annotate JIT compiled methods with `_[j]`, inlined methods with `_[i]`, interpreted methods with `_[0]` and C1 compiled methods with `_[1]`.

* `-l` - prepend library names to symbols, e.g. ``libjvm.so`JVM_DefineClassWithSource``.

* `-o fmt` - specifies what information to dump when profiling ends.
  `fmt` can be one of the following options:
    - `traces[=N]` - dump call traces (at most N samples);
    - `flat[=N]` - dump flat profile (top N hot methods);  
      can be combined with `traces`, e.g. `traces=200,flat=200`
    - `jfr` - dump events in Java Flight Recorder format readable by Java Mission Control.
      This *does not* require JDK commercial features to be enabled.
    - `collapsed` - dump collapsed call traces in the format used by
      [FlameGraph](https://github.com/brendangregg/FlameGraph) script. This is
      a collection of call stacks, where each line is a semicolon separated list
      of frames followed by a counter.
    - `flamegraph` - produce Flame Graph in HTML format.
    - `tree` - produce Call Tree in HTML format.  
      `--reverse` option will generate backtrace view.

* `--total` - count the total value of the collected metric instead of the number of samples,
  e.g. total allocation size.

* `--chunksize N`, `--chunktime N` - approximate size and time limits for a single JFR chunk.
  Example: `./profiler.sh -f profile.jfr --chunksize 100m --chunktime 1h 8983`

* `-I include`, `-X exclude` - filter stack traces by the given pattern(s).
  `-I` defines the name pattern that *must* be present in the stack traces,
  while `-X` is the pattern that *must not* occur in any of stack traces in the output.
  `-I` and `-X` options can be specified multiple times. A pattern may begin or end with
  a star `*` that denotes any (possibly empty) sequence of characters.  
  Example: `./profiler.sh -I 'Primes.*' -I 'java/*' -X '*Unsafe.park*' 8983`

* `--title TITLE`, `--minwidth PERCENT`, `--reverse` - FlameGraph parameters.  
  Example: `./profiler.sh -f profile.html --title "Sample CPU profile" --minwidth 0.5 8983`

* `-f FILENAME` - the file name to dump the profile information to.  
  `%p` in the file name is expanded to the PID of the target JVM;  
  `%t` - to the timestamp;  
  `%n{MAX}` - to the sequence number;  
  `%{ENV}` - to the value of the given environment variable.  
  Example: `./profiler.sh -o collapsed -f /tmp/traces-%t.txt 8983`

* `--loop TIME` - run profiler in a loop (continuous profiling).
  The argument is either a clock time (`hh:mm:ss`) or
  a loop duration in `s`econds, `m`inutes, `h`ours, or `d`ays.
  Make sure the filename includes a timestamp pattern, or the output
  will be overwritten on each iteration.  
  Example: `./profiler.sh --loop 1h -f /var/log/profile-%t.jfr 8983`

* `--all-user` - include only user-mode events. This option is helpful when kernel profiling
  is restricted by `perf_event_paranoid` settings.  

* `--sched` - group threads by Linux-specific scheduling policy: BATCH/IDLE/OTHER.

* `--cstack MODE` - how to walk native frames (C stack). Possible modes are
  `fp` (Frame Pointer), `dwarf` (DWARF unwind info),
  `lbr` (Last Branch Record, available on Haswell since Linux 4.1),
  and `no` (do not collect C stack).

  By default, C stack is shown in cpu, itimer, wall-clock and perf-events profiles.
  Java-level events like `alloc` and `lock` collect only Java stack.

* `--begin function`, `--end function` - automatically start/stop profiling
  when the specified native function is executed.

* `--ttsp` - time-to-safepoint profiling. An alias for  
  `--begin SafepointSynchronize::begin --end RuntimeService::record_safepoint_synchronized`  
  It is not a separate event type, but rather a constraint. Whatever event type
  you choose (e.g. `cpu` or `wall`), the profiler will work as usual, except that
  only events between the safepoint request and the start of the VM operation
  will be recorded.

* `--jfrsync CONFIG` - start Java Flight Recording with the given configuration
  synchronously with the profiler. The output .jfr file will include all regular
  JFR events, except that execution samples will be obtained from async-profiler.
  This option implies `-o jfr`.
    - `CONFIG` is a predefined JFR profile or a JFR configuration file (.jfc).

  Example: `./profiler.sh -e cpu --jfrsync profile -f combined.jfr 8983`

* `--fdtransfer` - runs "fdtransfer" alongside, which is a small program providing an interface
  for the profiler to access `perf_event_open` even while this syscall is unavailable for the
  profiled process (due to low privileges).
  See [Profiling Java in a container](#profiling-java-in-a-container).

* `-v`, `--version` - prints the version of profiler library. If PID is specified,
  gets the version of the library loaded into the given process.

## Profiling Java in a container

It is possible to profile Java processes running in a Docker or LXC container
both from within a container and from the host system.

When profiling from the host, `pid` should be the Java process ID in the host
namespace. Use `ps aux | grep java` or `docker top <container>` to find
the process ID.

async-profiler should be run from the host by a privileged user - it will
automatically switch to the proper pid/mount namespace and change
user credentials to match the target process. Also make sure that
the target container can access `libasyncProfiler.so` by the same
absolute path as on the host.

By default, Docker container restricts the access to `perf_event_open`
syscall. There are 3 alternatives to allow profiling in a container:
1. You can modify the [seccomp profile](https://docs.docker.com/engine/security/seccomp/)
or disable it altogether with `--security-opt seccomp=unconfined` option. In
addition, `--cap-add SYS_ADMIN` may be required.
2. You can use "fdtransfer": see the help for `--fdtransfer`.
3. Last, you may fall back to `-e itimer` profiling mode, see [Troubleshooting](#troubleshooting).

## Restrictions/Limitations

* macOS profiling is limited to user space code only.

* On most Linux systems, `perf_events` captures call stacks with a maximum depth
  of 127 frames. On recent Linux kernels, this can be configured using
  `sysctl kernel.perf_event_max_stack` or by writing to the
  `/proc/sys/kernel/perf_event_max_stack` file.

* Profiler allocates 8kB perf_event buffer for each thread of the target process.
  Make sure `/proc/sys/kernel/perf_event_mlock_kb` value is large enough
  (more than `8 * threads`) when running under unprivileged user.
  Otherwise the message _"perf_event mmap failed: Operation not permitted"_
  will be printed, and no native stack traces will be collected.

* There is no bullet-proof guarantee that the `perf_events` overflow signal
  is delivered to the Java thread in a way that guarantees no other code has run,
  which means that in some rare cases, the captured Java stack might not match
  the captured native (user+kernel) stack.

* You will not see the non-Java frames _preceding_ the Java frames on the
  stack. For example, if `start_thread` called `JavaMain` and then your Java
  code started running, you will not see the first two frames in the resulting
  stack. On the other hand, you _will_ see non-Java frames (user and kernel)
  invoked by your Java code.

* No Java stacks will be collected if `-XX:MaxJavaStackTraceDepth` is zero
  or negative.

* Too short profiling interval may cause continuous interruption of heavy
  system calls like `clone()`, so that it will never complete;
  see [#97](https://github.com/jvm-profiling-tools/async-profiler/issues/97).
  The workaround is simply to increase the interval.

* When agent is not loaded at JVM startup (by using -agentpath option) it is
  highly recommended to use `-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints` JVM flags.
  Without those flags the profiler will still work correctly but results might be
  less accurate. For example, without `-XX:+DebugNonSafepoints` there is a high chance
  that simple inlined methods will not appear in the profile. When the agent is attached at runtime,
  `CompiledMethodLoad` JVMTI event enables debug info, but only for methods compiled after attaching.

## Troubleshooting

```
Failed to change credentials to match the target process: Operation not permitted
```
Due to limitation of HotSpot Dynamic Attach mechanism, the profiler must be run
by exactly the same user (and group) as the owner of target JVM process.
If profiler is run by a different user, it will try to automatically change
current user and group. This will likely succeed for `root`, but not for
other users, resulting in the above error.

```
Could not start attach mechanism: No such file or directory
```
The profiler cannot establish communication with the target JVM through UNIX domain socket.

Usually this happens in one of the following cases:
1. Attach socket `/tmp/.java_pidNNN` has been deleted. It is a common
   practice to clean `/tmp` automatically with some scheduled script.
   Configure the cleanup software to exclude `.java_pid*` files from deletion.  
   How to check: run `lsof -p PID | grep java_pid`  
   If it lists a socket file, but the file does not exist, then this is exactly
   the described problem.
2. JVM is started with `-XX:+DisableAttachMechanism` option.
3. `/tmp` directory of Java process is not physically the same directory
   as `/tmp` of your shell, because Java is running in a container or in
   `chroot` environment. `jattach` attempts to solve this automatically,
   but it might lack the required permissions to do so.  
   Check `strace build/jattach PID properties`
4. JVM is busy and cannot reach a safepoint. For instance,
   JVM is in the middle of long-running garbage collection.  
   How to check: run `kill -3 PID`. Healthy JVM process should print
   a thread dump and heap info in its console.

```
Failed to inject profiler into <pid>
```
The connection with the target JVM has been established, but JVM is unable to load profiler shared library.
Make sure the user of JVM process has permissions to access `libasyncProfiler.so` by exactly the same absolute path.
For more information see [#78](https://github.com/jvm-profiling-tools/async-profiler/issues/78).

```
No access to perf events. Try --fdtransfer or --all-user option or 'sysctl kernel.perf_event_paranoid=1'
```
or
```
Perf events unavailable
```
`perf_event_open()` syscall has failed.

Typical reasons include:
1. `/proc/sys/kernel/perf_event_paranoid` is set to restricted mode (>=2).
2. seccomp disables `perf_event_open` API in a container.
3. OS runs under a hypervisor that does not virtualize performance counters.
4. perf_event_open API is not supported on this system, e.g. WSL.

For permissions-related reasons (such as 1 and 2), using `--fdtransfer` while running the profiler
as a privileged user will allow using perf_events.

If changing the configuration is not possible, you may fall back to
`-e itimer` profiling mode. It is similar to `cpu` mode, but does not
require perf_events support. As a drawback, there will be no kernel
stack traces.

```
No AllocTracer symbols found. Are JDK debug symbols installed?
```
The OpenJDK debug symbols are required for allocation profiling.
See [Installing Debug Symbols](#installing-debug-symbols) for more details.
If the error message persists after a successful installation of the debug symbols,
it is possible that the JDK was upgraded when installing the debug symbols.
In this case, profiling any Java process which had started prior to the installation
will continue to display this message, since the process had loaded
the older version of the JDK which lacked debug symbols.
Restarting the affected Java processes should resolve the issue.

```
VMStructs unavailable. Unsupported JVM?
```
JVM shared library does not export `gHotSpotVMStructs*` symbols -
apparently this is not a HotSpot JVM. Sometimes the same message
can be also caused by an incorrectly built JDK
(see [#218](https://github.com/jvm-profiling-tools/async-profiler/issues/218)).
In these cases installing JDK debug symbols may solve the problem.

```
Could not parse symbols from <libname.so>
```
Async-profiler was unable to parse non-Java function names because of
the corrupted contents in `/proc/[pid]/maps`. The problem is known to
occur in a container when running Ubuntu with Linux kernel 5.x.
This is the OS bug, see https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1843018.

```
Could not open output file
```
Output file is written by the target JVM process, not by the profiler script.
Make sure the path specified in `-f` option is correct and is accessible by the JVM.
