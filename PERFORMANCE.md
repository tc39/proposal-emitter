# Performance

One of the goals and constraints of this proposal has been to achieve all the other design goals, whilst also matching or beating the high record set by most.js - which is not just a little bit faster, but an order of magnitude or two faster than alternatives. 

Using the benchmark used by existing libraries in this space, Emitter comfortably and consistently outperforms most.js:

```console
$ node ./perf/filter-map-reduce.js
filter -> map -> reduce 1000000 integers
-----------------------------------------------
Emitter     141.10 op/s ±  3.49%   (63 samples)
most        113.53 op/s ± 13.72%   (70 samples)
cb-basics     8.52 op/s ± 11.13%   (41 samples)
xstream       4.42 op/s ± 12.01%   (26 samples)
rx 5         24.84 op/s ±  9.62%   (60 samples)
rx 4          0.54 op/s ± 28.13%    (9 samples)
kefir         3.66 op/s ± 37.29%   (26 samples)
bacon         0.70 op/s ± 18.09%    (8 samples)
highland      1.92 op/s ± 17.10%   (14 samples)
lodash        4.91 op/s ±  9.11%   (16 samples)
Array         1.77 op/s ± 11.25%    (9 samples)
-----------------------------------------------
```

This is doubly significant because, for performance reasons, [most.js does not support multiple consumers by default](https://github.com/cujojs/most/issues/207). In a more equal comparison, if you were to run most.js with multicast support at each node, it would significantly drop it's performance.
