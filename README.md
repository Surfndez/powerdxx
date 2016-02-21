                                       _
                                      | |
         ____  ___  _  _  ___  ___  __| |  _     _
        /    \/   \| || |/ _ \|  _\/    |_| |_ _| |_
        | || || | ||    || __/| |  | || |_   _|_   _|
        |  __/\___/\_/\_/\___/|_|  \____/ |_|   |_|
        | |
        \_/  multi-core CPU clock daemon for FreeBSD

The `powerd++` daemon is a drop-in replacement for FreeBSD's native
`powerd(8)`. It monitors the system load and adjusts the CPU clock
accordingly, avoiding some of the pitfalls of `powerd`.

What Pitfalls?
--------------

At the time `powerd++` was first created (February 2016), `powerd`
exhibited some unhealthy behaviours on multi-core machines.

In order to make sure that single core loads do not suffer from the
use of `powerd` it was designed to use the sum load of all cores
as the current load rating. A side effect of this is that it causes
`powerd` to never clock down on systems with even moderate numbers
of cores. E.g. on a quad-core system with hyper threading a background
load of 12.5% per core suffices to score a 100% load rating.

The more cores are added, the worse it gets. Even on a dual core
machine (with HT) having a browser and an e-mail client open, suffices
to keep the load rating above 100% for most of the time, even without
user activity. Thus `powerd` never does its job of saving energy
by reducing the clock frequency.

Advantages of powerd++
----------------------

The `powerd++` implementation addresses this issue and more:

- `powerd++` groups cores with a common clock frequency together and
  handles each group's load and target frequency separately. I.e. the
  moment FreeBSD starts offering individual clock settings on the
  CPU, core or thread level, `powerd++` already supports it.
- `powerd++` takes the highest load within a group of cores to rate
  the load. This approach responds well to single core loads as well
  as evenly distributed loads.
- `powerd++` sets the clock frequency according to a load target, i.e.
  it jumps right to the clock rate it will stay in if the load does
  not change.
- `powerd++` supports taking the average load over more than two
  samples, this makes it more robust against small load spikes, but
  sacrifices less responsiveness than just increasing the polling
  interval would. Because only the oldest and the newest sample are
  required for calculating the average, this approach does not even
  cause additional runtime cost!
- `powerd++` parses command line arguments as floating point numbers,
  allowing expressive commands like `powerd++ --batt 1.2ghz`.

Building
--------

Download the repository and run `make`:

    > make
    c++ -O2 -pipe  -std=c++11 -Wall -Werror -c src/powerd++.cpp -o powerd++.o
    c++ -lutil -O2 -pipe  -std=c++11 -Wall -Werror -o powerd++ powerd++.o

To avoid building directly in the repository folder an `obj` folder can
be created prior to building.

Documentation
-------------

The manual page can be read with the following command:

    > nroff -mdoc powerd++.8 | less -r

FAQ
---

- **Why C++?** The `powerd++` code is not object oriented, but it uses
  some *C++* and *C++11* features to avoid some of the common pitfalls of
  writing *C* code. E.g. there is a small *RAII* wrapper around the
  pidfile facilities (`pidfile_open()`, `pidfile_write()`,
  `pidfile_remove()`), turning the use of pidfiles into a fire and forget
  affair. Templated wrappers around calls like `sysctl()` use array
  references to infer buffer sizes at compile time, taking the burden of
  safely passing these buffer sizes on to the command away from the
  programmer. The `std::unique_ptr<>` template obsoletes memory cleanup
  code, providing the liberty of using exceptions without worrying
  about memory leaks.
