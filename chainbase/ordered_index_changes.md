## chainbase index improvements

chainbase always uses ordered_index for its tables. This is a sorted data structure that has the benefit of allowing sorted iteration over index but with a log(n) lookup time.

Well, it turns out that log(n) lookup time is becoming increasingly a burden. Profiling data suggests well in to the double digit percentage of time is spent looking up an entry while using OC. But many of those lookups don't need to be sorted*! The account objects, table ids, permissions, etc do not need to be sorted. A hash table of some sort would be the better choice to ensure constant time lookup.

Sadly on a cursory look we can't just use a multiindex hashed_index because it's a simple data structure that seems to do things like grow its buckets by an order of 2 and rehash. Something more like Linear Hashing is what would be appropriate to provide constant time lookup, low overhead (little wasted buckets), and no gross rehashing.

So it seems like (to stick with the current chainbase scheme) we'd need to customize boost::multi_index to provide a new hash index type based on a custom hash data structure. This may imply a fork of the pertinent boost bits, but we've noodled that as a possibility anyways to ensure that consensus bits (which boost multi_index is certainly included in) are pinned down. So maybe it's not so extreme to pursue this.

It may be possible to get a ballpark of the performance improvement attainable just by using the existing hashed_index.

It's a little hard to estimate the change in memory usage of this modification. Certainly the existing trees have some overhead too, but it's not clear the efficiency of Linear Hashing is good enough to match.

*: well, snapshots do need those tables sorted. I don't really know a good solution here other then incurring the cost (of time and RAM) of sorting the structure only when a snapshot is being created.

