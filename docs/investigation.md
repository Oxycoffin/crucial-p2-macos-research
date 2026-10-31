# Native NVMe firmware update through an ASMedia ASM246x bridge: investigation

## Scope and reading guide

This is a chronological technical case study of one successful update on a single Apple Silicon Mac and a Qwiizlab UH61-U4 enclosure. It is intended for researchers, maintainers and AI-assisted investigations. It is **not** a general-purpose flashing procedure or a claim of compatibility with another bridge, firmware build, SSD revision or macOS version. Bridge memory writes and NVMe firmware commands can destroy data or make hardware inaccessible. Back up and verify data independently before any experiment, coordinate active consumers, and never force an unmount. A USB device capture can disrupt attached storage even when the intended command is read-only.

Evidence labels used below: **observed** means an actual device response or post-update verification; **static** means analysis of software or firmware, not proof of hardware behavior; **model** means a software test; **hypothesis** means an idea not demonstrated on this combination. The working program is not published here: it was specialized to one disk, filesystem layout and backup guard, and generalizing those guards without hardware validation would be unsafe. No firmware binaries or device-specific dumps are redistributed.

## Baseline and constraints

- SSD: retail Crucial P2 CT500P2SSD8, 500 GB, firmware P2CR031. Identify capability data: OACS `0017`, FRMW `12` hex, MDTS `6`, FWUG `4`, MTFA `100` (100 ms units, 10 s). FRMW described one writable firmware slot and advertised activation without reset. FWUG is in 4 KiB units: **16 KiB** granularity, not 4 KiB.
- Bridge: Qwiizlab UH61-U4, USB vendor/product `174c:2463`. E4 version bytes `25 04 17 85 1d 03` identify build `250417_85_1D_03`. Separately, USB4 reported 54.17; do not equate these fields. The SSD was reachable as PCIe over USB4, while the successful bridge investigation used USB 2.0 data mode and Bulk-Only Transport (BOT).
- Before firmware writes an encrypted third-destination backup was completed and independently checked: file content SHA-256, paths, links, relevant metadata and protected provenance attributes archived separately in the encrypted backup. The source filesystems were unmounted normally and the SSD identity and image compatibility checked before capture.
- A prior VM USB handoff had caused a kernel panic; it was not repeated for this update. No partitioning, formatting or SSD user-sector raw write was performed.

## Native access and query limitations

Initial libusb legacy capture removed the storage driver but failed to reopen the legacy interface (`IOCreatePlugInInterfaceForService: out of resources`). Cleanup reattached the driver; no CDB was sent. The modern IOUSBHost capture initially failed interface creation with `0xe00002c9`; macOS logs showed a removable-volume TCC denial. Running from the normal user's Terminal under sudo allowed the normal permission path. No SIP or TCC database changes were made. Successful BOT access then returned E6 Identify data for the exact disk. USB4 PCIe discovery by itself did not provide the BOT device and was not proof of firmware command transport.

**Observed:** E6 Identify and Get Log worked; static examination of the bridge's actual firmware showed its E6 dispatcher accepting opcode `06` (Identify) and `02` (Get Log Page), rejecting other opcodes. This finding applies to the read-back build. An Identify response proves a query channel, not support for Firmware Download.

## Storage Executive packet path

**Static:** analysis of Storage Executive 11.11.112025.00 located `mse-api.dll` exports for ASMedia NVMe packet non-data, data-in and data-out operations. A 16-byte SCSI CDB used EA/EB, magic `5A`, phases `80` (command), `81` (data), `82` (completion), and big-endian transfer length in bytes 10–13. A separate normal firmware path selected the same packet transport for a connection-type branch. This established a candidate protocol, not device support.

**Observed:** EB bridge-info and EA packet Identify each returned SCSI sense `05/20/00` (invalid command opcode). Endpoint stall recovery allowed the CSW and REQUEST SENSE to be read. The actual bridge firmware dispatcher had no EA/EB cases. Thus this promising generic route was rejected on this build. E3 is bridge firmware write, not SSD update, and was never issued.

## Inspecting the actual bridge firmware

Generic E4/E2 field assumptions first produced sense `05/24/00`; the ASM246x selector `50` was required. E4 read six version bytes at offset `07F0`. E2 read a 98,304-byte flash **prefix**, whose embedded firmware container passed the observed byte-sum and CRC checks. This was not a full SPI backup. Static disassembly traced the vendor dispatcher, E6 opcode checks and E5 byte-write handler. Its ordinary 16-bit XDATA form was `E5 value 50 address_hi address_lo 00`. These conclusions are specific to this firmware body and mapping.

Static tracing also suggested queue-window selection through C8D7/C8D8 and an admin submission route via the B296/B251/B254 registers. A custom-firmware project provided hypotheses about the memory map; its F0/F2 commands were not present in this stock bridge. Static references alone did not establish working DMA, completion ownership or safe queue manipulation.

## From RAM experiment to fresh Identify

After the backup, **observed:** a reversible E5 write changed one byte in the identified RAM window, E4 confirmed it, then E5 restored the original byte and E4 confirmed restoration. With saved selector values restored afterwards, the selected admin window exposed an Identify SQE with opcode `06`, NSID 0, PRP1 `00200000`, and CNS 1.

The next experiment injected only an Identify SQE. The first 64 data bytes were deliberately changed before submission, the next admin slot was submitted and a new completion and CQ-head advance observed. All 4096 bytes of resulting Identify data matched the established E6 baseline, including replacement of the altered prefix. This provided evidence of a fresh admin command and DMA, not merely a stale E6 buffer. After a doorbell submission the queue indices were not rolled back; selector restoration was checked. The result did not yet validate a 16 KiB firmware data buffer.

## Firmware provenance and staging

The Storage Executive firmware service returned P2CR033 for the exact model and P2CR031 precondition, pointing to [Micron's HTTPS package](https://www.micron.com/content/dam/micron/global/public/ssdtool/firmware/p2/33/fwbin.zip). ZIP size 1,513,321 bytes; SHA-256 `cb81ad5910fcb9dadae2780fcd01100e22d81ad572aa16e3dbdd22daeef513f8`. Its properties specified model P2, type CR and P2CR031 precondition. Expanded `P2CR033/1.bin` was 1,523,712 bytes, SHA-256 `6f3ca03e842a879b56ee7b3b5c17f1c97f1726f856634fcb8ec32dfc6230c7d6`: exactly 93 × 16 KiB. These hashes identify acquired bytes, not independently signed vendor checksums.

The firmware image was stored as a file on the SSD. APFS `F_LOG2PHYS_EXT` file extents were translated to physical-device sectors by including the physical-store partition offset. **Observed:** a stock SCSI READ(16) of 32 sectors returned the expected first 16 KiB. Its corresponding I/O SQE reported NVMe read opcode `02`, NSID 1, NLB 31, PRP1 `00200000`, PRP2 `03201000`. This offered a staging path without raw user-sector writes. Every returned 16 KiB was checked against the expected image bytes; the visible 4 KiB window was rechecked after SQE construction. The remaining PRP pages were not separately read through the CPU-visible window, so no stronger claim is made.

## Ambiguous first download and correction

One bounded first-block Firmware Download experiment had no Commit path. On release, libusb teardown deadlocked against a hotplug callback; buffered final output was lost. Later success flags could not distinguish every intervening admin command, so the first-block result remains **unknown**. It is not counted as a success. A full power cycle preceded the complete update to clear the ambiguous staging state. Discovery's libusb context was then closed before IOUSBHost capture, and output was made unbuffered; the subsequent native flow exited normally. A submitted command was never blindly retried after an ambiguous outcome.

## Complete update and verification

Before the first complete-update firmware write, the updater read all mapped image extents and checked the complete SHA-256. It then re-read and checked each 16 KiB chunk and the hardware-generated PRPs. It submitted 93 NVMe Firmware Image Download commands (opcode `11`, NUMD `4095`, offset increasing in 4096-dword units); all returned successful completions in about 10.2 seconds. Only after all 93 successes did it submit Firmware Commit (opcode `10`, slot 1, commit action 1, CDW10 `9`). Completion reported SCT 0 / SC 0. The firmware-slot log showed P2CR033 in slot 1. Slot contents alone do not establish the running version.

After a complete power-off and reconnection via USB4, a fresh native Identify showed **active P2CR033**. Both APFS volumes passed macOS volume verification and independent read/write checks. SMART reported no critical warning or media/data-integrity errors; historical error and unsafe-shutdown counters were unchanged. This establishes success for this particular unit, not for all configurations. No bridge firmware was flashed.

## Test coverage and limitations

Software models covered RAM restoration and selector mapping, injected Identify, the full 93-block sequence with one commit, refusal after a failed download, and refusal of an incorrect image. These are **model tests**, not additional physical-device trials. The successful hardware update was one end-to-end run. Transport status, lengths, matching tags, SCSI sense and NVMe completion status were checked; errors stopped the sequence.

Afterwards, an 8 GiB file benchmark over USB4 measured approximately 2102 MB/s sequential read and 1127 MB/s write under the chosen workload, with no device errors or retries. This short test does not establish sustained post-cache performance. A separate 600-second passive observation recorded no detach, but background I/O limited the longest continuous idle interval to about 220 seconds. Neither proves that the user's earlier intermittent expulsions are fixed.

## Implications for other hubs and future tools

The generalizable idea is a **research workflow**, not a universal exploit: identify the actual USB/PCIe topology and firmware build; distinguish query support from arbitrary admin command support; inspect official utilities and actual bridge behavior; validate a benign fresh command and completion; establish correct DMA ownership and firmware alignment; then consider any write only with an independently verified backup and a compatible authentic image. Querying a bridge ID or successfully reading Identify is insufficient evidence of write safety.

Other bridges may expose a supported pass-through, reject these vendor commands, use a different queue layout, use different buffer mappings, or lack any comparable mechanism. The addresses, selectors, PRPs and queue sizes above are observations for this one stock build, not safe defaults. Do not cross-flash bridge firmware on the basis of matching USB IDs. A reusable updater would need parameterized identity and compatibility checks, robust topology/consumer checks, backup verification independent of one user's paths, authenticated firmware provenance, non-retry behavior for ambiguous completions, recovery design, and independent testing across each supported hardware combination. A future release must clearly distinguish simulated coverage from real-device trials.

## References

- [ASM2x6x reverse engineering notes](https://github.com/cyrozap/usb-to-pcie-re/blob/master/ASM2x6x/doc/Notes.md), including ASM246x selector and memory information.
- [tinygrad ASM2464 firmware](https://github.com/tinygrad/asm2464pd-firmware), used for static mapping hypotheses, not installed on this hub.
- [NVMe 1.4c specification](https://nvmexpress.org/wp-content/uploads/NVM-Express-1_4c-2021.06.28-Ratified.pdf).
- [NVMe Identify controller fields](https://learn.microsoft.com/en-us/windows/win32/api/nvme/ns-nvme-nvme_identify_controller_data) and [SPDK controller data](https://spdk.io/doc/structspdk__nvme__ctrlr__data.html).
