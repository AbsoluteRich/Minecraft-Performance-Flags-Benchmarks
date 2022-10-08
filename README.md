Discord for questions and such: https://discord.gg/zeFSR9PnUw 

Benchmarks
======

Flags are tested with Benchmark.py script. See the work-in-progress [Benchmarks.md](https://github.com/brucethemoose/Minecraft-Performance-Flags-Benchmarks/blob/main/Benchmarks.md).


Picking a Java Runtume
======

For Minecraft 1.16.5 and up, use Java 17. Some launchers like PolyMC or Curseforge ask you to use Java 8 on 1.16.X specifically, but Minecraft and *most* mods are compatible with Java 17.

1.12.2 and below generally requires Java 8. Sometimes Java 11+ will work. 

Most Java runtimes from Azul, Microsoft, Adoptium, Amazon and so on are basically identical. Some notable exceptions:

- **Oracle GraalVM Enterprise Edition** features a more aggressive Java compiler. This is what I personally run Minecraft with, see the GraalVM section below.

- **Intel's Clear Linux OpenJDK** uses the same code as any other OpenJDK (making it highly compatible with mods), but the build process itself is optimized for newer CPUs. Grab it from Clear Linux's repos, or from here: https://hub.docker.com/r/clearlinux/openjdk

- **Azul's Prime OpenJDK** is *very* fast since it hooks into llvm, but its currently incompatible with most mods and is linux-only. Get it from here: https://docs.azul.com/prime/prime-quick-start-tar

- **Red Hat Java 8** has the Shenandoah garbage collector. Its gated behind a free email signup: https://developers.redhat.com/products/openjdk/download

- **IBM's OpenJ9** is... *much* slower in Minecraft, and uses totally different flags than any other Java build. See [FAQ](#FAQ).

If you dont know what to pick on Windows or OSX, I recommend Graal EE (see below) or the latest Adoptium JRE: https://adoptium.net/

Base Java Flags
======
These optimized flags will work with any Java 11+ build. They work on both servers and clients:


```-XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+AlwaysPreTouch -XX:+DisableExplicitGC -XX:+UseNUMA -XX:NmethodSweepActivity=1 -XX:ReservedCodeCacheSize=400M -XX:NonNMethodCodeHeapSize=12M -XX:ProfiledCodeHeapSize=194M -XX:NonProfiledCodeHeapSize=194M -XX:-DontCompileHugeMethods -XX:MaxNodeLimit=240000 -XX:NodeLimitFudgeFactor=8000 -XX:+UseVectorCmov -XX:+PerfDisableSharedMem -XX:+UseFastUnorderedTimeStamps -XX:+UseCriticalJavaThreadPriority -XX:ThreadPriorityPolicy=1 -XX:AllocatePrefetchStyle=3```


**You *must* add garbage collection flags to these java arguments.**  

<details>
    <summary>A full set of flags looks like this:</summary>

    -xms8G -XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+AlwaysPreTouch -XX:+DisableExplicitGC -XX:+UseNUMA -XX:NmethodSweepActivity=1 -XX:ReservedCodeCacheSize=400M -XX:NonNMethodCodeHeapSize=12M -XX:ProfiledCodeHeapSize=194M -XX:NonProfiledCodeHeapSize=194M -XX:-DontCompileHugeMethods -XX:MaxNodeLimit=240000 -XX:NodeLimitFudgeFactor=8000 -XX:+UseVectorCmov -XX:+PerfDisableSharedMem -XX:+UseFastUnorderedTimeStamps -XX:+UseCriticalJavaThreadPriority -XX:ThreadPriorityPolicy=1 -XX:AllocatePrefetchStyle=3  -XX:+UseG1GC -XX:MaxGCPauseMillis=37 -XX:+PerfDisableSharedMem -XX:G1HeapRegionSize=16M -XX:G1NewSizePercent=23 -XX:G1ReservePercent=20 -XX:SurvivorRatio=32 -XX:G1MixedGCCountTarget=3 -XX:G1HeapWastePercent=20 -XX:InitiatingHeapOccupancyPercent=10 -XX:G1RSetUpdatingPauseTimePercent=0 -XX:MaxTenuringThreshold=1 -XX:G1SATBBufferEnqueueingThresholdPercent=30 -XX:G1ConcMarkStepDurationMillis=5.0 -XX:G1ConcRSHotCardLimit=16 -XX:G1ConcRefinementServiceIntervalMillis=150 -XX:GCTimeRatio=99 -XX:+UseLargePages -XX:LargePageSizeInBytes=2m
    
    
</details>



Memory Allocation
======
Minimum and maximum (`-xms` and `-xmx`) memory should be set to the same value, as explained here: https://dzone.com/articles/benefits-of-setting-initial-and-maximum-memory-siz

Sizes are set in megabytes (`-Xms4096M`) or gigabytes (`-Xmx8G`)

Allocating too much memory can force your operating system to page, make garbage collection pauses more severe, and/or slow the game down. Allocating too little can also slow the game down. Keep a close eye on the Windows Task manager (or your DE's system monitor) as Minecraft is running, and allocate only as much as it needs (which is usually less than 8G).

Garbage Collection
======

**Garbage collection flags must be added to Minecraft servers and clients**, as the default "pauses" to stop and collect garbage manifest as stutters on the client and lag on servers. Use the`sparkc gcmonitor` command in the [Spark](https://www.curseforge.com/minecraft/mc-mods/spark) mod to observe pauses. *Any* old generation pauses are bad, and young generation G1GC collections should be infrequent, but short enough to be imperceptible.  

Pick one set of flags. I recommend **Shenandoah on clients**, **ZGC on powerful Java 17 servers**, and **G1GC on Graal** or on servers/clients with less RAM and fewer cores:


### ZGC 

ZGC is great for high memory/high core count servers. It has no throughput hit I can measure, and absolutely does not stutter. However, it requires more RAM and more cores than other garbage collectors.

Unfortunately, it has a significant client FPS hit on my (8-core/16 thread) laptop. See the "ZGC" benchmark in the benchmarks folder. 

`-XX:+UseZGC -XX:AllocatePrefetchStyle=1 -XX:-ZProactive` enables it, but allocate more RAM and more `ConcGCThreads` than you normally would for other GC. Note that ZGC does not like AllocatePrefetchStyle=3, hence setting it to 1 overrides the previous entry.


### Shenandoah

Shenandoah performs well on clients, but kills server throughput in my tests. Enable it with `-XX:+UseShenandoahGC -XX:ShenandoahGCMode=iu -XX:ShenandoahGuaranteedGCInterval=1000000 -XX:AllocatePrefetchStyle=1` 

See more tuning options [here](https://wiki.openjdk.org/display/shenandoah/Main). The "herustic" and "mode" options don't change much for me (except for "compact," which you should not use). Like ZGC, Shenandoah does not like AllocatePrefetchStyle=3.

Note that Shenandoah is not in Java 8, or in any Oracle Java builds! If you are a Java 8 user, you must use Red Hat OpenJDK to use Shenandoah: https://developers.redhat.com/products/openjdk/download


### Client G1GC

G1GC is the default garbage collector, and is the only available garbage collector for [GraalVM users](https://github.com/oracle/graal/issues/2149). Aikar's [famous Minecraft G1GC arguments](https://aikar.co/2018/07/02/tuning-the-jvm-g1gc-garbage-collector-flags-for-minecraft/) run great on clients, with one caveat: they effectively [clamp](https://www.oracle.com/technical-resources/articles/java/g1gc.html) the `MaxGCPauseMillis` parameter by setting `G1NewSizePercent` so high, producing long stutters on some clients.

These are similar to the aikar flags, but with shorter, more frequent pauses, less aggressive G1 mixed collection and some tweaks to background collection: `-XX:+UseG1GC -XX:MaxGCPauseMillis=37 -XX:+PerfDisableSharedMem -XX:G1HeapRegionSize=16M -XX:G1NewSizePercent=23 -XX:G1ReservePercent=20 -XX:SurvivorRatio=32 -XX:G1MixedGCCountTarget=3 -XX:G1HeapWastePercent=20 -XX:InitiatingHeapOccupancyPercent=10 -XX:G1RSetUpdatingPauseTimePercent=0 -XX:MaxTenuringThreshold=1 -XX:G1SATBBufferEnqueueingThresholdPercent=30 -XX:G1ConcMarkStepDurationMillis=5.0 -XX:G1ConcRSHotCardLimit=16 -XX:G1ConcRefinementServiceIntervalMillis=150 -XX:GCTimeRatio=99`  

`G1NewSizePercent` and `MaxGCPauseMillis` can be used to tune the frequency/dureation of your young generation collections. `G1HeapWastePercent=18` should be removed if you are getting any old generation pauses on your setup. Alternatively, you can raise it and set `G1MixedGCCountTarget` to 2 or 1 to make mixed garbage collection even lazier. 


### Server G1GC

Longer pauses are more acceptable on servers. These flags are very close to the aikar defaults:

`-XX:+UseG1GC -XX:MaxGCPauseMillis=130 -XX:+UnlockExperimentalVMOptions -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -XX:G1NewSizePercent=28 -XX:G1HeapRegionSize=16M -XX:G1ReservePercent=20 -XX:G1MixedGCCountTarget=3 -XX:InitiatingHeapOccupancyPercent=10 -XX:G1MixedGCLiveThresholdPercent=90 -XX:G1RSetUpdatingPauseTimePercent=0 -XX:SurvivorRatio=32 -XX:MaxTenuringThreshold=1 -XX:G1SATBBufferEnqueueingThresholdPercent=30 -XX:G1ConcMarkStepDurationMillis=5 -XX:G1ConcRSHotCardLimit=16 -XX:G1ConcRefinementServiceIntervalMillis=150`

### Garbage Collection Threading

`-XX:ConcGCThreads=[Some Number]` controls the [*maximum* number](https://github.com/openjdk/jdk/blob/dd34a4c28da73c798e021c7473ac57ead56c9903/src/hotspot/share/gc/z/zHeuristics.cpp#L96-L104) of background threads the garbage collector is allowed to use, and defaults to `logical (hyperthreaded) cores / 4`. Recent versions of Java will [reduce the number of gc threads, if needed](https://wiki.openjdk.org/display/zgc/Main#Main-SettingConcurrentGCThreads).

In some cases (especially with ZGC or Shenandoh) you want to increase this thread cap past the default. I recommend "2" on CPUs with 4 threads, and `[number of real cores - 2]` on most other CPUs, but you may need to play with this parameter. If its too low, garbage collection can't keep up with Minecraft, and the game will stutter and/or start eating gobs of RAM and crash. If its too high, you might waste CPU on GC, especially if you are running Java 8. 


Large Pages
======

Enabling large pages improves the performance of Minecraft servers and clients. Here are some great tutorials for enabling it:

- Windows 10 Pro: https://www.chaoticafractals.com/manual/getting-started/enabling-large-page-support-windows
- Windows 10 Home: https://awesomeprojectsxyz.blogspot.com/2017/11/windows-10-home-how-to-enable-lock.html?m=1
- Tool mirrors for the W10 Home tutorial: https://gist.github.com/eyecatchup/0107bab3d92473cb8a3d3547848fc442
- Linux: https://kstefanj.github.io/2021/05/19/large-pages-and-java.html


On Windows, you **must** run java, and your launcher, as an administrator. That means checking the ["run as administrator" compatibility checkbox](https://support.sega.com/hc/en-us/articles/201556551-Compatibility-Mode-and-Running-as-Administrator-for-PC-Games) for `javaw.exe`, `java.exe` and `your launcher.exe`, otherwise Large Pages will silently fail. Add `-XX:+UseLargePages -XX:LargePageSizeInBytes=2m` to your arguments.  

On linux, you generally want to use `-XX:+UseTransparentHugePages`. To manually allocate memory instead (for a bigger performance boost), Red Hat has a good tutorial for RHEL-like linux distros, like Fedora, CentOS, or Oracle Linux: https://www.redhat.com/en/blog/optimizing-rhel-8-run-java-implementation-minecraft-server 

Check and see if large pages is working with the `-Xlog:gc+init` java argument. 


GraalVM Enterprise Edition
======

GraalVM is a new high performance Java VM from Oracle that can improve the performance of (modded) Minecraft. While client FPS gains are modest, server-side workloads like chunk generation can get a 20%+ boost!

Unfortunately, only GraalVM Enterprise Edition comes with the full set of optimizations, and downloading it requires making a free Oracle account.

Register and download it here: https://www.oracle.com/downloads/graalvm-downloads.html

Grab the newest "Oracle GraalVM Enterprise Edition Core" release available for Java 17+ (or Java 11 if you need it) from the "Archives" section. Unzip it, and put the unzipped folder somewhere safe.

Again, you *must* use 22.1.0, not 22.2.0.

These releases are not Java installers. You need to manually replace your launcher's version of Java, or use a Minecraft launcher that supports specifying your Java path. I recommend PolyMC, ATLauncher, or GDLauncher. When specifying a java path, navigate to the "bin" folder in the GraalVM download and use "javaw.exe" or "java.exe"

For servers, you need to replace the "java" command in your server start sh/bat file with the full path to graalvm java, in quotes.

If you don't feel comfortable making an Oracle account, grab the latest [GraalVM CE release](https://github.com/graalvm/graalvm-ce-builds/releases) and use the flags from above ^. But Oracle does not check the information you put into the registration page, and GraalVM CE lacks most of the EE optimizations.

Apple Silicon Mac Users: For now, you must use GraalVM 22.2.0 and add the following argument: `-Dgraal.PartialEscapeAnalysis=false`

GraalVM EE Java Arguments
======

Arguments for GraalVM EE 22+ Java 17 (or Java 11):

```-XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+AlwaysPreTouch -XX:+DisableExplicitGC -XX:+UseNUMA -XX:AllocatePrefetchStyle=3 -XX:NmethodSweepActivity=1 -XX:ReservedCodeCacheSize=400M -XX:NonNMethodCodeHeapSize=12M -XX:ProfiledCodeHeapSize=194M -XX:NonProfiledCodeHeapSize=194M -XX:-DontCompileHugeMethods -XX:MaxNodeLimit=240000 -XX:NodeLimitFudgeFactor=8000 -XX:+UseVectorCmov -XX:+PerfDisableSharedMem -XX:+UseFastUnorderedTimeStamps -XX:+UseCriticalJavaThreadPriority -XX:+EnableJVMCIProduct -XX:+UseJVMCICompiler -XX:+EagerJVMCI -Dgraal.TuneInlinerExploration=1 -Dgraal.CompilerConfiguration=enterprise -Dgraal.UsePriorityInlining=true -Dgraal.Vectorization=true -Dgraal.OptDuplication=true -Dgraal.DetectInvertedLoopsAsCounted=true -Dgraal.LoopInversion=true -Dgraal.VectorizeHashes=true -Dgraal.EnterprisePartialUnroll=true -Dgraal.VectorizeSIMD=true -Dgraal.StripMineNonCountedLoops=true -Dgraal.SpeculativeGuardMovement=true -Dgraal.InfeasiblePathCorrelation=true -Dgraal.LoopRotation=true```

**You must add G1GC arguments to these arguments.** GraalVM currently doesn't work with ZGC or Shenandoah.  


GraalVM EE Mod Compatibility
======
There are a few client-side rendering bugs specific to GraalVM:

- GraalVM EE 22.2.0 has issues with Minecraft, particularly with the `UsePriorityInlining` flag enabled. Please use 22.1.0 until 22.3.0 is out. See: https://github.com/oracle/graal/issues/4776

- `VectorizeSIMD` turns entities invisible with Iris or Occulus... but only under certain conditions. This will be fixed in GraalVM EE 22.3.0. See: https://github.com/oracle/graal/issues/4849

- GraalVM CE and EE both break constellation rendering in 1.16.5 Astral Sorcery. See: https://github.com/HellFirePvP/AstralSorcery/issues/1963

I have not observed any server-side bugs yet. 

If you run into any other mod issues you can trace back to GraalVM, please create a Github issue or post in the Discord! Generally, you can work around them by disabling the `dgraal` flags one at a time, or by finding the right function with `Dgraal.PrintCompilation=true`, and working around it with `-Dgraal.GraalCompileOnly=~...` once you find the miscompiled function.

SpecialK
======
SpecialK has 2 major Windows performance benefits:

- A "smart" frame limiter that reduces stutter, eliminates tearing, saves power, and saves CPU TDP to boost when needed. It even works in conjuction with VRR or Nvidia Reflex. 

- A OpenGL-to-DirectX11 wrapper called OpenGL-IK that eliminates Minecraft's windowed mode overhead, and enables other features (like auto-HDR).

Download it here: https://wiki.special-k.info/en/SpecialK/Tools

Add your MC launcher, and check the "elevated service" checkbox. Then navigate to your java bin folder where your javaw.exe is, and create an empty file called `SpecialK.OpenGL32`. Launch your Minecraft launcher with the SpecialK launcher, and the launcher will then "inject" SpecialK into Minecraft.
![SpecialK](Tutorial_Images/specialk.PNG)

You can create a desktop shortcut to your Minecraft launcher through the SpecialK UI for even more convenience. 

Be sure to turn off VSync and the in-game Minecraft frame limiter.


Process Priority
======
After launching Minecraft, set Java to run at a high process priority in Windows with the task manager:

![taskmanager](Tutorial_Images/taskmon.PNG)

Linux users can append the command `sudo nice -n -15` to thier launch arguments.


Performance Mods
======

This is a **fantastic** repo for finding performance mods: https://github.com/NordicGamerFE/usefulmods


Java 8
======

I recommend using Java 17 or, failing that, Java 11 unless 8 is absolutely necessary. But if it is, these flags will work with OpenJDK8, along with Shenandoh GC (for Red Hat OpenJDK on clients) or G1GC (for everything else):

```-XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+ParallelRefProcEnabled -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -XX:+PerfDisableSharedMem -XX:+AggressiveOpts -XX:+UseFastAccessorMethods -XX:MaxInlineLevel=15 -XX:MaxVectorSize=32 -XX:+UseCompressedOops -XX:ThreadPriorityPolicy=1 -XX:+UseNUMA -XX:+UseDynamicNumberOfGCThreads -XX:NmethodSweepActivity=1 -XX:ReservedCodeCacheSize=350M -XX:-DontCompileHugeMethods -XX:MaxNodeLimit=240000 -XX:NodeLimitFudgeFactor=8000 -XX:+UseFPUForSpilling -Dgraal.CompilerConfiguration=community```

x86 Java 8 users (aka most Java 8 users) can add these additional arguments:

```-XX:+UseXMMForArrayCopy```

You can also get Java 8 versions of GraalVM EE from the [21.X section on the Oracle site](https://www.oracle.com/downloads/graalvm-downloads.html), and use these arguments:

```-XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+ParallelRefProcEnabled -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -XX:+AggressiveOpts -XX:+UseFastAccessorMethods -XX:AllocatePrefetchStyle=1 -XX:ThreadPriorityPolicy=1 -XX:+UseNUMA -XX:+UseDynamicNumberOfGCThreads -XX:NmethodSweepActivity=1 -XX:ReservedCodeCacheSize=350M -XX:-DontCompileHugeMethods -XX:MaxNodeLimit=240000 -XX:NodeLimitFudgeFactor=8000 -XX:+UseFPUForSpilling -XX:+EnableJVMCI -XX:+UseJVMCICompiler -XX:+EagerJVMCI -Dgraal.TuneInlinerExploration=1 -Dgraal.CompilerConfiguration=enterprise -Dgraal.UsePriorityInlining=true -Dgraal.Vectorization=true -Dgraal.OptDuplication=true -Dgraal.DetectInvertedLoopsAsCounted=true -Dgraal.LoopInversion=true -Dgraal.VectorizeHashes=true -Dgraal.EnterprisePartialUnroll=true -Dgraal.VectorizeSIMD=true -Dgraal.StripMineNonCountedLoops=true -Dgraal.SpeculativeGuardMovement=true -Dgraal.InfeasiblePathCorrelation=true```



Other Performance Tips
======

- Run your Minecraft servers on Clear Linux! It's by far the most optimized linux distribution out-of-the-box, and it has some other nice features (like a stateless config system). It also runs clients on AMD/Intel GPUs quite well: https://docs.01.org/clearlinux/latest/tutorials/multi-boot/dual-boot-win.html

- Make sure the Minecraft client is using your discrete GPU! Check the F3 tab, and force Minecraft to use it in the "**Windows Graphics Settings**", *not* the AMD/Nvidia control panel (as they don't seem to work anymore).

- Minecraft client linux users should check out https://github.com/Admicos/minecraft-wayland 

- Close everything in the background, including Discord, game launchers and your browser! Minecraft is resource intensive, and does not like other apps generating CPU interrupts or eating disk I/O, RAM and so on.  

FAQ
======

- Java 18 is has some incompatibilities. At this time, I recommend using 17 over 18. 

- IBM's OpenJ9 is over 30% slower at server chunkgen in my tests. If there are any flags that make it competitive with OpenJDK, please let me know on Discord or here: https://github.com/brucethemoose/Minecraft-Performance-Flags-Benchmarks/issues/9


Flag Explanations
======
- Aikar G1GC flags are explained here: https://aikar.co/2018/07/02/tuning-the-jvm-g1gc-garbage-collector-flags-for-minecraft/
- `-XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions` simply unlock more flags to be used. These can be listed with the `-XX:+PrintFlagsFinal` and `-XX:+JVMCIPrintProperties` flags, see [Flag Dumps](Flag_Dumps)
- `-XX:+UseNUMA` enables optimizations for multisocket systems, if applicable. Not sure if this applies to MCM CPUs like Ryzen or Epyc. 
- `-XX:-DontCompileHugeMethods` *Allows* huge methods to be compiled. Modded Minecraft has some of these, and we don't care about higher background compiler CPU usage.
- `-XX:MaxNodeLimit=240000 -XX:NodeLimitFudgeFactor=8000` Enable optimization of larger methods. See: https://bugs.java.com/bugdatabase/view_bug.do?bug_id=8058148
- `-XX:ReservedCodeCacheSize=400M -XX:NonNMethodCodeHeapSize=12M -XX:ProfiledCodeHeapSize=194M -XX:NonProfiledCodeHeapSize=194M` reserves more space for compiled code. I observed modded Minecraft run over the default 250 megabyte limit with `XX:+PrintCodeCache`, but even if its not filled, the larger size makes eviction of compiled code less aggressive. 
- `-XX:NmethodSweepActivity=1` (default 10) keeps "cold" code in the cache for a longer time.  
- ~~`-XX:+UseStringDeduplication`~~ This is a popular option, but not used here, as it's benching slower. Maybe its useful on low memory systems?  
- `-XX:+UseFastUnorderedTimeStamps` Avoid system calls for getting the time. The impact of this will vary per system, but we aren't really concerned with logging timestamp accuracy. 
- `-XX:+UseCriticalJavaThreadPriority` *Nothing* should preempt the Minecraft threads. GC and Compiler threads can wait. 
- `-XX:ThreadPriorityPolicy=1` Use a wider range of thread priorities. Requires sudo on linux to work. Some JDKs (like Graal) enable this by default, but some don't.
- `-XX:G1SATBBufferEnqueueingThresholdPercent=30 -XX:G1ConcMarkStepDurationMillis=5 -XX:G1ConcRSHotCardLimit=16 -XX:G1ConcRefinementServiceIntervalMillis=150`: Optimizes G1GC's concurrent collection, still being tested: https://research.spec.org/icpe_proceedings/2014/p111.pdf
- `-XX:G1RSetUpdatingPauseTimePercent=0`: We want *all* this work to be done in the G1GC concurrent threads, not the pauses. 
- `-XX:G1HeapWastePercent=18` Don't bother collecting from old gen until its above this percent. This avoids triggering slower "mixed" young generation GCs, which is fine since Minecraft (with sufficient memory) doesn't fill the old gen that fast. Idea from: https://www.reddit.com/r/Minecraft/comments/k9zb7m/tuning_jvm_gc_for_singleplayer/
- `-XX:GCTimeRatio=99` As a goal, 1% of CPU time should be spent on garbage collection. Default is 12, which seems way too low. 
- `-XX:AllocatePrefetchStyle=3` Generate one prefetch instruction per cache line instead of per allocation. It seems to break ZGC.
- `-Dgraal.LoopRotation=true` A non default optimization, will probably be default soon. 
- `-Dgraal.TuneInlinerExploration=1` Spend more time making inlining decisions. For Minecraft, we want the C2 compiler to be as slow and aggressive as possible. 
- Most other `-Dgraal` arguments are enabled by default, and are either there as a sanity check, for debugging or as a failsafe (if, for instance, someone unknowingly disables JVCMI with some other flag). 
- Many Java 8 flags (such as `-XX:MaxInlineLevel=15 -XX:MaxVectorSize=32`) are just copied from the Java 17 defaults. Others (like `+AggressiveOpts`) are only non-default in some older Java 8 builds. 

Flags Under Consideration:
======
- More aggressive inlining, via `-Dgraal.BaseTargetSpending=160` (default 120) in Graal and some other flags in OpenJDK.
- OpenJDK flags which are disabled by default: `-XX:+AlignVector -XX:+OptoBundling -XX:+OptimizeFill -XX:+AlwaysCompileLoopMethods -XX:+EnableVectorAggressiveReboxing -XX:+EnableVectorSupport -XX:+OptoScheduling -XX:+UseCharacterCompareIntrinsics -XX:+UseCopySignIntrinsic -XX:+UseVectorStubs`
- ~~`-Dgraal.LSRAOptimization=true`~~ seems to hurt performance 
- `-Dgraal.OptWriteMotion=true` and `graal.WriteableCodeCache=true`, which *do not* seem stable, but may be more stable in GraalVM 22.3.0 
- Extreme `G1HeapWastePercent` values.
 
Sources
======
- Updated Aikar flags from this repo: https://github.com/etil2jz/etil-minecraft-flags
- Reddit post from a Forge dev: https://www.reddit.com/r/feedthebeast/comments/5jhuk9/modded_mc_and_memory_usage_a_history_with_a/
- Red Hat's optimization guide: https://www.redhat.com/en/blog/optimizing-rhel-8-run-java-implementation-minecraft-server
- GraalVM release notes: https://www.graalvm.org/release-notes/
- Oracle's Java 17 Documentation: https://docs.oracle.com/en/java/javase/17/docs/specs/man/java.html
- VM Options explorer: https://chriswhocodes.com/
- Java itself, via the `-XX:+PrintFlagsFinal` and the `-XX:+JVMCIPrintProperties` flags to dump flag descriptions/defaults. 
- OpenJDK source: https://github.com/openjdk/jdk/
- Testing from @keyboard.tn in Discord.
- https://research.spec.org/icpe_proceedings/2014/p111.pdf
- https://www.diva-portal.org/smash/get/diva2:1466940/FULLTEXT01.pdf
- https://malloc.se/blog/zgc-jdk17
- https://docs.oracle.com/javase/8/embedded/develop-apps-platforms/codecache.htm
