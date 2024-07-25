# Stuck Scrub Reservations

Scrubbing of an eligible+idle PG is initiated by its primary OSD by trying to acquire a scrub reservation for all of its member OSDs. An OSD can only grant `max_scrubs` scrub reservations and if an OSD is found that has already `max_scrubs` reservations granted, the process is aborted and the scrub reservations are returned. If an OSD can acquire a scrub reservation for all member OSDs, a scrub starts. Depending on a number of conditions and config settings, a deep-scrub might be executed instead.

While investigating the issue reported in the thread [increasing number of (deep) scrubs](https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/thread/NHOHZLVQ3CKM7P7XJWGVXZUXY24ZE7RK) ([thread in archive](https://www.spinics.net/lists/ceph-users/msg75292.html)) we found that the [scrub slot paradox](ScrubSlotParadox.md) explains part of the story, but not all of it. Our long-term study trying many different settings showed that the ceph scrub machine has a surprisingly low scrub slot utilization of no more than about 50-66% even if we have a very high idle% ([see script pool-scrub-report](../scripts/pool-scrub-report)).

To get to the root of it, we started looking at the source code ([pg_scrubber.h is a good entry point](https://github.com/ceph/ceph/blob/main/src/osd/scrubber/pg_scrubber.h)) and found a second quite important clue. The comments in this collection of source files mention that there is a problem with scrub reservations not always being released properly, which is actually the main reason for low scrub slot utilization besides the inherent racyness of the process of acquiring scrub reservations. When we look at [the scrub slot paradox](ScrubSlotParadox.md) it is clear that only a handful of stuck scrub reservations can drastically reduce the number of remaining available scrub slots. The comments in the code also mention that all scrub reservations get cleared on certain events, one of which is a change to `scrub_max_interval`.

In a simple experiment we confirmed that running the [script bump_scrubs](../scripts/bump_scrubs) (note: requires using [per-pool scrub settings](RecommendationsForScrub.md#why-use-pool-settings-for-scrub-instead-of-osd-settings)) in a cron job, for example, with a crontab line like

    #    m  h dom M dow
    # */15  * *   * *   /path/to/bump_scrubs pool1-name pool2-name ...

seriously gets things going. Running it every 15 minutes gives already a significant speed-up. Increasing the frequency to every 5 minutes raised the scrub slot utilization to almost 100%, resulting in a speed-up of ordinary scrubs by a factor of three or more and of deep-scrub by an extra 10-30%. Its the ordinary scrubs that suffer most from low scrub slot utilization, because all the allocated slots are occupied with long-running deep-scrubs and only very few ordinary scrubs squeeze through when scrub reservations get stuck for too long.

Having found out about this, we decided to relax the scrub times such that the low scrub slot utilization of the scrub machine can handle scrubbing on its own. However, we keep the cron job as a life line should we encounter serious delays again. In case you really need to reduce (deep-)scrub times at all cost, give this script a shot. It will pause while ceph is in health warn state, which includes maintenance mode by setting noout to prevent interference with admin operations.

---
Back: [The scrub slot paradox.](ScrubSlotParadox.md)
Start: [Scrub tuning guide.](TuningScrub.md)