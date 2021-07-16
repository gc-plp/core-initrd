# How to measure core-initrd boot time

This repository is a fork of the https://github.com/snapcore/core-initrd project and contains two changes:

1. A PoC for measuring the total time spent in the initrd.
2. Small code improvements that contribute to quicker initrd generation.

The sections below detail each one.


# PoC for measuring the total time spent in the initrd

Measuring the initrd duration is done through two services that are called at the beginning and end of the initrd initialization:

1. [`start-measure-boottime.service`](https://github.com/gc-plp/core-initrd/commit/f00fec1b6da0b0d7a3821f5dbeaae86bb8896650#diff-a35afa547fa049d324236fb9cf3cac2ebc064f2490fbe754c4a788374698e584) - is responsible for creating a simple file descriptor using the current date in milliseconds in `/tmp/boot-start`, by simply calling `date +%s%N | cut -b1-13`. Since `date` command outputs the current date in nanoseconds, we only keep the first 13 digits to save the date in milliseconds.

2. [`end-measure-boottime.service`](https://github.com/gc-plp/core-initrd/commit/f00fec1b6da0b0d7a3821f5dbeaae86bb8896650#diff-9e136312bb4db02f67129bf5fecb5fdb97bf995058595df2adc99cb049efc2ba) - is responsible for getting the same timestamp as above, after which it substracts the timestamp created by `start-measure-boottime.service`. Finally, it prints the Initrd boot time: `Initrd boot time: XXX ms`.

## How it is implemented

According to the systemd [documentation](https://www.freedesktop.org/software/systemd/man/bootup.html), systemd goes through
a few critical steps during the initrd:

```bash
                                               : (beginning identical to above)
                                               :
                                               v
                                         basic.target
                                               |                                 emergency.service
                        ______________________/|                                         |
                       /                       |                                         v
                       |            initrd-root-device.target                    emergency.target
                       |                       |
                       |                       v
                       |                  sysroot.mount
                       |                       |
                       |                       v
                       |             initrd-root-fs.target
                       |                       |
                       |                       v
                       v            initrd-parse-etc.service
                (custom initrd                 |
                 services...)                  v
                       |            (sysroot-usr.mount and
                       |             various mounts marked
                       |               with fstab option
                       |              x-initrd.mount...)
                       |                       |
                       |                       v
                       |                initrd-fs.target
                       \______________________ |
                                              \|
                                               v
                                          initrd.target
                                               |
                                               v
                                     initrd-cleanup.service
                                          isolates to
                                    initrd-switch-root.target
                                               |
                                               v
                        ______________________/|
                       /                       v
                       |        initrd-udevadm-cleanup-db.service
                       v                       |
                (custom initrd                 |
                 services...)                  |
                       \______________________ |
                                              \|
                                               v
                                   initrd-switch-root.target
                                               |
                                               v
                                   initrd-switch-root.service
                                               |
                                               v
                                     Transition to Host OS
```

Based on the steps depicted above, we can add two hooks at the beginning and the end of the boot-up sequence:

- Before `basic.target` is reached, we add a dependency on having `start-measure-boottime.service` executed.
- Before `initrd-switch-root.target` is reached, we add a dependency on having `end-measure-boottime.service` executed.

By simply chaining the two services, we obtain the initrd duration.

Advantages:

- Simple implementation that is clearly visible thanks to the services.
- Modularization through services.

Disadvantages:

- More work for systemd to run.
- Does not measure the exact duration due to having to wait for `start-measure-boottime.serviec` to start (details below).

## Potential improvements

Since this method relies on having systemd launch the two services, it misses the duration between when `/sbin/init` is executed and `start-measure-boottime.service` is executed.

One potential improvement would be to replace `/sbin/init` with a bash script that creates the timestamp when initrd execution started after which the normal `init` would be executed. The disadvantage of this method would be that a `tmpfs` would have to be mounted in order to create the file.


# Small code improvements that contribute to quicker initrd generation

This [change](https://github.com/gc-plp/core-initrd/commit/a7924f63d50a58c1950b8c0b4a18af4a8e789723) is quite straightforward and uses the asyncio module in Python to run tasks in parallel.

Since the initrd creation relies on running multiple subprocesses, these subprocesses can be executed in parallel instead of sequentially, thus having the potential of increasing code execution speed.