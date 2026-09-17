# Suspend / resume

**State: not attempted.** This page exists so the gap is explicit rather than implied.

## Why suspend is unavailable

The machine boots with `acpi=off`, so the ACPI sleep path is not available. A suspend on this class
of hardware would have to come from the firmware/PSCI and the SoC's idle states, which for Glymur
would be a device-tree and driver matter, and there is not yet a known-working recipe for this SoC
in the tree we build against.

Closing the lid currently does nothing.

## How to check the current state

    cat /sys/power/state                     # what the kernel advertises it can do
    cat /sys/power/mem_sleep                 # if the above offers "mem"
    journalctl -k -b 0 -o cat | grep -iE 'psci|idle|s2idle|suspend' | tail

If `/sys/power/state` only offers `freeze`, expect real suspend to need work upstream first.

## Related failure modes

- **Wi-Fi**: an unreproduced flaky moment (a wireless list that came up empty after a long idle)
  was noted while suspend was *not* in play; see [wifi.md](wifi.md). Resume would be the first
  suspect if it happens after suspend is added.
- **Bluetooth**, **input** and especially the **eDP panel** all need to survive a power-cycle of
  their domains. On this SoC a failed eDP enable followed by a disable path has been observed to
  reset the whole SoC silently — see the patch carried in
  [display-outputs.md](display-outputs.md). That failure mode is exactly what a naive
  suspend/resume implementation would walk into.
- README.md phase 5 is the work item.
