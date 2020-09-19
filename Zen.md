Experimental support for AMD Zen-based CPUs has been merged on September 18, 2020.

So far it has been validated to work on:
- Threadripper 3970X
- Ryzen 9 3950X

You may see the following message when running `rr`:
```
On Zen CPUs, rr will not work reliably unless you disable the hardware SpecLockMap optimization.
```

In that case, please run the [`scripts/zen_workaround.py` script](https://github.com/mozilla/rr/blob/master/scripts/zen_workaround.py) from the `rr` repository as root. You will need to rerun this script after each reboot/suspend.

If the `scripts/zen_workaround.py` script tells your the workaround doesn't stick, it means you'll have to try some other or additional workaround:

## [SSB mitigation](https://en.wikipedia.org/wiki/Speculative_Store_Bypass)

Status: untested.

The reason why the workaround doesn't stick is that the kernel switches Speculative Store Bypass mitigation on and off when switching between processes with different mitigation configuration. When doing so, it resets the workaround on a core. After some time, all cores have the workaround reset.

By either fully enabling or fully disabling SSB mitigation system wide, the workaround should be able to stick. This is done by adding either `nospec_store_bypass_disable` or `spec_store_bypass_disable=on` on the kernel command line. The former has security implications, and the latter has performance implications.

After boot, running the `scripts/zen_workaround.py` script should work.

## Kernel module

Status: untested.

Note: it doesn't work on Linux kernel >= 5.7.

[A kernel module](https://gist.github.com/glandium/01d54cefdb70561b5f6675e08f2990f2) can be used to apply the workaround in a way that prevents SSB mitigation from resetting it.

Download both `Makefile` and `zen_workaround.c` files, and run `make`. Assuming you have all the right packages installed to build kernel modules, and that the path for modules is `/lib/modules/$(uname -r)`.

Loading the module with `insmod zen_workaround.ko` should apply the workaround. This can be validated with `scripts/zen_workaround.py --check`.

Unloading the module with `rmmod zen_workaround` resets the workaround.

The module currently doesn't preserve the workaround after resuming from suspend. You'll have to unload and reload the module.

## Grub

Status: Untested

Recent versions of grub have a way to set MSRs. Doing so before the Linux kernel boots ensures the kernel will preserve the workaround, at the expense of not being able to disable it when not necessary.

First, determine the current MSR value with `rdmsr` from `msr-tools`:
```
rdmsr -x 0xc0011020
```

Take the resulting value (it's hexadecimal), and use your favorite REPL to set bit 54 and unset bit 10. For example in python:
```
>>> '0x%x' % ((0x<value_from_rdmsr> | (1 << 54)) & ~(1 << 10))
```

Now put that modified value in some appropriate location in your grub configuration:
```
wrmsr 0xc0011020 0x<modified_value>
```