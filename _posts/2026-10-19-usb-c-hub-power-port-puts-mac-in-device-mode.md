---
layout: post
title: "Why a Powered USB-C Hub Made the Attached Keyboard Disappear"
date: 2026-10-19 21:45:00 +0530
categories: macos usb hardware
---

Plugging a power cable into a USB-C hub's pass-through charging port made the USB keyboard attached to that hub vanish from macOS.

## Problem

On an M1 MacBook Air (`MacBookAir10,1`), both USB-C ports were occupied:

- **Port 2** drove an external `MSI MP223 E2` monitor through a Chrontel `CH7213` USB-C display adapter.
- **Port 1** held a multiport USB-C adapter (USB-A + HDMI + USB-C power pass-through) with a wired USB keyboard (`Sino Wealth`, `0x258a:0x013b`) and a power cable connected to the hub's charging port.

The Mac reported `Now drawing from 'AC Power'`, yet `ioreg -p IOUSB` and `hidutil list` showed no external keyboard and no USB hub on Port 1.

## What `ioreg` Showed

Inspecting the `IOPort`, `IOUSB`, and `IOService` planes alongside `AppleSmartBattery` exposed why the keyboard was invisible:

| Metric | Port 1 (Power Plugged Into Hub) | Port 1 (Power Unplugged From Hub) | Port 2 (Display Adapter) |
|---|---|---|---|
| `PortDataRoleDescription` | `Device` | `Host` | `Host` |
| `IOAccessoryUSBConnectString` | `Host` (partner is Host) | `Device` (partner is Device) | `Device` (partner is Device) |
| Active USB Controller | `AppleT8103USBXDCI` (Device) | `AppleT8103USBXHCI` (Host) | `AppleT8103USBXHCI` (Host) |
| `AppleT8103USBXHCI` Power State | `0` (Off) | `3` (On) | `2` (On) |
| `Power In` / Adapter | `5V / 1500mA` (`7.5W` passive `TypeC`) | None (Mac sourcing `VBUS`) | None (Mac sourcing `VBUS`) |
| USB-PD `SOP` on `CC` | Absent (`FedPdSpecRevision = 0`) | Present (`PID 0x1012`, PD Rev 2) | Present (`VID 0x0639`, PD Rev 3) |
| Enumerated USB Device | None | `Sino Wealth USB Keyboard` | `Chrontel Billboard (CH7213)` |

With power plugged into the hub, Port 1's Type-C PHY (`AppleT8103TypeCPhy`) bound both `USB2` and `USB3` lanes to `AppleT8103USBXDCI` — the Mac's **USB Device controller** — and powered down `AppleT8103USBXHCI` (the **USB Host controller**):

```text
+-o AppleT8103TypeCPhy
    "AppleTypeCPhyLane" = {"Lane 1"={"Transport"="USB3","Power Level"="on","Client"="AppleT8103USBXDCI"},"Lane 0"={}}
    "AppleTypeCPhyUSB2" = {"Transport"="USB2","Power Level"="on","Client"="AppleT8103USBXDCI"}
```

Meanwhile `AppleSmartBattery` showed the Mac trickle-charging from a passive 5V Type-C resistor advertisement with no USB Power Delivery contract:

```text
"AdapterDetails" = {"AdapterVoltage"=5000,"Watts"=8,"Current"=1500,"PMUConfiguration"=1500,"UsbHvcMenu"=()}
"FedDetails" = ({"FedPortPowerRole"=0,"FedPdSpecRevision"=0,"FedExternalConnected"=1,"FedDualRolePower"=1}, ...)
```

## Why Powering the Hub Flipped the USB Data Role

In the USB Type-C specification, initial power role and initial data role are coupled on the Configuration Channel (`CC`) pins:

1. The port presenting a pull-up resistor (`Rp`) supplies `VBUS` (**Source**) and starts as the **Downstream Facing Port (`DFP` / USB Host)**.
2. The port presenting a pull-down resistor (`Rd`) sinks `VBUS` (**Sink**) and starts as the **Upstream Facing Port (`UFP` / USB Device)**.

For a USB-C hub to charge a laptop while letting the laptop act as USB Host for downstream peripherals, two things must happen over **USB Power Delivery (`USB-PD`)** on the `CC` wire:

1. The hub and charger negotiate a USB-PD contract (`SOP` messages).
2. The hub executes a USB-PD **Data Role Swap (`DR_Swap`)**, keeping the laptop as the power **Sink** while flipping its data role from `Device` (`UFP`) to `Host` (`DFP`).

Here, when external power was attached to the hub:

- **No USB-PD (`SOP`) messages were sent to the Mac** (`FedPdSpecRevision = 0`).
- Instead, the hub presented a passive `5V / 1.5A` (`7.5W`) Type-C `Rp` pull-up on `CC`.
- Following the Type-C state machine for a non-PD `Rp` source, `AppleTCControllerType10` configured the Mac as `Sink` + `DataRole = Device`, handed the PHY to `AppleT8103USBXDCI`, and turned off `AppleT8103USBXHCI`. A USB Device cannot enumerate downstream USB keyboards.

## Fix and Verification

Unplugging the power cable from the hub and reconnecting the hub to Port 1 immediately restored `DataRole = Host`:

- Without external 5V forcing `Rp` on `CC`, the Mac sourced 5V to the hub.
- The hub's internal PD controller (`Vendor ID = 0x0000`, `Product ID = 0x1012`, `bcdDevice = 0x0158`) booted and answered USB-PD Rev 2 `SOP` discovery, provisioning `USB2`, `USB3`, and 2-lane `DisplayPort` Alt Mode in `Host` mode.
- `AppleT8103USBXHCI` powered on (`CurrentPowerState = 3`) and enumerated the `Sino Wealth USB Keyboard` at `USB Address = 1`.

### Can You Connect Power to This Hub?

Look at the two failure modes that cause a hub's power port to present dumb `5V / 1.5A` `Rp` instead of USB-PD + `DR_Swap`:

1. **A non-PD 5V charger or USB-A-to-USB-C cable was plugged into the hub.** Pass-through hubs require a real **USB-C Power Delivery** brick (such as Apple's 30W/35W/67W USB-C PD adapter over a C-to-C cable) on their power input. Fed plain 5V without PD signaling, the hub's controller cannot negotiate a PD contract or issue `DR_Swap`, so it falls back to passive `5V / 1.5A` `Rp` and locks the Mac into `Device` mode.
2. **Plug-order glitch on budget PD controllers (`0x0000:0x1012`).** Clone PD controllers often fail `DR_Swap` if power is plugged into the hub *before* the hub is connected to the Mac. Connect the hub to the Mac first (so `DataRole = Host` and the keyboard comes up), then plug a real USB-C PD charger into the hub. If the keyboard drops again and `FedPdSpecRevision` returns to `0`, the hub's power port lacks working `DR_Swap` and cannot pass power and USB host data at the same time.

Because this adapter (`PID 0x1012`) also provisions 2-lane DisplayPort Alt Mode alongside USB 2.0/3.0, there is a cleaner workaround on a two-port MacBook Air: plug **both** the HDMI monitor and the USB keyboard into the hub on Port 1 (leaving the hub's power port empty), and plug the charger directly into Port 2.

## Diagnostic Commands

Run these to check whether a USB-C port is stuck in `Device` mode:

```bash
# Check USB host buses and enumerated devices
ioreg -p IOUSB -l -w 0

# Check Type-C port data role (Host vs Device), CC/SOP PD state, and Power In
ioreg -p IOPort -l -w 0

# Check which controller (AppleT8103USBXHCI Host vs AppleT8103USBXDCI Device) owns the PHY
ioreg -r -n "AppleT8103TypeCPhy" -l

# Check charger wattage and whether USB-PD (FedPdSpecRevision > 0) is active
ioreg -r -c AppleSmartBattery -l | grep -E '(AdapterDetails|FedDetails)'
```

## Source

- [USB Power Delivery Specification Overview](https://www.usb.org/usb-charger-pd)
- [USB Type-C Cable and Connector Specification](https://www.usb.org/usb-type-ctm-cable-and-connector-specification)
- [Apple USBDriverKit Documentation](https://developer.apple.com/documentation/usbdriverkit)
