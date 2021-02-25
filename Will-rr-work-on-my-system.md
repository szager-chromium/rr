This question mostly pertains to Virtual Machines rather than real hardware. To check if it's supposed to work execute:

```
 $ perf stat -e br_inst_retired.conditional true

 Performance counter stats for 'true':

            95,027      br_inst_retired.conditional

       0.003255114 seconds time elapsed

       0.003127000 seconds user
       0.000000000 seconds sys
```

Check that `br_inst_retired.conditional` value is not zero in the output.
