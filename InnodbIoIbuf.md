InnoDB performance on many IO bound workloads is much better than expected because of the insert buffer. Unfortunately, InnoDB does not try hard enough to keep the insert buffer from getting full. And when it gets full it kills performance because it continues to use memory from the buffer pool but cannot be used to defer IO for secondary index maintenance.

The v4 patch has several changes to fix this:
  * the my.cnf variable _innodb\_ibuf\_max\_pct\_of\_buffer_ specifies a soft limit on the size of the buffer pool. The hard limit is 50%. When the hard limit is reached no more inserts are done to the insert buffer. When the soft limit is reached, the main background IO thread aggressively requests prefetch reads to merge insert buffer records.
  * the my.cnf variable _innodb\_ibuf\_flush\_pct_ specifies the number of prefetch reads that can be submitted at a time as a percentage of _innodb\_io\_capacity_. Prior to the v4 patch, InnoDB did 5 prefetch read requests at a time and this was usually done once per second.
  * the my.cnf variable _innodb\_ibuf\_reads\_sync_ determines whether async IO is used for the prefetch reads. Prior to the v4 patch, sync IO was used for the prefetch reads done to merge insert buffer records. This variable was added for testing as the default value (_skip\_innodb\_ibuf\_reads\_sync_) should be used in production.
  * code is added to delay user sessions and make them merge insert buffer records when the size of the insert buffer exceeds the soft limit.