# Changelog

All notable changes to usb-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `usbdesc` — the device, configuration, interface, endpoint and
  interface-association descriptors as `@value` records that answer
  their own wire bytes, the string descriptors as functions over a
  `Str`, the endpoint address constructors, and the packet sizes the
  specification allows per transfer type and speed.
- `usbctl` — the setup packet, the standard requests, the six device
  states, the thirteen actions, and `UsbCtlStep`.
- `usbcdc` — CDC-ACM: the line coding and its seven bytes, the two
  interfaces and the association that binds them, the four functional
  descriptors, DTR and RTS, and the serial state notification.
- `usbhid` — HID: the report descriptor as a typed item builder with
  thirteen named item constructors, the HID descriptor, and the boot
  keyboard and mouse reports.
- `usbbos` — the binary object store, the WebUSB platform capability
  and its URL descriptors, and the MS OS 2.0 capability with the WinUSB
  compatible ID.
- `usberr` — one error type for descriptor faults and transfer faults
  alike, with `is_recoverable` as the distinction a retry loop needs.
- `usbxfer` — `UsbHostBackend[e]`, the device identifier that survives
  a replug, and the standard requests written once over the trait.
- `usbenum` — the `host` module: what is plugged in, read from
  `/sys/bus/usb/devices/`, `[fs]`.

### Known

- **The load-bearing interface is `UsbCtlStep`'s two action fields.**
  USB 2.0 § 9.4.6 requires SET_ADDRESS to take effect after the status
  stage, and a machine that answers one action has no moment that
  corresponds to it. `step_deferred` is that moment.
- **The layer is `core` with one `host` module**, not the plan's
  `host`. The subject is the device half, and declaring the narrow
  layer is what makes the device claim something the audit builds.
- **A `@value` struct cannot hold an enum** (E2011), so the actions,
  the device states, the descriptor sources and the CDC parity and
  stop-bit codes are all `Int` discriminants with named constants as
  the vocabulary.
- **A `@value` struct cannot be a boxed struct's field** (E2015), so
  `usbenum.UsbDeviceInfo` carries the four identifier numbers flat and
  `info_id` assembles a `UsbDeviceId` on demand.
- **A descriptor answers its bytes one at a time**, because a buffer
  per GET_DESCRIPTOR is an allocation a device does not have and a
  control endpoint chops it up again anyway.
- **`UsbHostBackend` has no implementation in this project.** The
  missing rows are `libusb-sys` on the bindings shelf and a native
  usbfs backend the language cannot write yet; the README names both.
- **The device claim covers `usbdesc` and `usbctl`** and
  `tests/embedded_probe.nv` builds them for a Cortex-M4.
- A peripheral driver, mass storage, audio, MIDI and USB 3 are
  deliberately outside.

### Design notes

One more missing row the README no longer names: UTF-16LE in the
standard library.  String descriptors are the only place USB uses it,
and `usbdesc.string_desc_byte` encodes one code point at a time for
want of anywhere better to put the conversion.
