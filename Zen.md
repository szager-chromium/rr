Experimental support for AMD Zen-based CPUs has been merged on September 18, 2020.

So far it has been validated to work on:
- Threadripper 3970X
- Ryzen 1800X

You may see the following message when running `rr`:
```
On Zen CPUs, rr will not work reliably unless you disable the hardware SpecLockMap optimization.
```

In that case, please run the [`scripts/zen_workaround.py` script](https://github.com/mozilla/rr/blob/master/scripts/zen_workaround.py) from the `rr` repository as root. You will need to rerun this script after each restart/suspend.