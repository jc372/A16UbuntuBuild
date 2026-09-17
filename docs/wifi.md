# Wi-Fi — `ath12k` (Wi-Fi 7)

**State: working.** No patches, no module builds — the in-tree driver handles this part.

## The hardware

| | |
|---|---|
| Driver | `ath12k` (also `ath12k_wifi7`) |
| Board data | the board file the driver expects is present in the firmware package; see `notes/2026-09-16-hermes-wifi-board-data.md` |
| Interface | `wlP4p1s0`, managed by NetworkManager |

## Verify

    nmcli device status
    ip -brief addr show wlP4p1s0
    journalctl -k -b 0 -o cat | grep -i ath12k | tail

## An unreproduced observation

On one occasion, after roughly twenty minutes of idle with no login, the desktop's Wi-Fi list was
empty: the connection could be seen, toggling the radio did not help, and networks were not found.
Nothing in the logs points at a cause, and the interface was healthy before and after. It is written
up as README.md phase 8 rather than as a defect, because:

- there were no `ath12k` errors in that boot's log,
- there were no suspend/resume events (see [suspend.md](suspend.md) — suspend is not implemented),
- power saving is not the cause here: `nmcli` shows powersave off for this interface.

If it happens again, the useful evidence is:

    journalctl -b -1 -o cat | grep -iE 'ath12k|wlan|cfg80211|NetworkManager' | tail -50

## Related

Bluetooth shares the combo chip but is a separate driver path and a separate bring-up; see
[bluetooth.md](bluetooth.md).
