# Experimental MiProjM05 wake sender

`tools/xiaomi_projector_wake.py` is an opt-in Linux host tool. It reproduces a
wake advertisement captured from an original Xiaomi RC. It is separate from
the integration's default pairing-request method: installing MiPower does not
automatically select this sender or change the adapter identity.

## Device observation

One Mi Smart Compact Projector (`MiProjM05`, Android TV 9) was investigated with
Ubuntu, BlueZ 5.72 and a Linux 7.0 host. Classic HID sleep/wake worked while
connected, but Bluetooth and network access disappeared after several minutes.
Replaying the packet from an unrecognized host address failed, including with
a 20 ms advertising interval.

The projector's Bluetooth service recognizes wake remotes by Xiaomi VID/PID
or certain name prefixes, including `MI RC`. Renaming the already paired HID
host to `MI RC HA Projector` and rediscovering it caused the firmware to save
its unchanged address alongside the original remote in `rc_macaddr`. The bond
file did not change, and this name change required no new pairing.

A subsequent test waited for Bluetooth and TCP access to disappear, then sent
the packet from the recognized host. The network returned 46 seconds after
transmission began. Android reported Awake/display ON, and the operator
confirmed no physical remote press. This is evidence from one device/firmware;
it is not a compatibility claim for other Xiaomi projectors.

A separate host service using the same sender was then exercised through the
Home Assistant REST API: the sleep action reached screen-off, Bluetooth/network
access was allowed to disappear, and the HA wake script returned to its actual
`on` state in 66.8 seconds. Android independently confirmed display ON. The
Home Assistant container and shared Bluetooth service were not restarted.

## Requirements and usage

- Run on the Linux Bluetooth host with `CAP_NET_ADMIN` (normally root).
- The adapter must already be accepted by the projector as a wake remote.
  This tool does not supply a HID profile or edit the projector's wake list.
- `--target` is the projector's **Bluetooth** address, not its Wi-Fi address.
  The sender keeps its own controller address and does not load or replace keys.
- An unused advertising instance and support for the extended advertising
  parameter management command are required. Unsupported controllers fail.
- Coordinate access to the adapter: concurrent sender processes or other
  advertising reconfiguration during a transmission are unsupported.
- A separate host service can expose the sender to HA (for example over MQTT).
  The HA container then needs no added privileges. This PR does not install
  that service or restart Home Assistant/Bluetooth.

Example with an anonymized address:

```sh
sudo python3 tools/xiaomi_projector_wake.py --target AA:BB:CC:11:22:33
```

Duration is bounded to 1–20 seconds, with a kernel timeout and cleanup of the
selected temporary instance. Existing instances and shared adapter state are
checked before/after. Zero exit status confirms transmission and host checks,
**not** projector power state; a caller must independently verify power and
allow time for a full boot.

## Captured advertisement

The 31-byte payload is:

| AD type | Observed bytes |
| --- | --- |
| Flags `01` | `05` |
| Manufacturer field `FF` | `00 01`, with no following manufacturer payload |
| Short name `08` | `MI RC` |
| Partial 16-bit UUIDs `02` | `12 18` (HID) |
| Class `0D` | `04 05 00` |
| TX power `0A` | `00` |
| Observed type `FE` | Last three target Bluetooth address bytes, reversed |

The tool uses legacy connectable undirected advertising (`ADV_IND`) at a
20 ms interval through an additional bounded management instance. It does
not reset advertising globally or toggle controller power.

See [BlueZ's management API](https://raw.githubusercontent.com/bluez/bluez/5.72/doc/mgmt-api.txt)
for per-instance behavior and [Xiaomi's FAQ](https://www.mi.com/global/support/faq/details/KA-06604/)
for original-remote power behavior. The packet and firmware classification
above come from capture/code inspection, not a public Xiaomi protocol spec.
