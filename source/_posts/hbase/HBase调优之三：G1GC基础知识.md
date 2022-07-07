---
title: HBase调优之三：G1GC基础知识
date: 2022-03-28 16:52:38.566
updated: 2022-03-28 16:52:38.566
url: /archives/basicg1gc
categories: 
- HBase调优
tags: 
- hbase
- 性能调优
description: HubSpot engineering is a invested heavily in microservices and continuous deployment. Java is not only used to run our thousands of deployables, but also our queues
---

## G1基础

<!--more-->

HubSpot engineering is a [Java shop](https://product.hubspot.com/blog/modern-java-at-hubspot) invested heavily in microservices and continuous deployment. Java is not only used to run our thousands of deployables, but also our queues ([Kafka](https://product.hubspot.com/blog/kafka-at-hubspot-part-1-critical-consumer-metrics)) and Big Data solution (HBase). Keeping all these JVMs performant while providing a good user experience has forced us to dig deep into Garbage Collection (GC), particularly the Garbage First Garbage Collector (G1GC).

## Motivation


Interestingly enough, the initial motivation for monitoring GC performance was not the elimination of performance issues. At the time, the goal was to shrink the overall RAM footprint of our applications to reduce the number of servers and save $$. The responsible way to shrink heap involves monitoring GC performance to know how low to go. Representative metrics were identified and collected out of the GC logs, and we started experiments to reduce the heaps of larger applications.

GC metrics were not even two weeks old before the first performance issue was found. Out of a cluster of application instances, one would randomly start to have response times 2x the norm or higher. With a bit of poking around it was determined the new metric for overall time spent in GC correlated with the slow response times. With the y-axis in millis, the graph below shows a single instance start spending 25s of wall clock time in GC for each reporting interval. At 60s apart, 25s/60s or 40% of the reporting interval the JVM was in GC and not doing any useful work.

![G1GC1.png](https://product.hubspot.com/hs-fs/hubfs/G1GC1.png?width=672&height=404&name=G1GC1.png)

The issue was successfully resolved through GC tuning alone which at the time was both shocking and depressing, as it more or less ruled out the idea that GC "just worked". The above situation is representative of the "too much time spent in GC" scenario, and below we have the "individual latency too high" scenario. Here again the y-axis is millis and the metric being graphed is the longest individual GC time per reporting interval. This REST API application is regularly pausing for 10s+. For reference, the memcached operation timeout is set to 1s and front end requests timeout after 5s.

![G1GC2.png](https://product.hubspot.com/hs-fs/hubfs/G1GC2.png?width=672&height=428&name=G1GC2.png)

### Poor User Experience

The types of GC problems that directly affect our customers fall into the following three categories.

- Extended periods of slow response time (too much time spent in GC)
- Random high latency spikes 5s+ (individual long GC time)
- Traffic based OOMs (Out Of Memory)

The first two categories have been described already and are relatively straightforward. The last category, traffic based OOMs, involves a grey line and a fair bit of handwaving. OOM failures happen in Java when the JVM does not have enough heap to cover the amount of live data currently being used by the application. By labeling some OOMs as traffic based, we are attempting to differentiate between apps that are hopelessly misconfigured and consistently crash after some short period of time vs. those that run fine for long periods of time (days) yet OOM randomly due to traffic conditions. The OOMs in the former camp can be eliminated with more heap, those in the latter camp can be reduced or eliminated through GC tuning.

### Actions

Over the past few months tackling GC issues we've taken the following actions.

- Collect GC metrics for all applications
- Setting safe GC parameter defaults
- Set alerts to detect when apps are in GC Hell
- Define and detect requests of unusual size (R.O.U.S.)
- Dive deeper into G1GC
- Introduce G1GC lectures and training for all backend devs
- Create a tool for fine grained G1GC log analysis ([gc_log_visualizer](https://github.com/HubSpot/gc_log_visualizer))
- Tune REST API instances
- Tune Zookeeper
- Tune Kafka
- Tune HBase

Now that the dust has settled and HubSpot has a good handle on GC, and G1GC in particular, we’re ready to share our hard earned lessons with the following information on the nuts and bolts of G1GC.

## Topics and Overview

The goal of the following information is to provide the necessary G1GC knowledge to effectively monitor, troubleshoot and tune any given G1GC installation (of Oracle's JDK versions 1.8.45-1.8.65, other versions may/will vary subtly).

**The topics will cover broad foundational principles as well as detailed looks at individual features, starting with more theory and ending with more details.**

**Jump to:**

1. [Concepts and Architecture](https://product.hubspot.com/blog/g1gc-fundamentals-lessons-from-taming-garbage-collection#ConceptsArchitecture)
2. [Regions](https://product.hubspot.com/blog/g1gc-fundamentals-lessons-from-taming-garbage-collection#Regions)
3. [Starting to Tie it All Together](https://product.hubspot.com/blog/g1gc-fundamentals-lessons-from-taming-garbage-collection#TieItTogether)
4. [Concepts and Architecture Continued](https://product.hubspot.com/blog/g1gc-fundamentals-lessons-from-taming-garbage-collection#ConceptsArchitectureCont)
5. [Understanding Mixed Events](https://product.hubspot.com/blog/g1gc-fundamentals-lessons-from-taming-garbage-collection#MixedEvents)
6. [Humongous Objects](https://product.hubspot.com/blog/g1gc-fundamentals-lessons-from-taming-garbage-collection#HumongousObjects)
7. [More Concepts and Final Thoughts](https://product.hubspot.com/blog/g1gc-fundamentals-lessons-from-taming-garbage-collection#FinalThoughts)



## Concepts and Architecture

G1GC (Garbage First Garbage Collector) is the low latency garbage collection algorithm included in recent versions of both OpenJDK and Oracle Java. Like other Java GC algorithms, to reclaim heap space G1GC must halt all application threads, a process referred to as stopping-the-world (STW) or pausing (a GC pause). There are two main STW characteristics that affect the applications running on a JVM, the duration of individual pauses, and the overall time spent in STW.

G1GC is geared towards consistently short STW times and will naturally spend more overall time in GC than the other most popular collector, ParallelGC.

A lot more can be written on the uses and differences of G1GC and ParallelGC than will be included here. At HubSpot, we use G1GC for any JVM that is in the path of a user's browser, and ParallelGC for everything else. In effect we are saying user experience will be better with consistently fast requests, while async processing throughput will be better/cheaper with ParallelGC.

This initial section will introduce a few of the core concepts and architectural decisions present in many modern day garbage collectors, as well as implementation details specific to G1GC.

- The GC Epoch
- What it means to be a Generational Collector
- The significance of G1GC's regions
- How Evacuation style collectors work and how they defragment heap
- How Tenured space can be collected in subsets instead of all at once
- Humongous objects, the downside of G1GC's regions
- G1GC's ability to resize the heaps each epoch
- The significance of Free space

***\*
\****

### The GC Epoch

A full circuit of the garbage collection life cycle is called an epoch. The following diagram highlights the events of concern to this document.

![G1GC3.png](https://product.hubspot.com/hs-fs/hubfs/G1GC3.png?width=588&height=472&name=G1GC3.png)

- App Threads Started - This event triggers the start of the application code after which real work gets done.
- GC Triggered - Something triggers the need to reclaim unused heap space.
- App Threads Stopped - Application threads need to stop at a java safepoint in order for heap space to be safely reclaimed. Real work stops here.
- Heap Reclaimed - The GC algorithm determines what occupied space is no longer being used and reclaims that space.
- Heap Sizes Adjusted - If heap spaces are adjustable, they will be adjusted based on current conditions.

The application threads are started in the 'App Threads Started' state, at which point the JVM begins running the application code and does what would be considered useful work. Eventually GC is triggered through an event such as a heap filling up and the JVM halts all the application threads in order to reclaim no longer used heap space. After the heap is reclaimed the heap sizes may be adjusted, then the application threads are started once more.

### Generational Collector

Generational collectors are based on the following statements both being mostly true:

- Data that is very young will not survive long
- Data that is old will continue to exist

Consider a simple Java web server application as it just starts up. The init sequence will load the logging mechanism, the connection pools, perhaps pre-initialize some data/caches, determine what endpoints it will accept and so on. Post warm up, the application receives and fields a never-ending supply of REST calls. Each REST call may include decoding/parsing the http request into Request/Response objects, DB/memcached calls, marshalling of data into json and finally streaming the data through the Response object. One set of data will hang around for the life of the application server, the other will exist for a brief instant.

Old and Young generations are designed specifically to handle those two types of data based on their longevity characteristics. The connection pools, logging metadata, endpoint information and caches would exist for long periods of time in the Old generation, while shorter lived transient data from REST API calls would briefly reside in the Young generation.

The usage pattern of the generational collector is to allocate all new objects in the Young generation and promote objects that live long enough into the Old generation. G1GC has two types of heap in the Young generation, Eden and two Survivor spaces labeled To & From. The Survivor spaces exist in order to weed out transient data that manages to survive through an epoch cycle or two. The Old generation in G1GC is comprised of a single heap space named Tenured, and the two names (Old/Tenured) are often used interchangeably.



## Regions

The defining feature of G1GC is the region, a small independent heap. G1GC divides the total heap into homogenous regions that are logically combined into the traditional heaps Eden, Survivor and Tenured. These multiple smaller heaps cost more in overhead (cpu/ram) but are quite flexible and provide the following benefits:

***\*
\****

- Allows Tenured to be collected in portions, which caps latency
- Allows generations to be easily resized as necessary
- Auto-defragging with evacuation-style collections

***\*
\****

The following illustration shows 64 regions logically grouped into Eden (E), Survivor spaces (S) and Tenured (T). The remaining empty regions are considered Free space.

![G1Gc4.png](https://product.hubspot.com/hs-fs/hubfs/G1Gc4.png?width=467&name=G1Gc4.png)Region sizes are not changeable and must be a power of two between 1MB - 32MB, inclusive. If not explicitly set on the command line via -XX:G1HeapRegionSize=#m, the region size will be set such that there are the optimal 2k or more regions on startup. The heap size of the JVM on startup is the minimum heap, so to keep things simple HubSpot always sets min heap to be the same as max heap when using G1GC. The following table shows the region size that will be chosen based on the minimum heap size should the region size not be explicitly set.

***\*
\****

| **Min Heap Size**   | **Region Size** |
| ------------------- | --------------- |
| heap < 4GB          | 1MB             |
| 4GB <= heap < 8GB   | 2MB             |
| 8GB <= heap < 16GB  | 4MB             |
| 16GB <= heap < 32GB | 8MB             |
| 32GB <= heap < 64GB | 16MB            |
| 64GB <= heap        | 32MB            |

***\*
\****

### Object allocation > Region

The primary pain point to the breaking apart of heap into isolated Regions is the large allocation, where large is relative to the Region size. How should G1GC handle an allocation that is 3x larger than a Region itself? G1GC names these problem allocations Humongous objects, and it must be noted that a Humongous object is single allocation such as a byte[30 * 1024 * 1020] and not a String[1024] pointing to String objects of length 30k each.

A few key points about Humongous objects:

- Humongous object size ≥ (G1HeapRegionSize/2)
- They’re allocated in contiguous regions of Free space
- They’re instantly added to Tenured

### Evacuation style collections

In G1GC reclaiming space is done by copying live data out of existing regions into empty regions. The regions the data came from are considered empty at the end of the process and will become Free space. This evacuation process will naturally defragment as data is continually consolidated into empty regions.

![G1GC5.png](https://product.hubspot.com/hs-fs/hubfs/G1GC5.png?width=671&height=371&name=G1GC5.png)

### 4 types of heap space

Harkening back to the illustration of regions, we end up with 4 types of heap space to be concerned with. Eden, Survivor, Tenured and the regions not part of the other three (Free).

***Eden\***

- Consists of objects allocated since the current epoch started
- Resized for each epoch to be between -XX:G1NewSizePercent (default 5) and -XX:G1MaxNewSizePercent (default 60)
- All new objects are allocated in Eden, except Humongous objects
- Empty at the beginning of each epoch

***Survivor\***

- Survivor From consists of objects that have survived at least one epoch
- Survivor To is allocated but empty during the epoch
- Each object in survivor has a counter for the number of epochs survived
- Objects surviving long enough get promoted to Tenured
- Survivor To space is resized as a ratio (-XX:SurvivorRatio, default 8) of the current Eden size

***Tenured\***

- Consists of Working set + Humongous objects + fragmentation (Reclaimable and Un-reclaimable heap waste)
- Garbage-collected in batches during qualifying epochs

***Free\***

- Consists of regions not allocated to any logical heap
- Humongous objects are allocated from Free regions, which then become Tenured



## Starting to Tie it All Together

The initial section introduced the foundational concepts of G1GC, this section will switch gears and focus on how the pieces work together and become more than the sum of their parts. Walking through the simplest epoch of a REST application server will provide insight into:

- Why G1GC's approach to the Young generation is efficient for transient data
- How and why Survivor space is useful
- The usefulness of evacuation
- How regions are used
- The driving force behind STW times

### Epoch lifecycle of a Minor GC event

The vast majority of G1GC use cases at HubSpot are REST API applications that talk to multiple data sources and communicate via json objects. For this example assume traffic volume around 100 requests/second with individual requests averaging 50ms.

 

#### 10,000ft View

The basic mechanics of an epoch ending in a Minor GC event are summed up in the following illustration. Eden starts empty and is filled with new data allocations. Eden fills up triggering the end of the epoch and the Evacuation phase. The live data in Eden and Survivor From is moved (evacuated) into Survivor To space, with the exception of any data promoted from Survivor From to Tenured. After the Evacuation phase, Eden and Survivor From are devoid of live data and reclaimed.

![G1GC6.png](https://product.hubspot.com/hs-fs/hubfs/G1GC6.png?width=550&height=1557&name=G1GC6.png) 

#### Details

The epoch starts with Eden and Survivor To space regions allocated and empty. Survivor From space contains some objects tagged with their survival counts. The useful work the application threads are doing is gradually filling Eden's regions with new data. At some point, Eden runs out of available space and an allocation request fails. Application threads will be stopped at their next safe point, such as an allocation request.

At this point the clock is ticking as no useful work is being done. A number of GC worker threads (-XX:ParallelGCThreads) begin concurrently scanning through all the places heap pointers can exist (threads/stacks/registers) for pointers to live objects in the heap.

Objects traced back to...

- ...Tenured: are ignored

- ...Eden: are evacuated into Survivor To space with survival counter 1

- ...From, and

- - Survival counter + 1 > tenuring threshold: are promoted to Tenured
  - Survival counter + 1 <= tenuring threshold: are evacuated into Survivor To space with survival counter incremented

***\*
\****

Evacuation includes both copying an object into a new region and updating all pointers to that object. Once scanning has completed and all the live objects have been evacuated, by definition all the objects left in Eden and Survivor From space are not referenced, are not live and can be safely reclaimed as Free space.

Survivor To is renamed to Survivor From. Eden's size is recalculated and regions are allocated from Free space. The new Survivor To is calculated off of Eden's new size and again regions are allocated. The application threads are then allowed to continue from where they left off.

 

#### To-space overflow and To-space exhaustion

To-space overflow happens when the Survivor To space cannot accommodate all the surviving data from Eden and the Survivor From space. When this happens the overflow data is added to Tenured in regions pulled from Free space. Should there not be enough available regions in Free space for this, To-space exhaustion occurs. Recovering from To-space exhaustion involves rolling back all evacuations up to that point and initiating a Full GC.

 

#### Survivor space

At the end of the epoch, Survivor space is filled with objects like these:

- Transient request data from active http requests
- Recently cached data
- Changes/updates to long lived objects (eg conn pools)

Because GC events are triggered by activity (active threads servicing http requests), we expect there will always be data of the first category in each epoch. The goal of Survivor space is to keep that transient data from making it into Tenured, where it would be expensive to remove. Having the other two categories of data bounce back and forth a few times between Survivor spaces is a cost associated with keeping transient data out of Tenured.

 

#### Length of STW

In this example, actions taken while the application threads were stopped include:

- Scanning stack/application threads/registers et al
- Copying objects from one region to another
- Updating pointers

Scanning stack regions and updating pointers are probably not time intensive operations, however copying an object from one memory location to another (memcpy) can take some time. So it ends up that the driver of STW time is the size and amount of objects that are copied. The size and amount of objects being copied, if extrapolated from the previous Survivor space section, is driven by the number and activity level of http requests that are active at the point in time that GC is triggered.

Now this is an interesting piece of data. Consider that most http based applications servers will receive traffic at a relatively even distribution over a given period of time. For example, during a five minute period, an app server could field an average of 100 requests/sec, with each request averaging 50ms. The average concurrency (active threads processing in parallel) for this period of time would be 5. So at any given point in time during the 5 minute period, we'd expect a Minor GC event to take however long it takes to copy the transient data for those 5 active and concurrent http requests.

 

#### Effects of Eden size

Given that the STW time is driven by the concurrency of the http application and that GC events are 30s apart, what would happen if we grew or shrunk the size of Eden? With the STW time fixed against the concurrency, doubling the size of Eden will result in GC events happening every 60s instead of every 30s, cutting the frequency of total GC events and probably overall time spent in GC in half. The opposite would happen by shrinking the size of Eden by 50%, GC events would come twice as frequently, and we'd be spending twice the amount of overall time in STW.

At HubSpot we have found it to be a universal truth* for our REST API instances that the more Eden we can allocate the better. Individual STW times are unaffected, and overall time spent in STW drops.

*Caveat: On a couple of rare occasions increasing Eden bumped up STW times, but dropped back down to the expected level after enabling parallel reference processing via -XX:+ParallelRefProcEnabled.

 

#### Takeaways (for REST API servers)

- Survivor spaces are buffers to keep transient data out of Tenured
- Regions + Evacuation == defragmentation
- Stw times are driven by the copying of objects, and are thus driven by the amount and size of data held by each http request x concurrency
- Doubling Eden will halve the overall time spent in GC
- Eden size doesn’t affect individual STW times*

*HBase instances with heavy block cache churn are an exception, as would be any other data source with a constantly refreshing large cache. On the other hand, we found that both Kafka’s and ZooKeeper’s individual STW times were not affected by Eden size.



## Concepts and Architecture Continued...

Running through the simplest G1GC epoch has hopefully provided enough context that pieces are beginning to click into place. Next up are the different types of GC events along with how they’re triggered.

### Types and Triggers of GC Events

During normal operation GC is triggered through need/activity, as in an application thread needs space and none is available. External forces can instigate Full GC events, commonly through jcmd and jmap for diagnostic purposes. Common triggers are:

- Eden full
- Free space cannot accommodate a Humongous object
- Humongous object allocated successfully and GC event conditions met
- Externally initiated (jcmd, jmap, Runtime.gc())

The happy path here is for GC events to only be triggered when Eden is full or a Humongous object is allocated successfully and certain conditions are met. This happy path involves only Minor and Mixed events. Full GC events are likely to exceed the max desired STW and need to be avoided. Humongous objects are often a liability as lots of Humongous objects will increase the likelihood of running out of Free space, which would trigger a Full GC.

The types of GC events are as follows

- Minor: Eden + Survivor From -> Survivor To
- Mixed: Minor + (# reclaimable Tenured regions / -XX:G1MixedGCCountTarget) regions of Tenured
- Full GC: All regions evacuated
- Minor/Mixed + To-space exhaustion: Minor/Mixed + rollback + Full GC

In a smoothly running application, batches of Minor events alternate with batches of Mixed events. Full GC events and To-space exhaustion are things you absolutely don't want to see when running G1GC, they need to be detected and eliminated.

For http applications and many other types of applications, Evacuation of Eden (Minor event) is strictly a function of concurrency and request data sizes. In other words, to reduce Minor STW times, either reduce data loaded/munged/sent or reduce concurrency.

So for maintaining short STW times, the variable factor is the reclaiming of Tenured space. G1GC's approach is to reclaim Tenured in bite sized chunks, and has a whole host of support levers allowing fine grained control.



## Understanding Mixed Events

This next section is focused on how Tenured is cleaned and will provide insight into:

- The conditions necessary to start a Mixed event
- The conditions necessary to end a Mixed event cycle
- Why Mixed events come in batches
- Why we need to know the working set size
- How to-space exhaustion could occur

 

#### 10,000ft View

At the end of each Minor event, a check is made to determine if it's time to consider cleaning Tenured. If the check passes, a Multi-Phase Concurrent Marking Cycle (MPCMC) is kicked off. As per its name, the MPCMC will run concurrently with the application threads to produce a set of liveness metadata (amount of reclaimable space) for each Tenured region. At the beginning of the next GC event following the completion of the liveness metadata, a different check is made against the amount of reclaimable data. If there is not enough reclaimable data, the liveness metadata is thrown away and the event is a Minor one. If there is enough reclaimable data, the GC event is a Mixed event. The subsequent GC events will also notice the presence of the liveness metadata and decide whether or not to be Mixed or Minor based on how much reclaimable Tenured space remains, with any decision to run a Minor event clearing the liveness metadata.

 

#### Detailed Walkthrough

Mixed GC events can only occur when the liveness metadata exists. The liveness metadata is generated by a run of the MPCMC. The MPCMC is initiated at the end of a Minor event, which is where we will start.

A Minor event is in progress, the Eden/From spaces have been emptied and resized and the application threads are still stopped. At this point, G1GC will make a check to determine if the MPCMC should be kicked off or not:

*100(Heap currently used Total available heap) > -XX:InitiatingHeapOccupancyPercent*

Since the check is made after Eden/From have been evacuated, the check is basically asking if Tenured's current size exceeds a configurable threshold (45% by default) of the total heap. If this check passes, a snapshot of the data (threads/stacks/registers/etc) that was used to trace pointers into Eden/Survivor From is taken and given to the MPCMC, which will run on -XX:ConcGCThreads. The application threads are then given the go ahead, and the MPCMC and the application go on about their business in parallel. The MPCMC run times have been observed to range from 50ms to 5s or more. Most apps at HubSpot under 8GB heap usually complete MPCMC runs in 100-800ms.

The MPCMC does work similar to the work done during Minor events, tracing pointers into heap. In this case however the pointers into Tenured are the ones of interest. Each Tenured object that is found this way is marked as live. Again similar to the tracing of objects in heap during the Minor event, objects in Tenured that are not found via tracing are considered unreferenced and not live and can be reclaimed. At the end of the MPCMC each region will have a liveness value, as in what percentage of the total space in the region is live data. Collecting a region with 18% liveness would net 82% of a region added to Free space.

The MPCMC will not be interrupted by Minor GC events, though a Full GC would negate the need for liveness data and abort the MPCMC. After the liveness metadata has been created, the existence of the metadata will be noticed by the next GC event and a Reclaimability check will be made to decide if the GC event should be a Minor or Mixed one. Should the Reclaimability check fail, the metadata is thrown away.

 

Reclaimable

To determine the amount of data that can be reclaimed by collecting Tenured, G1GC starts by creating a list of reclaimable Tenured regions. The list only includes regions with liveness ≤ -XX:G1MixedGCLiveThresholdPercent (default 85), and follows natural ordering based on liveness (ie lowest liveness first). The total bytes that would be reclaimed by collecting just the regions in this list is then tallied, and is divided by the total heap to get a percentage. If that reclaimable percent ≥ -XX:G1HeapWastePercent (default 5), the Reclaimable check passes.

It is interesting here to note that there is a distinction between Reclaimable and unreclaimable heap waste in Tenured. If 20% of the heap consisted of Tenured regions with liveness of 86%, that would be .2 * .14 = .028, or around 3% of the heap that is wasted and unreclaimable.

 

Garbage First

If the check passes, the GC event will be Mixed, and the algorithm will next determine how many and which Tenured regions to collect in addition to Eden/From. Choosing which Tenured regions to collect is how the algorithm got its name, the regions to be collected are the ones with the most garbage in them (least liveness, the regions at the head of the Reclaimable list).

Determining how many regions to collect is a bit more complex and at a high level has three steps involving a floor, an adder and a ceiling.

1. Floor -> Start with the floor by evenly dividing the total work
2. Adder -> Add more regions if time permits
3. Ceiling -> Cap regions with a hard limit of total heap space

The starting number of regions will be:

*length(Reclaimable Region List) -XX:G1MixedGCCountTarget*

Additional Tenured regions will be collected if doing so can be done under the target pause time of -XX:MaxGCPauseMillis. Finally, the number of Tenured regions to be collected will be capped at the -XX:G1OldCSetRegionThresholdPercent (default 10%) percentage of total heap.

 

##### Multiple Mixed Events

After determining the number of Tenured regions to collect, that number of regions will be popped off the front of the sorted Reclaimable Region List and collected along with Eden/From. Since the liveness metadata already exists, there won't be a check to determine if the MPCMC should be kicked off before completing the GC event. The GC event ends and the application threads continue from where they left off.

At the next GC event, once again at the beginning of the event the existence of the liveness metadata is noted, and the Reclaimability check is made. There are differences between this Reclaimability check vs. the previous:

- A bunch of the Tenured regions with the most garbage have already been collected
- There could be newly added Tenured regions without liveness metadata

Should the check succeed again, there will be another Mixed event that will collect some portion of the remaining Tenured regions with the most garbage in them.

At this point, somewhere around 2/G1MixedGCCountTarget of the Reclaimable regions with the most garbage in them have been collected. This process will continue until the amount of reclaimable space as listed in the liveness metadata is less than the allowable waste set by G1HeapWastePercent. At this point the Reclaimability check will fail, the liveness metadata will get tossed, and the next GC event will be a Minor one. Unless G1HeapWastePercent is set to 0, we expect to run fewer consecutive Mixed events than the target count.

### Inherent To-space Exhaustion Vulnerability

Consider the following illustration showing two separate Mixed event cycles.

![G1GC7.png](https://product.hubspot.com/hs-fs/hubfs/G1GC7.png?width=671&height=226&name=G1GC7.png)

The liveness metadata is generated at the tail end of the MPCMC. For the entire period between the end of those MPCMCs, new Tenured regions will not have metadata, and will not be eligible to be reclaimed.

The danger is that during the period in between MPCMCs, Free space gets entirely engulfed into Tenured through to-space overflow or Humongous objects. At HubSpot the underlying cause of to-space exhaustion is usually related to a burst of Requests Of Unusual Size (R.O.U.S.). R.O.U.S. detection will be elaborated on in a future post.

### IHOP and the Working Set

Now that there is some context behind the Mixed event lifecycle, we can dig a little deeper into the repercussions of the InitiatingHeapOccupancyPercent (IHOP) parameter. For an application such as an http REST API server, there is expected to be a relatively stable amount of Tenured data. This chunk of data, or Working Set, could consist of, but not limited to, any of the following:

- Object caches
- Singletons
- Connection Pools
- Thread Pools
- Metrics

Let's consider one of these REST API servers with a total heap of 1GB, and a working set of 256MB, or 25%. With the default IHOP of 45%, Mixed events should be triggered immediately after Tenured is 45% full, as there would be 25% working set and 45-25 = 20% waste in the heap, well above the default G1HeapWastePercent of 5%. After the cycle of Mixed events complete, we'd expect to see the Tenured size (Working Set + waste) to be around 30% of total heap.

Now consider a similarly configured REST API instance with a working set of 5% of the total heap. Mixed events won't trigger until Tenured is 45% full. There's lots of wasted heap between the 10% low end of Tenured (5% working set + 5% allowed waste) and the high end at 45% when Mixed events finally occur.

Next consider another similarly configured REST API instance with a working set of 75% of total heap. With IHOP at 45%, each and every Minor GC event will trigger an MPCMC if one is not already started. Mixed events will begin as soon as Reclaimable hits the threshold. In this scenario, MPCMC is running almost constantly, and Mixed events are evacuating lots of regions to reclaim little space. To make matters worse, these evacuations are more expensive than evacuations of regions with mostly non-live data.

To efficiently utilize resources and STW times, it's highly recommended that IHOP be tuned per application. At HubSpot we aim for IHOP to be around Working Set + G1HeapWastePercent + 10-15%. If Mixed event cycles are 5 minutes apart, we lower IHOP, and if Mixed event cycles are 5 seconds apart, we raise IHOP.



## Humongous Objects

At this point the majority of the functionality and architecture of G1GC has been flushed out, with the exception of the biggest weakness/complexity, the Humongous object. As mentioned previously, any single data allocation ≥ G1HeapRegionSize/2 is considered a Humongous object, which is allocated out of contiguous regions of Free space, which are then added to Tenured. Let's run through some basic characteristics and how they affect the normal GC lifecycle. The following discussion on Humongous objects will provide insight into the downsides of Humongous objects such as:

- Increase the risk of running out of Free space and triggering a Full GC
- Increase overall time spent in STW

### Humongous objects draw from the Free space pool

Humongous objects are allocated out of Free space. Allocation failures trigger GC events. If an allocation failure from Free space triggers GC, the GC event will be a Full GC, which is very undesirable in most circumstances. To avoid Full GC events in an application with lots of Humongous objects, one must ensure the Free space pool is large enough as compared to Eden that Eden will always fill up first. One usually ends up being over cautious and the application ends up in a state where the Free ram pool is quite large and never fully utilized, which is by definition wasting RAM.

### Humongous objects are freed at the end of an MPCMC

Up until around Oracle jdk 8u45, it was true that Humongous objects were only collected at the end of runs of the MPCMC. The release notes for versions of Oracle 8u45-8u65 have a few commits indicating some, but not all, Humongous objects are being collected during Minor events.

Humongous objects that are only collectable at the end of a MPCMC will increase the requirements for reserved Free space or be more likely to trigger a Full GC.

### Humongous objects kick off MPCMC early

In an effort to reduce the space wasted by unreferenced Humongous objects, the MPCMC is run more often when Humongous objects are present. Each Humongous object allocation will run the IHOP check detailed in the Mixed event section. This time around however, the check is not run after Eden/From have been cleared, which makes it much more likely that an MPCMC will be kicked off. The check will not be run if an MPCMC is already in progress or if the liveness metadata exists.

*Heap currently used Total available heap > IHOP*

If the check passes, a Minor event is immediately started, no matter if Eden was 3% or 93% full.

This characteristic only has an effect during the Minor event cycle, during the Mixed event cycle the MPCMC will not be rerun by Humongous object allocations. As such, during Minor event cycles, Free space will be more quickly reclaimed from unused Humongous objects than during the Mixed event cycle. Overall then, it's probable that the duration of the Mixed event cycle will determine the peak Free space required to avoid To-space Exhaustion.

### Defending against Humongous objects

There are two main strategies, which are often used in parallel, in defending against poor GC behavior due to Humongous objects. The first defense is to increase the G1HeapRegionSize such that fewer allocations qualify as Humongous objects. One can get scientific about picking the region size by running a GC log through a [helper script](https://github.com/HubSpot/gc_log_visualizer/blob/master/regionsize_vs_objectsize.sh). Be warned however, the G1GC algorithm is optimized to work with 2k regions and enlarging the region sizes such that there are only 128 regions may not result in better behavior.

The second defense as alluded to earlier is to increase the Free space such that Eden will fill up first. The parameter -XX:G1ReservePercent (default 10) was created to allow a configurable floor on Free space, however there are caveats to this option as laid out in the section 'Controlling heap sizes'.



## More Concepts and Final Thoughts

Now that we've gone through Mixed GC and the complications Humongous objects introduce, let's conclude by digging deep into a few concepts touched on earlier but left for later.

### Controlling Heap Sizes

What are ideal heap allocations, and how are the heaps configured? The following list is a recap what has been learned so far with a few new facts thrown in.

- Eden size does not affect Minor STW times (except for apps with giant, churning caches)
- Tenured consists of the Working Set + G1HeapWastePercent
- The larger Eden is, the less overall time spent in STW
- Humongous objects/R.O.U.S. are mitigated through larger amounts of Free space
- Eden is configured as a range from G1NewSizePercent (default 5) to G1MaxNewSizePercent (default 60)
- InitiatingHeapOccupancyPercent (default 45) controls when Mixed events will start
- G1ReservePercent (default 10) will attempt to apply a lower bound on Free space

For HubSpot's many REST API instances, as well as a few other services (Kafka and ZooKeeper), the ideal configuration is:

- IHOP 10-15% higher than the Working Set + G1HeapWastePercent
- Free space 10%
- Eden using the rest of the space (see below)
- MaxGCPauseMillis 50% higher than the average Minor GC time

***\*
\****

#### Complication - Not meeting MaxGCPauseMillis target

The -XX:MaxGCPauseMillis (default 250) parameter is used to control the size of Eden. Should the target time consistently be met, the maximum size of the Eden range will be used. Should the target rarely/never be met, the minimum of the Eden range will be used. In theory the Eden value chosen could be somewhere in the middle of the range, but in these situations at HubSpot we generally see the chosen Eden size flap back and forth between the max and the min rather than stabilize in between.

In the case where the MaxGCPauseMillis target is not being met, one can either increase the Eden lower bound G1NewSizePercent or increase the MaxGCPauseMillis target. Increasing the MaxGCPauseMillis target has proven to be the safer option because the lower end of the Eden range does not respect G1ReservePercent.

 

#### Complication - G1NewSizePercent overrides G1ReservePercent

The algorithm to choose the Eden size looks vaguely like the following:

if (recent_STW_time < MaxGCPauseMillis)

 eden = min(100% - G1ReservePercent - Tenured, G1MaxNewSizePercent)

else

 eden = min(100% - Tenured, G1NewSizePercent)

Notice that G1ReservePercent is not a factor if the MaxGCPauseMillis is not being met. If the goal is to dedicate as much heap as possible to Eden, consider the following scenario.

The MaxGCPauseMillis is not being met. Tenured generally sits around 15%, IHOP is set to 30%, the Eden range is the default 5-60%. A G1ReservePercent of 15% is considered safe as occasionally 5% of the Free space is occupied through To-space overflow. The goal for Eden is to use all unused space, which in this case would be 100% - 15% G1ReservePercent - (15-30% Tenured) = 55-70% range.

If we do nothing, an Eden size of 5% will be used, which will result in lots of overall time spent in STW. If we set min Eden in the range of 55-70%, consider the heap distribution if a traffic spike or Request Of Unusual Size raises Tenured to 40%:

40% Tenured + 55-70% Eden = 95-110%

The G1ReservePercent was previously set to 15% to add enough buffer to handle the common occurrence of To-space overflow of 5%. With min Eden set to 55%, there would only be 5% left of Free space for some period of time, greatly increasing the chances of To-space exhaustion.

 

#### Complication - Humongous objects and/or R.O.U.S.

If an application has many Humongous objects and/or R.O.U.S., increase the buffer of Free space set by G1ReservePercent. As per the previous section, there are a few caveats to consider when attempting to reserve a specified amount of Free space.

 

### Comments on -XX:MaxGCPauseMillis

The effects of MaxGCPauseMillis are much more subtle than the name suggests and are primarily seen in two places:

1. Helps choose the Eden size on each epoch. Not meeting the MaxGCPauseMillis goal will result in the minimum end of the Eden range to be used. Meeting the goal will result in the max size being used. Practical experience shows Eden gets set at the ends of the range and rarely anywhere in the middle. This has a major effect on overall time spent in STW as an Eden of 5% will run GC 12x times more frequently than an Eden of 60% of heap.
2. Extends the cap on # of tenured regions that can be collected in a given Mixed GC event. The result could be slightly few Mixed GC events for slightly larger individual STW times of the remaining Mixed GC events.

Setting MaxGCPauseMillis does not in any way guarantee that all STW times will be under the configured value.

**G1GC is complex and for many use cases will require a bit of tuning to get the desired results. In our experience poorly tuned G1GC can provide a much worse user experience than ParallelGC. All in all, G1GC has provided a ton of value for HubSpot, but only after we shifted our mentality from set-and-forget to active monitoring and tuning.**



原文地址：[G1GC Fundamentals: Lessons from Taming Garbage Collection](