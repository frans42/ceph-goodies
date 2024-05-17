[WIP]

# Tuning the scrub machine

*Note: while the discussion below focuses mainly on HDDs due to their low IOPS performance per TB, we also address solid state storage and give recommendations.*

## Introduction

The subsequent investigation was the result of trying to fix issues with PGs not being deep scrubbed in time. After blindly trying a few recommendations provided by the ceph user list making matters worse, we decided to look under the hood and find out why we had such problems and what the actual result of certain tuning parameters is.

Threads and posts related to problems addressed and problems observed are (search for thread title if the link is dead, some threads might have moved to archive):

- [Thundering herd of deep-scrubs](https://lore.kernel.org/all/CABZ+qq=7izv5o-5ACygqZyr=ho58nLoKb7XmRKT2yyTqKFwrZQ@mail.gmail.com/)
- [increasing number of (deep) scrubs](https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/thread/NHOHZLVQ3CKM7P7XJWGVXZUXY24ZE7RK)
- [How to configure something like osd_deep_scrub_min_interval?](https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/thread/YUHWQCDAKP5MPU6ODTXUSKT7RVPERBJF)
- [Ceph and Deep-Scrubs](https://silvenga.com/ceph-and-deep-scrubs/)

With HDDs becoming larger while the IOPS performance per disk stays essentially constant, the available IOPS per TB are decreasing with disk size. For disks of sizes >=5-10TB this leads to the problem that it becomes more and more difficult to perform admin operations like scrub and deep-scrub alongside user-IO.

With the current default scrub settings for ceph this leads to "PGs not (deep-)scrubbed in time" warnings that start piling up and scrub seems never to be able to catch up again. For large HDDs one needs to look into the scrub parameters together with what performance HDDs can actually provide to find a balance between user experience and (deep-)scrub times.

A side conclusion of the observations reported here is that there is an upper limit of HDD size for which HDDs become essentially useless for non-archival (or even any?) storage systems. The hard upper limit is reached when the sustained IOPS performance/TB is too low to complete (deep-)scrubbing with reasonable scrub-times at all. The soft limit is when remaining IOPS per TB performance is so low that users consider the storage unusable. Our experience indicates that this soft limit is reached somewhere around 5-10 IOPS per TB, which are drives of size 16-32TB capacity.

As a consequence, we limit drive sizes to <=16TB and aim for no more than 10TB/drive utilization. This is the first step to ensure that a HDD pool can serve reasonable IOPS while (deep-)scrubbing within acceptable intervals.

All this, of course, depends on actual workload and will be different from cluster to cluster. To add some background, we are here mainly concerned with an 850+ HDD-OSD pool with EC8+3 profile used as a data pool in a ceph fs. Usable capacity is about 9PB and used is 5.3PB. We have a large percentage cold data and a thin layer of between 50-100TB hot data. User IO hits the hot data while scrubbing must go through everything.

This pool has approximately an aggregated IOPS budget of 5800, which sounds little but is good enough for our use case of a bulk data store. In the future we plan a hybrid OSD config with dm-cache layer for the hot data on solid state storage. One of the major issues is that we have a larger percentage of small files and also a lot of hard links on the file system, which are a pain for both, deep-scrub and recovery.

## Goal

With our tuning recommendations we aim at configuring (dep-)scrub such that:

- PGs scrub every A-B days
- PGs deep-scrub every C-D days
- no PGs (deep-)scrub before (C)A days, that is, no premature (deep-)scrubbing
- PGs with (deep-)scrub ages <(C)A should be evenly distributed in the scrub time histogram
- (deep-)scrub load under normal conditions should be low enough such that the scrub machine has sufficient reserves to handle exceptional situations causing (deep-)scrub to pause, for example, rebalancing after adding OSDs or periods of high load on the servers; (deep-)scrub should catch up after a reasonable amount of time

With the current implementation of the ceph scrub machine and parameters available for tuning we were able to find configurations that approximate these goals sufficiently well for our purposes. In many cases, our [general recommendations](RecommendationsForScrub.md) will do.

## Terminology and relevant parameters

We call a PG *idle* if it is active+clean and no OSD that is a member of this PG is scrubbing or scrubbing+deep. We call a PG *eligible* for (deep-)scrub if its (deep-)scrub age has passed the (deep-)scrub threshold. PGs that are eligible+idle can be (deep-)scrubbed at any time. This explains the mystery why a "`ceph pg deep-scrub x.y`" often does not cause a PG to deep-scrub (immediately). The command forces the PG to become eligible, but (deep-)scrub start has to wait until it also becomes idle.
