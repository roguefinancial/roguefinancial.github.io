---
layout: post
title: Erlang NIF with timeslice reductions
tags: C Elixir Erlang
---

Recently, I put together an Erlang asynchronous port driver named [keccakf1600](https://github.com/potatosalad/erlang-keccakf1600) which implements the [SHA-3](https://en.wikipedia.org/wiki/SHA-3) algorithms used in another one of my projects, [jose](https://github.com/potatosalad/erlang-jose).

See version [1.0.2 of keccakf1600](https://github.com/potatosalad/erlang-keccakf1600/tree/1.0.2) for the original port driver implementation.

When interfacing with native C and the Erlang VM, you essentially have 3 options to choose from:

1. [Port Driver](http://erlang.org/doc/tutorial/c_portdriver.html) &mdash; a shared library linked with [`driver_entry`](http://erlang.org/doc/man/driver_entry.html) (I/O heavy operations are typically best suited for this type)
2. [NIF](http://erlang.org/doc/tutorial/nif.html) &mdash; a shared library linked with [`ERL_NIF_INIT`](http://erlang.org/doc/man/erl_nif.html) (fast synchronous operations are typically best suited for this type)
3. [Port](http://erlang.org/doc/tutorial/c_port.html) &mdash; an external program which typically communicates with the Erlang VM over `stdin` and `stdout`

My goal was to have a fast and asynchronous way to call blocking functions without disrupting the Erlang VM schedulers from carrying out their work.  The original plan was to use [`driver_async`](http://erlang.org/doc/man/erl_driver.html#driver_async%20) combined with [`ready_async`](http://erlang.org/doc/man/driver_entry.html#ready_async) to perform the blocking operations on &ldquo;a thread separate from the emulator thread.&rdquo;  I used the [`ei`](http://erlang.org/doc/man/ei.html) library in order to communicate between the Erlang VM and the port driver written in C.

Having accomplished my goal, I decided to run a simple benchmark against the equivalent SHA-2 algorithms out of curiosity as to how my implementation might stack up against the native Erlang [`crypto`](http://erlang.org/doc/man/crypto.html) library.

The results were not terribly impressive:

<div id="chart1">
  <svg height="300"></svg>
</div>

The two main concerns I had with the results were:

1. Was the SHA-3 implementation I used (based on [ed448goldilocks](http://sourceforge.net/projects/ed448goldilocks/)) really 5-7 times slower than the SHA-2 algorithms?
2. Why was there so much variance between the SHA-3 algorithms versus the variance observed between the SHA-2 algorithms?

Concern #1 was ruled out by directly testing the C version of the algorithms, for small message sizes they were typically within 1-2μs of each other.

Concern #2 required more research, which eventually led me to the [bitwise](https://github.com/vinoski/bitwise) project by Steve Vinoski.  The project explores some of the strategies for dealing with the synchronous nature of a NIF without blocking the scheduler by keeping track of reductions during a given [timeslice](http://erlang.org/doc/man/erl_nif.html#enif_consume_timeslice).  It also explores strategies using the experimental [dirty NIF](http://erlang.org/doc/man/erl_nif.html#dirty_nifs) feature.

I highly recommend reading the two presentations from the bitwise project: [vinoski-opt-native-code.pdf](https://github.com/vinoski/bitwise/raw/master/vinoski-opt-native-code.pdf) and [vinoski-schedulers.pdf](https://github.com/vinoski/bitwise/raw/master/vinoski-schedulers.pdf).

After experimenting with the two options, I decided to use [`enif_consume_timeslice`](http://erlang.org/doc/man/erl_nif.html#enif_consume_timeslice) combined with [`enif_schedule_nif`](http://erlang.org/doc/man/erl_nif.html#enif_schedule_nif) to yield control back to the main Erlang VM on larger inputs to prevent blocking other schedulers.

I rewrote the port driver as a NIF and released it as version [2.0.0 of keccakf1600](https://github.com/potatosalad/erlang-keccakf1600/tree/2.0.0) and ran the same benchmark again:

<div id="chart2">
  <svg height="300"></svg>
</div>

These results are much more consistent and closer to my original expectations.  I plan on refactoring the [erlang-libsodium](https://github.com/potatosalad/erlang-libsodium) project using the same technique.

<script>
(function() {
  var seriesData1 = [
    {
      key: "SHA2",
      values: [
        {
          label: "SHA2/3-224",
          value: 2.186853
        },
        {
          label: "SHA2/3-256",
          value: 2.183836
        },
        {
          label: "SHA2/3-384",
          value: 2.190898
        },
        {
          label: "SHA2/3-512",
          value: 2.200743
        }
      ]
    },
    {
      key: "SHA3",
      values: [
        {
          label: "SHA2/3-224",
          value: 14.37063
        },
        {
          label: "SHA2/3-256",
          value: 11.97354
        },
        {
          label: "SHA2/3-384",
          value: 12.42217
        },
        {
          label: "SHA2/3-512",
          value: 11.8663
        },
        {
          label: "SHAKE128",
          value: 12.38027
        },
        {
          label: "SHAKE256",
          value: 12.96572
        }
      ]
    }
  ];
  nv.addGraph(function() {
    var chart = nv.models.multiBarHorizontalChart()
      .x(function(d) { return d.label; })
      .y(function(d) { return d.value; })
      .margin({left: 100, bottom: 75})
      .barColor(d3.scale.category20().range())
      .showValues(true)
      .duration(250)
      .showControls(false)
    ;
    chart.yAxis.axisLabel('Microseconds (Lower is Better)');
    d3.select("#chart1 svg")
      .datum(seriesData1)
      .call(chart);
    nv.utils.windowResize(chart.update);
    return chart;
  });
  var seriesData2 = [
    {
      key: "SHA2",
      values: [
        {
          label: "SHA2/3-224",
          value: 2.186853
        },
        {
          label: "SHA2/3-256",
          value: 2.183836
        },
        {
          label: "SHA2/3-384",
          value: 2.190898
        },
        {
          label: "SHA2/3-512",
          value: 2.200743
        }
      ]
    },
    {
      key: "SHA3",
      values: [
        {
          label: "SHA2/3-224",
          value: 3.221057
        },
        {
          label: "SHA2/3-256",
          value: 3.222021
        },
        {
          label: "SHA2/3-384",
          value: 3.200662
        },
        {
          label: "SHA2/3-512",
          value: 3.222704
        },
        {
          label: "SHAKE128",
          value: 3.228499
        },
        {
          label: "SHAKE256",
          value: 3.257091
        }
      ]
    }
  ];
  nv.addGraph(function() {
    var chart = nv.models.multiBarHorizontalChart()
      .x(function(d) { return d.label; })
      .y(function(d) { return d.value; })
      .margin({left: 100, bottom: 75})
      .barColor(d3.scale.category20().range())
      .showValues(true)
      .duration(250)
      .showControls(false)
    ;
    chart.yAxis.axisLabel('Microseconds (Lower is Better)');
    d3.select("#chart2 svg")
      .datum(seriesData2)
      .call(chart);
    nv.utils.windowResize(chart.update);
    return chart;
  });
})();
</script>
