# Performance

One of the goals and constraints of this proposal has been to achieve all the other design goals, whilst also matching or beating the high record set by most.js - which is not just a little bit faster, but an order of magnitude or two faster than alternatives. 

Using the benchmark used by existing libraries in this space, Emitter comfortably and consistently outperforms most.js:

```console
$ node ./filter-map-reduce.js
filter -> map -> reduce 1000000 integers
-----------------------------------------------------
Emitter           204.48 op/s ±  2.14%   (70 samples)
most (multicast)   21.17 op/s ± 19.64%   (37 samples)
most               74.31 op/s ± 22.75%   (59 samples)
cb-basics           6.50 op/s ± 17.46%   (18 samples)
xstream             4.09 op/s ±  5.73%   (24 samples)
rx 5               12.51 op/s ±  6.51%   (61 samples)
rx 4                0.67 op/s ±  5.81%    (8 samples)
kefir               3.94 op/s ± 24.58%   (26 samples)
bacon               0.58 op/s ± 15.76%    (7 samples)
highland            1.42 op/s ±  7.54%   (12 samples)
lodash              5.82 op/s ± 33.08%   (17 samples)
Array               5.88 op/s ± 11.36%   (19 samples)
-----------------------------------------------------
```

This is doubly significant because, for performance reasons, [most.js does not support multiple consumers by default](https://github.com/cujojs/most/issues/207). In a more equal comparison, if you were to run most.js with multicast support at each node, it would significantly drop it's performance.
