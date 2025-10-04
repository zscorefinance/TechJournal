HyperLogLog is a probabilistic data structure in Redis. It is actually an algorithm for the count-distinct problem 👉🏼 [Wikipedia](https://en.wikipedia.org/wiki/HyperLogLog)

It's used as a Redis data type but has nothing to do with logs; it estimates cardinality of a data set, which means, counting unique values in the collection.

Use when you want to count unique elements in a collection (well approximately), like number of times a video has been watched by unique users, or, unique number of vehicles passing thru a traffic intersection. 

This is similar to sets in a way, but without storing the elements. A set would be storing all the elements and then count them which would need a lot of memory. The use case for HyperLogLog is that you don't want to retrieve the elements back, so you don't store them, you just count the number of unique elements that you come across. Therefore, HyperLogLog doesn't store the data, it "remembers" them, and HyperLogLog uses (up to) 12KB of memory for 2^64 elements. Even at this scale, it has a standard error of just 0.81%, which means for an expected count of 1000, you will get somewhere between 992 and 1008. Extremely good trade-off!

Commands:

➡️ `PFADD` - Takes a string and adds it to the HyperLogLog data structure. In other words, it remembers the string without storing! Returns 1 if added, 0 if already exists (hence not counting the same key twice). Meaning, doesn't matter how many times you watch the video, HyperLogLog will count your UserID only once. Internally, HyperLogLog uses registers and sub-registers to remember what was added. All you need to know about HyperLogLog and the mathematics behind it, is here 👉🏼 [Original post on HyperLogLog by antirez](http://antirez.com/news/75).

Example (users watching a video), 
```bash
PFADD VID001 USERID1 USERID2
```

Returns `1`, if USERID1 and/or USERID2 were/was added (meaning, at least one of those registers were altered to remember either of them)
Returns `0`, if neither USERID1 nor USERID2 were not added, as it remembers them from a previous add (meaning, none of those registers were altered). But if one of them were added, meaning register altered, the command would have returned `1`.

➡️ `PFCOUNT` - Counts the number of unique elements (well, approximately). 
Example,

```bash
PFADD VID001 USERID1 USERID2 USERID3 USERID1
```

Returns `3`. (Note, though USERID1 watched the video twice, the ID is counted once)

➡️ `PFMERGE` - Merges multiple HyperLogLogs into one.
Example,
```bash
PFMERGE newHLL oldHLL1 oldHLL2
```

If you are wondering why the commands are prefixed with PF, it's in honor of Philippe Flajolet, the French computer scientist, who along with his colleagues invented the HyperLogLog algorithm for near-optimal cardinality estimation 👉🏼[Original paper by the inventor](https://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf).
