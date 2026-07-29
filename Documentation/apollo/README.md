# Xiaomi Mi 10T Pro (apollo / J3S) on postmarketOS

Linux 6.17 for the Xiaomi Mi 10T Pro / Redmi K30S Ultra, with working
**mobile data and SMS** through the phone's external Snapdragon X55 (SDX55M)
modem.

This is a working fork, not a mainline submission. It is published so that
people interested in this device — or in any SM8250 board with an external
X55 — can use it and build on it. Expect rough edges: some commits are
grouped by subsystem rather than split per logical change, and intermediate
commits are not individually buildable.

**Base:** the postmarketOS SM8250 tree at tag `sm8250-6.17.0`
(<https://gitlab.postmarketos.org/soc/qualcomm-sm8250/linux>). postmarketOS has
since moved to 7.1.0, so this is a snapshot, not a rebase target. Nothing here
depends on 6.17 specifically; the modem work should port forward.

## Status

| Area | State |
|---|---|
| Display, touchscreen, 144 Hz | works |
| Battery, charger, fuel gauge | works |
| Audio playback and capture | works |
| WiFi, Bluetooth | works (QCA6390) |
| **Mobile data (LTE)** | **works — ~834 Mbit/s down** |
| **SMS** | **works** |
| Voice registration (CS attach) | works |
| **Modem crash recovery (SSR)** | **works** — no reboot needed |
| Voice call audio | **not working** — no mic, no earpiece |
| Sustained peak throughput | wedges downlink at high rates (see below) |
| GPS | untested |
| Camera | not supported |

## The modem is the hard part

The SM8250 has **no on-die modem**. This board pairs it with a discrete
SDX55M on PCIe2, spoken to over MHI, with sideband GPIOs managed by the
downstream `esoc` framework. Almost everything in this tree that is not
board plumbing exists to make that arrangement work.

### Runtime prerequisites — the kernel alone is not enough

None of these are optional and none are obvious. Without them the modem will
either never boot or will die ~15 s into mission mode.

**1. Real EFS partitions.** This is the one that matters most. postmarketOS
serves dummy `efs1/2/3.bin` over Sahara, and the modem crashes with an
ERRFATAL about 15 s after reaching mission mode. Copy the real Android-era NV
partitions from the device:

```sh
for i in 1 2 3; do
    sudo cp /dev/disk/by-partlabel/mdm1m9kefs$i /lib/firmware/sdx55m/efs$i.bin
done
```

With the real EFS the modem stays up indefinitely.

**2. A TFTP/RFS server on QRTR instance 3.** The modem's remote file system
is `RMTEFS_PACK`; it looks up a TFTP service and every RFS read times out
without one. Stock `tqftpserv` publishes **instance 1**, so it appears to run
fine while the modem logs `could not resolve remote host`. Apply
`tqftpserv-qrtr-instance-3.patch` (publishes instance 3, and retries sends on
`EAGAIN`).

**3. The esoc request engine.** `tools/remoteproc/mdm_helper_native.c`, run as
a service. Without it `mdm_rproc_powerup()` blocks forever on `pon_done`
while holding `rproc->lock`, and `remoteproc0` stops responding to
`start`/`stop` entirely.

**4. Two ModemManager patches**, in `modemmanager/`. `0001` teaches the
qcom-soc plugin about the `mhi_net` kernel driver; `0002` lets a bearer bind
a PCIe data endpoint. ModemManager ≥ 1.25 is also required — older versions
hit a LOC assertion and abort. Run it with `--test-multiplex-requested`, since
NetworkManager 1.52 has no multiplex property.

**5. `CONFIG_RMNET=m`.** Easy to lose: it lives under
`drivers/net/ethernet/`, so `# CONFIG_ETHERNET is not set` silently drops it.
Without QMAP/rmnet multiplexing the downlink wedges permanently after ~10 MB.

**6. `pd-mapper` running, and a SIM physically inserted.** Secure-NV QMI needs
the protection-domain mapper; the UIM stack needs a real card.

**7. Leave `sysmon-probe-native` disabled.** It sends `RESTART_REQ` to SSCTL,
which crashes the modem.

A reference `.config` known to produce a working system is in
`config-apollo-6.17`.

### Crash recovery

A modem crash used to cost a full reboot. It no longer does:

```sh
sudo sh -c 'echo stop  > /sys/class/remoteproc/remoteproc0/state'
sleep 5
sudo systemd-run --unit=rproc-start --no-block \
    sh -c 'echo start > /sys/class/remoteproc/remoteproc0/state'
```

(Use `systemd-run` for the start: a backgrounded ssh command gets killed with
the session cgroup, and the boot takes longer than the session lives.)

Recovery works by warm-resetting the modem through the PMIC, rebuilding the
PCIe link, and then **re-enumerating the device** so probe rebuilds MSI, the
IOMMU mappings and the MHI controller from scratch. Restoring that state
piecemeal does not work.

Afterwards, `systemctl restart ModemManager` — it probes while the modem is
still loading its 85 MB firmware over Sahara, gives up, and marks the modem
invalid.

### Known problems

**Downlink wedges at sustained high rates.** A speedtest reaching ~834 Mbit/s
can leave the downlink permanently stopped, with no host-side error of any
kind: uplink keeps flowing, the QMI control plane stays alive, and no counters
increment. QMAP/rmnet raised the threshold roughly 5–10× over raw IP (which
died at ~10 MB / 87 Mbit/s) without eliminating it. Untried leads: downlink
coalescing via the `IP_HW0_RSC` channel, or a larger aggregation size.

**Do not tear down a call that heavy traffic has already wedged.** `nmcli c
down` sends `WDS Stop Network`, gets no response, and the modem ERRFATALs
about 2 s later.

**Voice call audio does not work.** The q6voice port creates the full CVD
session and every APR command returns status 0, but there is no audio in
either direction. Two things are known to be missing: the topology is
committed with the stock topology IDs while the vendor takes them from ACDB
calibration, and although the MHI satellite proxy completes its handshake
with the ADSP, the ADSP never drives the voice channel.

## Debugging

Packet tracing is compiled in but off, behind dynamic debug — it used to be at
`info` level and made the kernel log useless:

```sh
# QMI/QRTR control plane, decoded
echo "file net/qrtr/mhi.c +p" > /sys/kernel/debug/dynamic_debug/control
# MHI channel activity and doorbells
echo "file drivers/bus/mhi/host/main.c +p" > /sys/kernel/debug/dynamic_debug/control
```

The modem's own bootloader log arrives on the MHI `BL` channel and appears in
dmesg via `mhi_bl`.

## Building

```sh
pmbootstrap init                     # device: xiaomi-apollo
pmbootstrap build --envkernel linux-postmarketos-qcom-sm8250 --force
pmbootstrap install
pmbootstrap flasher flash_kernel
```

Build the device-side tools **on the phone**, not on a cross host:

```sh
cc -O2 -Wall -o mdm_helper_native tools/remoteproc/mdm_helper_native.c
```

## Credit

The ESOC framework and the MHI satellite proxy are ported from Xiaomi's
msm-4.19 vendor kernel for sm8250. q6voice is ported from Stephan Gerhold's
msm8916 work and extended for the newer CVD on sm8250. Original copyright
headers are kept intact.
