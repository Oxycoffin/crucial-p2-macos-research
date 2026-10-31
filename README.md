# Updating Crucial P2 firmware from Apple Silicon macOS through an ASMedia USB bridge

A native macOS investigation: a Crucial P2 **CT500P2SSD8, 500 GB** was updated from **P2CR031 to P2CR033** while installed in a Qwiizlab UH61-U4. After a complete power cycle, a fresh NVMe Identify query reported P2CR033 as the active firmware. This is a single-device research result, not a general compatibility claim.

**Status: research documentation, not a downloadable updater.** The working experimental updater was hard-coded to one machine, disk and verified backup. It has not yet been adapted and validated for public use. Do not attempt to reproduce the write sequence from this report on important data. Firmware updates can render a disk inaccessible. Make and verify an independent backup first.

## Tested configuration and result

- Apple Silicon Mac, macOS; native host-side code using IOUSBHost and libusb for discovery.
- Crucial P2 CT500P2SSD8, original firmware P2CR031, authentic P2CR033 image.
- Qwiizlab UH61-U4 bridge, ASMedia USB vendor/product `174c:2463`, bridge firmware build `250417_85_1D_03` as read through E4. The device separately advertised USB4 version 54.17. These are distinct identifiers.
- Update over a USB 2.0 data connection; after activation the normal USB4/PCIe connection was restored. No Windows virtual-machine USB passthrough was used for the update.
- Full image: 1,523,712 bytes, SHA-256 `6f3ca03e842a879b56ee7b3b5c17f1c97f1726f856634fcb8ec32dfc6230c7d6`. It was obtained through Crucial Storage Executive's firmware service from [Micron's official package](https://www.micron.com/content/dam/micron/global/public/ssdtool/firmware/p2/33/fwbin.zip). The hash identifies the image we used; it is not an independently published vendor signature.
- All 93 firmware download commands completed successfully; the firmware commit completed successfully. The active revision was confirmed by a fresh Identify after a full power cycle. Both APFS volumes passed macOS volume verification and basic read/write checks afterwards.

The bridge firmware was **not** updated. SMART's media/data integrity errors were zero after the update. This does not prove future reliability or that unrelated intermittent disconnections are fixed.

## What worked

The ASMedia E6 tunnel could read Identify and health information but, on this bridge build, accepted only the observed NVMe query opcodes. It is not a general NVMe firmware-download channel. The EA/EB packet path found by static analysis of Storage Executive was rejected by this device. ASMedia E3 writes the **bridge firmware**, not the SSD firmware; it was not sent.

The eventual approach used the bridge's E4/E5 memory access and reverse-engineered queue and doorbell behavior to submit native NVMe admin commands. A controlled Identify command was first injected and its 4096-byte response checked against the query baseline. The firmware was then staged in 16 KiB chunks using standard SCSI READ(16) from a preverified file's physical extents, with the returned bytes and DMA pointers checked. Each chunk was submitted with NVMe Firmware Image Download (`0x11`), followed only after all 93 successful completions by Firmware Commit (`0x10`, slot 1, action 1). The slot log showed the new revision; a power cycle and independent fresh Identify established that it was active.

The device reported FWUG=4, corresponding to 16 KiB download granularity. This staging method relied on exact bridge behavior, memory mapping, queue state, APFS physical extents and the tested controller. Those assumptions must not be generalized to other builds or disks. Flashing a bridge image is a different operation.

## What has and has not been verified

| Finding | Scope |
|---|---|
| Firmware active after power cycle | One P2 CT500P2SSD8 and the bridge build above |
| 93 download completions and one successful commit | One successful complete update |
| Volume verification and read/write checks | Performed after update; not a guarantee of future data integrity |
| Firmware package provenance | Official Micron HTTPS endpoint; local SHA-256 verification |
| Other capacities, revisions, bridges or macOS versions | Not tested |
| Generic public updater | Not yet available |
| Historical spontaneous disconnections | Cause and resolution not established |

The original work included a verified encrypted backup before writing, coordinated unmounting, authentication of the complete image and each staged chunk, and stopping on unexpected outcomes. Do not interpret the protocol overview as a copy-paste flashing guide. In particular, NVMe Identify access alone does not establish firmware update capability.

## References

- [ASMedia ASM2x6x reverse-engineering notes](https://github.com/cyrozap/usb-to-pcie-re/blob/master/ASM2x6x/doc/Notes.md) — bridge memory maps and vendor commands.
- [Crucial P2 product information](https://www.crucial.com/content/dam/crucial/ssd-products/P2/flyer/crucial-P2-productflyer-consumer.pdf).
- [NVM Express 1.4c specification](https://nvmexpress.org/wp-content/uploads/NVM-Express-1_4c-2021.06.28-Ratified.pdf) — firmware download and commit commands.

This independent research is not affiliated with Crucial, Micron, ASMedia, Apple or Qwiizlab.
