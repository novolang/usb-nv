# usb-nv

**Status: NOT IMPLEMENTED — interface only.**

The USB 2.0 device side, with no peripheral attached: the standard
descriptors as values that answer their own wire bytes, the
control-transfer state machine with the standard requests, CDC-ACM and
HID as classes over one device model, and a host-side enumeration and
transfer surface beside them.

Every `pub fn` body is a `todo()`. The signatures and the effect rows
are published so the design can be reviewed and effect-checked before
anyone writes a body against it; `novo pkg add usb-nv` resolves,
downloads and builds, and the first call panics with
`not implemented: usb-nv.<module>.<fn>`.

## The one example that will work

A full-speed CDC-ACM device — a serial port over USB — answering the
first request every host makes:

```novo
use usbdesc
use usbctl

// The device descriptor a host reads first.  bLength and
// bDescriptorType are not fields: 18 and 1, decided by which
// descriptor this is.
let dev = usbdesc.device_desc(
    0x0210,                                 // bcdUSB — 2.1, so a host asks for the BOS
    usbdesc.USB_CLASS_MISC, 0x02, 0x01,     // a composite device described by an IAD
    64,                                     // bMaxPacketSize0
    0x1209, 0x0001, 0x0100,                 // idVendor, idProduct, bcdDevice
    1, 2, 3,                                // the three string indices
    1)                                      // bNumConfigurations

// A host's first GET_DESCRIPTOR asks for eight bytes, because it does
// not yet know how long the descriptor is.
var ctl = usbctl.control(64)
let s = usbctl.setup(0x80, usbctl.USB_REQ_GET_DESCRIPTOR, 0x0100, 0, 8)
let step = usbctl.step(ctl, s, usbdesc.USB_DEVICE_DESC_LEN)
// step_action is USB_ACT_DATA_IN, step_len is 8, step_needs_zlp is
// false — and the driver fills the endpoint FIFO straight out of
// usbdesc.device_desc_byte(dev, i), with no buffer anywhere.
```

## Build, run and test

```bash
novo pkg build          # type-check and effect-check every module; a library, so no binary
novo test               # the API suites — RED until the bodies land
novo doc .              # the reference page, with every example block compiled
```

There is nothing to run: this is a library, and at 0.0.1 every body is
a `todo()`, so `novo test` is red on purpose and the suites are the
specification written as assertions.

## The layer, and why

`core`, with one `host` module named in the manifest.

The must-have plan placed this row at `host` because it saw the driver
half. The subject is the other half: a USB device stack is what a
microcontroller runs, and descriptors, the control-transfer state
machine, CDC-ACM and HID are all arithmetic over bytes the caller
already holds. `docs/publishing.md` § A package with a core and a host
half is the shape — the narrow layer declared, the wide module named —
and it is what makes the device claim a thing the audit **builds**
rather than a sentence here.

- **A device links** `usbdesc`, `usbctl`, and whichever of `usbcdc`,
  `usbhid` and `usbbos` its product needs. All five are `[]`
  throughout, and `tests/embedded_probe.nv` builds the first two for
  `--target=nrf52-qemu`.
- **A laptop links** `usbenum` — the one module with a row, `[fs]` —
  and `usbxfer`, whose driving functions are effect-polymorphic over
  `UsbHostBackend[e]` and cost whatever the caller's backend costs.
  A host tool that only wants to know what is plugged in links
  `usbenum` and nothing else.

`usbxfer` stays `core` even though it is the host's view of a bus,
because every row in it is an effect **parameter**. A package that
declares a transfer surface has not performed a transfer.

## The load-bearing interface

**`UsbCtlStep`, and specifically its two action fields.**

USB 2.0 § 9.4.6: a device that receives SET_ADDRESS keeps answering on
address 0 until the **status stage** of that transfer has completed,
and only then adopts the new address. A stack that writes the address
register when the SETUP packet arrives sends its status packet from the
new address, the host never sees it, and the device enumerates on some
hosts and not on others depending on how fast their controller is.

The bug is unwritable in a design with two fields and invisible in a
design with one. `handle_setup(packet) -> Response` has no moment in it
that corresponds to "the status stage completed", so the ordering has
to live in a comment. Here `step_action` is performed now and
`step_deferred` is performed when the status stage completes, and
`USB_ACT_SET_ADDRESS` is the only value the second field ever takes.

The same shape pays for three more orderings that are each a day of
somebody's life:

- a GET_DESCRIPTOR shorter than `wLength` whose length is a whole
  multiple of the endpoint packet size **must** be followed by a
  zero-length packet, or the host waits for a transfer that never ends
  — `step_needs_zlp`;
- SET_CONFIGURATION(0) is not "select configuration 0", it is "disable
  every endpoint but 0 and go back to the address state" —
  `USB_ACT_UNCONFIGURE`;
- CLEAR_FEATURE(ENDPOINT_HALT) resets the data toggle as well as the
  halt, and an endpoint resumed on the wrong toggle has every packet
  discarded silently — `USB_ACT_CLEAR_HALT` is both halves.

`UsbCtlStep` is a `@value` struct, which is what lets a driver answer a
SETUP packet inside the USBD interrupt with nothing on a heap. That
also forced the actions to be `Int` codes with `USB_ACT_*` as the
vocabulary rather than an enum: a `@value` struct cannot hold an enum
at all, even a payload-free one (E2011). `usbctl.action_name` turns a
code back into words for a log.

## The reference implementation

[`usb-device`](https://github.com/rust-embedded-community/usb-device)
for the device half — its `UsbDevice`, `ControlPipe` and class traits
are what this is a port of, and its handling of the deferred address is
the behaviour the state machine copies. [`libusb`](https://libusb.info/)
is the reference for the host half, and the shelf's `libusb-sys` is the
binding that would implement `UsbHostBackend` first. The descriptor
layouts, the standard requests and the device states are USB 2.0 § 9;
CDC-ACM is the CDC 1.2 specification; HID is HID 1.11 and its appendix
B.

## What is here

| Module | Layer | What it is |
| --- | --- | --- |
| `usbdesc` | `core` | The standard descriptors as `@value` records that answer their own wire bytes, plus the string descriptors as functions over a `Str`. |
| `usbctl` | `core` | The setup packet, the standard requests, and the control-transfer state machine. |
| `usbcdc` | `core` | CDC-ACM: the line coding, the two interfaces, the functional descriptors, the serial state notification. |
| `usbhid` | `core` | HID: the report descriptor as a typed item builder, the HID descriptor, and the boot keyboard and mouse reports. |
| `usbbos` | `core` | The binary object store, and WebUSB and MS OS 2.0 inside it. |
| `usberr` | `core` | One error type for descriptors and transfers alike. |
| `usbxfer` | `core` | `UsbHostBackend[e]` and the standard requests written once over it. |
| `usbenum` | `host` | What is plugged in, read from `/sys/bus/usb/devices/`. `[fs]`. |

## Three things the design decided, and why

**A descriptor answers its bytes one at a time.** The obvious shape,
`to_bytes(d) -> [u8]`, is wrong twice on a device: it allocates a
buffer per GET_DESCRIPTOR, and a full-speed control endpoint moves
eight bytes at a time, so the buffer is immediately chopped up again.
`device_desc_byte(d, i)` needs no buffer at all. A host tool that wants
the whole thing loops.

**The first two bytes of a descriptor are not fields.** `bLength` and
`bDescriptorType` are decided by which descriptor it is — an endpoint
descriptor is seven bytes and type 5, always. Carrying them as fields
would let a caller build a descriptor that disagrees with itself, and a
configuration descriptor whose `wTotalLength` is wrong is a device that
enumerates on one host and not another.

**A report descriptor is built, not written.** Every HID tutorial shows
a wall of hex copied from another device. The format is a small
stack-based language, a copied wall works until a field moves, and a
host that disagrees with a report descriptor reports no error at all —
it just delivers nothing. `usbhid`'s items are values with one
constructor each.

## What a firmware pairs this with

- **heapless-nv** for the endpoint tables and the transmit buffers. An
  endpoint table is the caller's storage on purpose: a device stack
  that owned one would have to know how many interfaces a product has
  before the product exists.
- **dfu-nv** for firmware update over this package's control endpoint.
  It depends on usb-nv and carries the DFU 1.1 state machine.
- **bitfield-nv** where a board's USBD peripheral registers need
  describing. Nothing in this package touches memory; the MMIO reads
  and writes are `orbit/hal`'s.

## What is deliberately outside

- **A peripheral driver.** Nothing here writes a register. The nRF52840
  and the RP2040 both have a full-speed USB device controller, and the
  code that drives one belongs with the board, not with the protocol.
- **Mass storage, audio, MIDI, and the rest of the classes.** CDC-ACM
  and HID are the two the plan named, and each new class is a module
  over the same device model rather than a change to it.
- **USB 3.** SuperSpeed's descriptors, its link layer and its
  endpoint companion descriptors are a different specification, and the
  BOS capability type for it is named here and nothing more.
- **A host transfer implementation.** `UsbHostBackend` is a contract
  and this project satisfies it nowhere. Publishing it before either
  provider exists is the point: the request shapes and the error cases
  get reviewed before somebody spends a month on URB submission.

## Missing rows this package found

- **`libusb-sys`** on the bindings shelf is the backend
  `UsbHostBackend` is shaped for, and it does not exist yet. Until it
  does, a host tool built on this package can enumerate and cannot
  transfer.
- **A native usbfs backend** — `USBDEVFS_SUBMITURB` through an ioctl —
  is the other provider, and the language has no way to make an ioctl
  today. It is worth naming because it is the one that would keep a USB
  tool on the grid rather than on the shelf.
- **UTF-16LE in the standard library.** String descriptors are the only
  place USB uses it, and `usbdesc.string_desc_byte` encodes a code
  point at a time for want of anywhere better to put the conversion.

## Licence

Apache-2.0.
