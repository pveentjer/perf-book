---
typora-root-url: ..\..\img
---

[TODO:] 1. reformat
[TODO:] 2. enhance Intel's section
[TODO:] 3. read about ARM's BRBE
[TODO:] 4. incorporate AMD's LBR

## Branch Recording Mechanisms {#sec:lbr}

Modern high-performance CPUs provide branch recording mechanisms that enable a processor to continuously log a set of previously executed branches. But before going into the details, one might ask: *Why are we so interested in branches?* Well, because this is how we can determine the control flow of a program. We largely ignore other instructions in a basic block (see [@sec:BasicBlock]) because branches are always the last instruction in a basic block. Since all instructions in a basic block are guaranteed to be executed once, we can only focus on branches that will “represent” the entire basic block. Thus, it’s possible to reconstruct the entire line-by-line execution path of the program if we track the outcome of every branch. In fact, this is what the Intel Processor Traces (PT) feature is capable of doing, which is discussed in Appendix D. Branch recording mechanisms that we will discuss here are based on samling, not tracing, and thus have different use cases and capabilities.

Processors designed by Intel, AMD, and ARM all have announced their branch recording extensions. Exact implementations may vary but the idea is the same. Hardware logs the “from” and “to” address of each branch along with some additional data in parallel with executing the program. If we collect a long enough history of source-destination pairs, we will be able to unwind the control flow of our program, just like a call stack, but with limited depth. Such extensions are designed to cause minimal slowdown to a running program often within 1%. 

With a branch recording mechanism in place, we can sample on branches (or cycles, it doesn't matter), but during each sample, look at the previous N branches that were executed. This gives us reasonable coverage of the control flow in the hot code paths but does not overwhelm us with too much information, as only a smaller number of the total branches are examined. It is important to keep in mind that this is still sampling, so not every executed branch can be examined. A CPU generally executes too fast for that to be feasible.

It is very important to keep in mind that only taken branches are being logged. [@lst:LogBranches] shows an example of how branch results are being tracked. This code represents a loop with three instructions that may change the execution path of the program, namely loop backedge `JNE` (1), conditional branch `JNS` (2), function `CALL` (3), and return from this function (4, not shown).

Listing: Example of logging branches.

~~~~ {#lst:LogBranches .asm}
----> 4eda10:  mov   edi,DWORD PTR [rbx]
|     4eda12:  test  edi,edi
| --- 4eda14:  jns   4eda1e              <== (2)
| |   4eda16:  mov   eax,edi
| |   4eda18:  shl   eax,0x7
| |   4eda1b:  lea   edi,[rax+rdi*8]
| └─> 4eda1e:  call  4edb26              <== (3)
|     4eda23:  add   rbx,0x4             <== (4)
|     4eda27:  mov   DWORD PTR [rbx-0x4],eax
|     4eda2a:  cmp   rbx,rbp
 ---- 4eda2d:  jne   4eda10              <== (1)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Below is one of the possibile branch histories that is logged with a branch recording mechanisms. It shows the last 7 branch outcomes (many more not shown) at the moment we executed the `CALL` instruction. Because on the latest iteration of the loop the `JNS` branch (`4eda14` -> `4eda1e`) was not taken, it is not logged and thus does not appear in the history.

```
    Source Address    Destination Address
    ...               ...
(1) 4eda2d            4eda10    <== next iteration
(2) 4eda14            4eda1e    <== jns taken
(3) 4eda1e            4edb26    <== call a function
(4) 4b01cd            4eda23    <== return from a function
(1) 4eda2d            4eda10    <== next iteration
(3) 4eda1e            4edb26    <== latest branch
```

Untaken branches not being logged might add an additional burden for analysis but usually don’t complicate it too much. We can still infer the complete execution path since we know that the control flow was sequential from destination address in entry `N-1` to source address in entry `N`.

Next we will take a look at each vendor's branch recording mechanism and then explore how they can be used in performance analysis.

### LBR on Intel Platforms

Intel has first implemented its Last Branch Record (LBR) facility in its Netburst microarchitecture. Initially, it could record only 4 most recent branch outcomes. It was later enhanced to 16 starting with Nehalem and to 32 starting from Skylake. Prior to the Goldencove microarchitecture, LBR was implemented as a set of model-specific registers (MSRs), but now it works within architectural registers. The primary advantage is that LBR features are clearly exposed without knowing the model number of the current CPU. Also, LBR entries can be configured to be included in the PEBS records (see [@sec:secPEBS]).

The LBR registers act like a ring buffer that is continuously overwritten and provides only 32 most recent branch outcomes. Each LBR entry is comprised of three 64-bit values:

- The source address of the branch (`From IP`).
- The destination address of the branch (`To IP`).
- Metadata for the operation, including mispredict, and elapsed cycle time information.

There are important applications to the additional information saved besides just source and destination addresses, which we will discuss later.

When a sampling counter overflows and a Performance Monitoring Interrupt (PMI) is triggered, the LBR logging freezes until software captures the LBR records and resumes collection.

LBR collection can be limited to a set of specific branch types, for example a user may choose to log only function calls and returns. When applying such filter to the code in [@lst:LogBranches], we would only see branches (3) and (4) in the history. Users can also filter in/out conditional and unconditional jumps, indirect jumps and calls, system calls, interrupts and others.

By default, the LBR array works as a ring buffer that captures control flow transitions. However, the depth of the LBR array is limited, which can be a limiting factor when profiling certain applications, in which a transition of the execution flow is accompanied by a large number of leaf function calls. These calls to leaf functions, and their returns, are likely to displace the main execution context from the LBRs. Consider the example in [@lst:LogBranches] again. Say, we want to unwind the call stack from our LBS, and thus we configured LBR to capture only function calls and returns. If the loop runs thousands of iterations and taking into account that the LBR array is 32 entries deep, there is a very high chance we would only see 16 pairs of entries (3) and (4). Those are all leaf function calls which don't help us to unwind the call stack.

This is why LBR supports call-stack mode. With this mode enabled, the LBR array captures function calls as before, but as return instructions are executed the last captured branch (`call`) record is flushed from the array in a last-in first-out (LIFO) manner. Thus, branch information pertaining to completed leaf functions will not be retained, while preserving the call stack information of the main line execution path. When configured in this manner, the LBR array emulates a call stack, where `CALL`s are “pushed” and `RET`s “pop” entries off the stack.

Users can make sure LBRs are enabled on their system by doing the following command:

```bash
$ dmesg | grep -i lbr
[    0.228149] Performance Events: PEBS fmt3+, 32-deep LBR, Skylake events, full-width counters, Intel PMU driver.
```

[TODO]: add Intel manual reference

Runtime overhead for the majority of LBR use cases is below 1%. [@Nowak2014TheOO]

### LBR on AMD Platforms

### BRBE on ARM Platforms

[TODO]: read https://documentation-service.arm.com/static/632dbdace68c6809a6b41710?token=
Chapter F1 Branch Record Buffer Extension

### Collect LBR Stacks

With Linux `perf`, one can collect LBR stacks using the command below:

```bash
$ ~/perf record -b -e cycles ./a.exe
[ perf record: Woken up 68 times to write data ]
[ perf record: Captured and wrote 17.205 MB perf.data (22089 samples) ]
```

LBR stacks can also be collected using `perf record --call-graph lbr` command, but the amount of information collected is less than using `perf record -b`. For example, branch misprediction and cycles data is not collected when running `perf record --call-graph lbr`. 

Because each collected sample captures the entire LBR stack (32 last branch records), the size of collected data (`perf.data`) is significantly bigger than sampling without LBRs. Below is the Linux perf command one can use to dump the contents of collected branch stacks:

```bash
$ perf script -F brstack &> dump.txt
```

The `dump.txt` file, which can be quite large, contains lines like those shown below:

```
...
0x4edabd/0x4edad0/P/-/-/2   0x4edaf9/0x4edab0/P/-/-/29
0x4edabd/0x4edad0/P/-/-/2   0x4edb24/0x4edab0/P/-/-/23
0x4edadd/0x4edb00/M/-/-/4   0x4edabd/0x4edad0/P/-/-/2
0x4edb24/0x4edab0/P/-/-/24  0x4edadd/0x4edb00/M/-/-/4
0x4edabd/0x4edad0/P/-/-/2   0x4edb24/0x4edab0/P/-/-/23
0x4edadd/0x4edb00/M/-/-/1   0x4edabd/0x4edad0/P/-/-/1
0x4edb24/0x4edab0/P/-/-/3   0x4edadd/0x4edb00/P/-/-/1
0x4edabd/0x4edad0/P/-/-/1   0x4edb24/0x4edab0/P/-/-/3
...
```

On the block above, we present eight entries from the LBR stack, which typically consists of 32 LBR entries. Each entry has `FROM` and `TO` addresses (hexadecimal values), predicted flag (`M`/`P`),[^16] and a number of cycles (number in the last position of each entry). Components marked with "`-`" are related to transactional memory (TSX), which we won't discuss here. Curious readers can look up the format of a decoded LBR entry in the `perf script` [specification](http://man7.org/linux/man-pages/man1/perf-script.1.html)[^2].

There is a number of important use cases for LBR. In the next sections, we will address the most important ones.

### Capture Call Graph

Discussions on collecting call graph and its importance were covered in [@sec:secCollectCallStacks]. LBR can be used for collecting call-graph information even if you compiled a program without frame pointers (controlled by compiler option `-fomit-frame-pointer`, ON by default) or debug information:[^3]

```bash
$ perf record --call-graph lbr -- ./a.exe
$ perf report -n --stdio
# Children   Self    Samples  Command  Object  Symbol       
# ........  .......  .......  .......  ......  .......
	99.96%  99.94%    65447    a.out    a.out  [.] bar
            |          
             --99.94%--main
                       |          
                       |--90.86%--foo
                       |          |          
                       |           --90.86%--bar
                       |          
                        --9.08%--zoo
                                  bar
```

As you can see, we identified the hottest function in the program (which is `bar`). Also, we found out callers that contribute to the most time spent in function `bar` (it is `foo`). In this case, we can see that 91% of samples in `bar` have `foo` as its caller function.[^4]

Using LBR, we can determine a Hyper Block (sometimes called Super Block), which is a chain of basic blocks executed most frequently in the whole program. Basic blocks from that chain are not necessarily laid in the sequential physical order; they're executed sequentially.

### Identify Hot Branches {#sec:lbr_hot_branch}

LBR also enables us to know what were the most frequently taken branches: 

```bash
$ perf record -e cycles -b -- ./a.exe
[ perf record: Woken up 3 times to write data ]
[ perf record: Captured and wrote 0.535 MB perf.data (670 samples) ]
$ perf report -n --sort overhead,srcline_from,srcline_to -F +dso,symbol_from,symbol_to --stdio
# Samples: 21K of event 'cycles'
# Event count (approx.): 21440
# Overhead  Samples  Object  Source Sym  Target Sym  From Line  To Line
# ........  .......  ......  ..........  ..........  .........  .......
  51.65%      11074   a.exe   [.] bar    [.] bar      a.c:4      a.c:5
  22.30%       4782   a.exe   [.] foo    [.] bar      a.c:10     (null)
  21.89%       4693   a.exe   [.] foo    [.] zoo      a.c:11     (null)
   4.03%        863   a.exe   [.] main   [.] foo      a.c:21     (null)
```

From this example, we can see that more than 50% of taken branches are inside the `bar` function, 22% of branches are function calls from `foo` to `bar`, and so on. Notice how `perf` switched from `cycles` events to analyzing LBR stacks: only 670 samples were collected, yet we have an entire LBR stack captured with every sample. This gives us `670 * 32 = 21440` LBR entries (branch outcomes) for analysis.[^5]

Most of the time, it’s possible to determine the location of the branch just from the line of code and target symbol. However, theoretically, one could write code with two `if` statements written on a single line. Also, when expanding the macro definition, all the expanded code gets the same source line, which is another situation when this might happen. This issue does not totally block the analysis but only makes it a little more difficult. To disambiguate two branches, you likely need to analyze raw LBR stacks yourself (see example on [easyperf](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability)[^6] blog).

### Analyze Branch Misprediction Rate {#sec:secLBR_misp_rate}

It’s also possible to know the misprediction rate for hot branches:[^7]

```bash
$ perf record -e cycles -b -- ./a.exe
$ perf report -n --sort symbol_from,symbol_to -F +mispredict,srcline_from,srcline_to --stdio
# Samples: 657K of event 'cycles'
# Event count (approx.): 657888
# Overhead  Samples  Mis  From Line  To Line  Source Sym  Target Sym
# ........  .......  ...  .........  .......  ..........  ..........
    46.12%   303391   N   dec.c:36   dec.c:40  LzmaDec     LzmaDec   
    22.33%   146900   N   enc.c:25   enc.c:26  LzmaFind    LzmaFind  
     6.70%    44074   N   lz.c:13    lz.c:27   LzmaEnc     LzmaEnc   
     6.33%    41665   Y   dec.c:36   dec.c:40  LzmaDec     LzmaDec 
```

In this example,[^8] lines that correspond to function `LzmaDec` are of particular interest to us. Using the reasoning from [@sec:lbr_hot_branch], we can conclude that the branch on source line `dec.c:36` is the most executed branch in the benchmark. In the output that Linux `perf` provides, we can spot two entries that correspond to the `LzmaDec` function: one with `Y` and one with `N` letters. Analyzing those two entries together gives us a misprediction rate of the branch. In this case, we know that the branch on line `dec.c:36` was predicted `303391` times (corresponds to `N`) and was mispredicted `41665` times (corresponds to `Y`), which gives us `88%` prediction rate.

Linux `perf` calculates the misprediction rate by analyzing each LBR entry and extracting misprediction bits from it. So that for every branch, we have a number of times it was predicted correctly and a number of mispredictions. Again, due to the nature of sampling, it is possible that some branches might have an `N` entry but no corresponding `Y` entry. It could mean there are no LBR entries for the branch being mispredicted, but that doesn’t necessarily mean the prediction rate is `100%`.

### Precise Timing of Machine Code {#sec:timed_lbr}

As was discussed in [@sec:lbr], starting from Skylake, LBR entries provide `Cycle Count` information. This additional field specifies the number of elapsed cycles between two taken branches. If the target address in the previous LBR entry is the beginning of a basic block (BB) and the source address of the current LBR entry is the last instruction of the same basic block, then the cycle count is the latency of this basic block. For example:

```
400618:   movb  $0x0, (%rbp,%rdx,1)    <= start of a BB
40061d:   add $0x1, %rdx 
400621:   cmp $0xc800000, %rdx 
400628:   jnz 0x400644                 <= end of a BB
```

Suppose we have two entries in the LBR stack:

```
  FROM_IP   TO_IP    Cycle Count
  ...       ...      ...
  40060a    400618    10
  400628    400644     5          <== LBR TOS
```

Given that information, we know that there was one occurrence when the basic block that starts at offset `400618` was executed in 5 cycles. If we collect enough samples, we could plot a probability density function of the latency for that basic block (see Figure @fig:LBR_timing_BB). This chart was compiled by analyzing all LBR entries that satisfy the rule described above. For example, the basic block was executed in ~75 cycles only 4% of the time, but more often, it was executed between 260 and 314 cycles. This block has a random load inside a huge array that doesn’t fit in CPU L3 cache, so the latency of the basic block largely depends on this load. There are two important spikes on the chart that is shown in Figure @fig:LBR_timing_BB: first, around 80 cycles corresponds to the L3 cache hit, and the second spike, around 300 cycles, corresponds to L3 cache miss where the load request goes all the way down to the main memory.

![Probability density function for latency of the basic block that starts at address `0x400618`.](../../img/pmu-features/LBR_timing_BB.jpg){#fig:LBR_timing_BB width=90%}

This information can be used for further fine-grained tuning of this basic block. This example might benefit from memory prefetching, which we will discuss in [@sec:memPrefetch]. Also, this cycle information can be used for timing loop iterations, where every loop iteration ends with a taken branch (back edge).

An example of how to build a probability density function for the latency of a basic block can be found on the [easyperf blog](https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf)[^9]. However, in newer versions of Linux perf, getting this information is much easier. For example:[^7]

```bash
$ perf record -e cycles -b -- ./a.exe
$ perf report -n --sort symbol_from,symbol_to -F +cycles,srcline_from,srcline_to --stdio
# Samples: 658K of event 'cycles'
# Event count (approx.): 658240
# Overhead  Samples  BBCycles  FromSrcLine  ToSrcLine   
# ........  .......  ........  ...........  ..........  
     2.82%   18581      1      dec.c:325    dec.c:326   
     2.54%   16728      2      dec.c:174    dec.c:174   
     2.40%   15815      4      dec.c:174    dec.c:174   
     2.28%   15032      2      find.c:375   find.c:376  
     1.59%   10484      1      dec.c:174    dec.c:174   
     1.44%   9474       1      enc.c:1310   enc.c:1315  
     1.43%   9392      10      7zCrc.c:15   7zCrc.c:17  
     0.85%   5567      32      dec.c:174    dec.c:174   
     0.78%   5126       1      enc.c:820    find.c:540  
     0.77%   5066       1      enc.c:1335   enc.c:1325  
     0.76%   5014       6      dec.c:299    dec.c:299   
     0.72%   4770       6      dec.c:174    dec.c:174   
     0.71%   4681       2      dec.c:396    dec.c:395   
     0.69%   4563       3      dec.c:174    dec.c:174   
     0.58%   3804      24      dec.c:174    dec.c:174   
```

Several insignificant lines were removed from the output of `perf record` to make it fit on the page. If we now focus on the branch in which source and destination is `dec.c:174`[^10], we can find multiple lines associated with it. Linux `perf` sorts entries by overhead first, which requires us to manually filter entries for the branch which we are interested in. In fact, if we filter them, we will get the latency distribution for the basic block that ends with this branch, as shown in Table {@tbl:bb_latency}. This data can be plotted to obtain a chart similar to the one shown in Figure @fig:LBR_timing_BB.

----------------------------------------------
Cycles  Number of samples  Probability density
------  -----------------  -------------------
1       10484              17.0%

2       16728              27.1%

3       4563                7.4%

4       15815              25.6%

6       4770                7.7%

24      3804                6.2%

32      5567                9.0%
----------------------------------------------

Table: Probability density for basic block latency. {#tbl:bb_latency}

Currently, timed LBR is the most precise cycle-accurate source of timing information in the system.

### Estimating Branch Outcome Probability

Later in [@sec:secFEOpt], we will discuss the importance of code layout for performance. Going forward a little bit, having a hot path in a fall through manner[^11] generally improves the performance of the program. Considering a single branch, knowing that `condition` 99% of the time is false or true is essential for a compiler to make better optimization decisions.

LBR enables us to get this data without instrumenting the code. As the outcome from the analysis, the user will get a ratio between true and false outcomes of the condition, i.e., how many times the branch was taken and how much not taken. This feature especially shines when analyzing indirect jumps (switch statement) and indirect calls (virtual calls). You can find examples of using it on a real-world application on the [easyperf blog](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability)[^6].

### Other Use Cases

* **Profile guided optimizations**: LBR can provide profiling feedback data for optimizing compilers. LBR can be a better choice as opposed to static code instrumentation when runtime overhead is considered.
* **Capturing function arguments**: when LBR is used together with PEBS (see [@sec:secPEBS]), it is possible to capture function arguments, since according to [x86 calling conventions](https://en.wikipedia.org/wiki/X86_calling_conventions),[^12] the first few arguments of a callee are passed in registers that are recorded in a PEBS entry. [@IntelOptimizationManual, Appendix B, Chapter B.3.3.4]
* **Basic Block Execution Counts**: since all the basic blocks between a branch IP (source) and the previous target in the LBR stack are executed exactly once, it’s possible to evaluate the execution rate of basic blocks inside a program. This process involves building a map of starting addresses of each basic block and then traversing collected LBR stacks backward. [@IntelOptimizationManual, Appendix B, Chapter B.3.3.4]


[^2]: Linux `perf script` manual page - [http://man7.org/linux/man-pages/man1/perf-script.1.html](http://man7.org/linux/man-pages/man1/perf-script.1.html).
[^3]: Utilized by `perf record --call-graph dwarf`.
[^4]: We cannot necessarily drive conclusions about function call counts in this case. For example, we cannot say that `foo` calls `bar` 10 times more than `zoo`. It could be the case that `foo` calls `bar` once, but it executes an expensive path inside `bar` while `zoo` calls `bar` lots of times but returns quickly from it.
[^5]: The report header generated by perf confuses users because it says `21K of event cycles`. But there are `21K` LBR entries, not `cycles`.
[^6]: Analyzing raw LBR stacks - [https://easyperf.net/blog/2019/05/06/Estimating-branch-probability](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability).
[^7]: Adding `-F +srcline_from,srcline_to` slows down building report. Hopefully, in newer versions of perf, decoding time will be improved.
[^8]: This example is taken from the real-world application, 7-zip benchmark: [https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip](https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip). Although the output of perf report is trimmed a little bit to fit nicely on a page.
[^9]: Building a probability density function for the latency of an arbitrary basic block - [https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf](https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf).
[^10]: In the source code, line `dec.c:174` expands a macro that has a self-contained branch. That’s why the source and destination happen to be on the same line.
[^11]: I.e., when outcomes of branches are not taken.
[^12]: X86 calling conventions - [https://en.wikipedia.org/wiki/X86_calling_conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)
[^16]: M - Mispredicted, P - Predicted.