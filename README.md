# usb-nv

**USB** is the bus a keyboard, a serial adapter or a firmware-update port
speaks over. A **device** on it answers requests from a **host**; it never
starts a transfer of its own. This package is the device side of USB 2.0 in
novo-lang, with no peripheral driver in it: the standard descriptors, the
control-transfer state machine, the serial and human-interface classes, and a
host-side surface beside them. The device half is a port of
[usb-device](https://github.com/rust-embedded-community/usb-device) and the
specification is USB 2.0 chapter 9.
[dfu-nv](https://novo-lang.org/packages/dfu-nv) is built on it.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What USB looks like from the device

A **descriptor** is a block of bytes describing part of the device. The host
reads them to find out what it has plugged in. The device descriptor names
the vendor and the product. The configuration descriptor is followed by the
interface and endpoint descriptors belonging to it, and its length field
covers all of them. A string descriptor holds text, encoded as UTF-16.

An **endpoint** is one channel. Endpoint 0 exists on every device and carries
**control transfers**, which is how the host asks everything. A control
transfer has up to three parts: a **setup stage** of eight bytes saying what
is wanted, an optional **data stage**, and a **status stage** of zero bytes
that says it is over.

**Enumeration** is what the host does when a device appears. It reads the
first eight bytes of the device descriptor, resets the device, gives it an
address, reads the descriptors properly, and selects a configuration. Until
the address is set the device answers on address 0.

A **class** is a standard behaviour a device declares so that the host
already has a driver. Two are here. **CDC-ACM** makes the device a serial
port. **HID** makes it a keyboard, a mouse, or anything else described by a
**report descriptor**, which is a small stack-based language saying what the
bytes of a report mean.

The **binary object store** is a later descriptor, read from a device that
says it is USB 2.1 or above, carrying capabilities that did not fit the
original scheme: WebUSB and the Microsoft OS 2.0 descriptors among them.

| Quantity | Value |
| --- | --- |
| Bytes in a setup packet | 8 |
| Bytes in a device descriptor | 18 |
| Bytes in an endpoint descriptor | 7 |
| Address before enumeration | 0 |
| Actions the control machine can ask for | 13 |
| Largest packet on a full-speed control endpoint | 64 |
| Classes implemented here | 2 |

## Install

```
novo pkg add usb-nv
```

## Example

```novo
use usbdesc
use usbctl

fn main() [io]
    // The device descriptor a host reads first. Its first two bytes are not
    // arguments: a device descriptor is eighteen bytes and type 1, always.
    let dev = usbdesc.device_desc(
        0x0210,                                 // the USB version this device answers as
        usbdesc.USB_CLASS_MISC, 0x02, 0x01,     // a composite device, described by an IAD
        64,                                     // the largest packet endpoint 0 accepts
        0x1209, 0x0001, 0x0100,                 // vendor, product, device version
        1, 2, 3,                                // the three string descriptor indices
        1)                                      // how many configurations there are

    // A control endpoint that moves sixty-four bytes at a time.
    var ctl = usbctl.control(64)

    // The host's first request asks for eight bytes, because it does not yet
    // know how long the descriptor is.
    let s = usbctl.setup(0x80, usbctl.USB_REQ_GET_DESCRIPTOR, 0x0100, 0, 8)
    let step = usbctl.step(ctl, s, usbdesc.USB_DEVICE_DESC_LEN)

    // What to do now, how many bytes, and whether a zero-length packet must
    // follow. The driver fills the endpoint straight from the descriptor.
    println("${usbctl.action_name(usbctl.step_action(step))} ${usbctl.step_len(step)}")
    for i in 0..usbctl.step_len(step)
        println("${usbdesc.device_desc_byte(dev, i)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails
on purpose: every test reaches a `not implemented: usb-nv.<module>.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `usbdesc` | The standard descriptors as values that answer their own wire bytes one byte at a time, and the string descriptors as functions over text. |
| `usbctl` | The setup packet, the standard requests, and the control-transfer state machine with its thirteen actions. |
| `usbcdc` | CDC-ACM: the line coding, the two interfaces, the functional descriptors, and the serial state notification. |
| `usbhid` | HID: the report descriptor as typed items with a constructor each, the HID descriptor, and the boot keyboard and mouse reports. |
| `usbbos` | The binary object store, with WebUSB and the Microsoft OS 2.0 descriptors inside it. |
| `usberr` | One error type, for descriptors and transfers alike. |
| `usbxfer` | The host's view of a bus: a backend trait with an effect parameter, and the standard requests written once over it. |
| `usbenum` | What is plugged in, read from the system's own record of it. |

## How to choose an entry point

**A device links `usbdesc` and `usbctl`**, plus whichever of `usbcdc`,
`usbhid` and `usbbos` its product needs. None of those five declares an
effect.

**A host tool that only asks what is plugged in links `usbenum`.** It reads
the operating system's own listing and costs one file-system effect.

**A host tool that transfers implements `usbxfer.UsbHostBackend[e]`.** The
module declares no effect of its own: every function that drives a backend
takes its effect as a parameter, so the same code costs whatever the caller's
backend costs. This project satisfies that trait nowhere yet. See "What is
not included".

**Build a HID report descriptor with `usbhid`'s items, not by copying
hexadecimal.** Every tutorial shows a wall of bytes taken from another
device. A host that disagrees with a report descriptor reports no error at
all: it simply delivers nothing.

## The rules a user needs

1. **A device that receives SET_ADDRESS keeps answering on address 0 until
   the status stage of that transfer has completed** (USB 2.0 section 9.4.6).
   `usbctl.step` answers two actions for this: `step_action` is what to do
   now, and `step_deferred` is what to do when the status stage completes.
   `USB_ACT_SET_ADDRESS` is the only value the second ever takes. A stack
   that writes the address register when the setup packet arrives sends its
   status packet from the new address, the host never sees it, and the device
   enumerates on some hosts and not others.
2. **A data stage shorter than the host asked for must end with a
   zero-length packet when its length is a whole multiple of the endpoint's
   packet size.** Otherwise the host waits for a transfer that never ends.
   `step_needs_zlp` says when.
3. **SET_CONFIGURATION with a value of 0 is not a selection.** It means
   disable every endpoint but 0 and return to the addressed state.
   `USB_ACT_UNCONFIGURE` is that action, and it is separate from
   `USB_ACT_SET_CONFIGURATION`.
4. **CLEAR_FEATURE on an endpoint halt resets the data toggle as well.** An
   endpoint resumed on the wrong toggle has every packet discarded, silently.
   `USB_ACT_CLEAR_HALT` is both halves.
5. **A descriptor answers its bytes one at a time, and there is no buffer.**
   A full-speed control endpoint moves at most sixty-four bytes per packet,
   so a buffer built for one request would be chopped up again immediately.
   `usbdesc.device_desc_byte(d, i)` needs no buffer. A host tool that wants
   the whole descriptor loops.
6. **The first two bytes of a descriptor are not arguments.** The length and
   the type are decided by which descriptor it is: an endpoint descriptor is
   seven bytes and type 5, always. A caller cannot build a descriptor that
   disagrees with itself, and a configuration descriptor whose total length
   is wrong is a device that enumerates on one host and not another.
7. **An action is an integer, not an enum value.** A `@value` struct cannot
   hold an enum field at all (E2011), and `UsbCtlStep` is unboxed so that a
   driver can answer a setup packet inside an interrupt with nothing on a
   heap. The `USB_ACT_*` constants are the vocabulary and
   `usbctl.action_name` turns a code back into words for a log.
8. **The endpoint table is yours.** A device stack that owned one would have
   to know how many interfaces a product has before the product exists.
9. **String descriptors are UTF-16, little-endian.**
   `usbdesc.string_desc_byte` encodes one code point at a time. USB is the
   only place this project meets that encoding.

## Running on a microcontroller

The package states that its modules run on a device with no heap allocator,
and the compiler checks that claim on every build. Seven of the eight modules
declare no effects and are inside it.

`tests/embedded_probe.nv` is that claim as a program that either builds or
does not. It covers `usbdesc` and `usbctl`, which is what a device stack runs
in its interrupt handler.

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

That command was run against this release. It produces a Cortex-M4
executable, `embedded_probe.elf`. The probe builds; it is not run, because
every function it calls is a `todo()` that would panic on the first line.

`usbenum` is outside the claim. It reads the file system, and one host-only
function anywhere in a compilation unit is an undefined symbol at link time
on a device, whether or not the firmware calls it.

`usbxfer` stays inside the claim even though it is the host's view of a bus,
because every effect in it is a parameter. A package that declares a transfer
surface has not performed a transfer.

## What is not included

- **A peripheral driver.** Nothing here writes a register. The nRF52840 and
  the RP2040 both have a full-speed device controller, and the code that
  drives one belongs with the board.
- **A host transfer implementation.** `usbxfer.UsbHostBackend[e]` is a
  contract that nothing in this project satisfies yet. Publishing it before a
  provider exists is deliberate: the request shapes and the error cases get
  reviewed before somebody spends a month on transfer submission. Until then
  a host tool built on this package can list devices and cannot talk to them.
- **Mass storage, audio, MIDI and the other classes.** Each would be a module
  over the same device model rather than a change to it.
- **USB 3.** SuperSpeed has its own descriptors, its own link layer and its
  own endpoint companion descriptors. The capability type for it is named in
  `usbbos` and nothing more.
- **A buffer for anything.** See rules 5 and 8.
  [heapless-nv](https://novo-lang.org/packages/heapless-nv) is where a
  firmware's endpoint tables and transmit buffers come from.

## Related packages

- [dfu-nv](https://novo-lang.org/packages/dfu-nv) is firmware update over
  this package's control endpoint. It carries the DFU 1.1 state machine and
  is written against these descriptor and setup types, so the two cannot
  disagree about a byte.
- [heapless-nv](https://novo-lang.org/packages/heapless-nv) is the
  fixed-capacity storage a device stack pairs this with.
- [bitfield-nv](https://novo-lang.org/packages/bitfield-nv) describes the
  registers of a board's USB controller. Nothing in this package touches
  memory.
- [can-nv](https://novo-lang.org/packages/can-nv) and
  [modbus-nv](https://novo-lang.org/packages/modbus-nv) are the other buses
  on the registry. Both are peer-to-peer or request-and-reply; USB has one
  host that starts every transfer.
- `std.fs` in the standard library is what `usbenum` reads.

## Tests

```bash
novo test                             # 114 tests
novo test tests/usbctl_tests.nv       # 22: the state machine and the four orderings
novo test tests/usbdesc_tests.nv      # 15: the descriptors, byte for byte
novo test tests/usbcdc_tests.nv       # 16: the serial class
novo test tests/usbhid_tests.nv       # 16: the report descriptor items
novo test tests/usbreaders_tests.nv   # 12: reading a descriptor back
novo test tests/usbbos_tests.nv       # 11: the binary object store
novo test tests/usbhost_tests.nv      # 10: the host-side requests
novo test tests/usbenum_tests.nv      #  4: the listing
novo test tests/usbxfer_tests.nv      #  4: the backend trait's shape
novo test tests/usberr_tests.nv       #  4: the error type
```

The references are USB 2.0 chapter 9 for the descriptors, the standard
requests and the device states, the CDC 1.2 specification for the serial
class, and HID 1.11 with its appendix B for the human-interface class. The
device behaviour follows `usb-device`, including its handling of the deferred
address. [libusb](https://libusb.info/) is the reference for the host half.

No test opens a bus. Every setup packet is a value the test writes out and
every descriptor is compared byte for byte.

The tests compile today and fail at run, each on the
`not implemented: usb-nv.<module>.<fn>` panic that is its body. That is the
expected state of an interface release. They turn green one at a time as
bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| Every `USB_*` constant in `usbdesc`, `usbctl`, `usbcdc`, `usbhid`, `usbbos` | yes (they are constants) |
| The descriptor, setup, step, class and transfer value types | declared |
| `usberr.UsbError`, `usbenum.UsbDeviceInfo`, `usbxfer.UsbHostBackend[e]` | declared |
| `usbdesc`: the constructors and the byte readers | no |
| `usbctl`: the setup packet, the requests and the state machine | no |
| `usbcdc`: the line coding, the descriptors and the notification | no |
| `usbhid`: the report items, the HID descriptor and the boot reports | no |
| `usbbos`: the store, WebUSB and the Microsoft OS 2.0 descriptors | no |
| `usberr.describe` | no |
| `usbxfer`: the standard requests over a backend | no |
| `usbenum`: the listing | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
