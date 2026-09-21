# Debugging a dead MacBook Touch Bar down to the T2 USB bus

## The fix everyone posts cannot work. Here is what actually broke mine.

My Touch Bar went dark. Not frozen, not stuck showing the wrong app's controls, just black. No escape key, no brightness, no volume. Keyboard and trackpad were fine.

Search for this and every forum thread and every video gives you the same two lines:

```bash
sudo pkill TouchBarServer
sudo killall ControlStrip
```

It doesn't work. Not "it didn't work for me", but cannot work in the failure mode most people who search for this are actually in. Here is the full trace of what was broken, why that command is useless, and what brought the Touch Bar back.

System under test: MacBook Pro 16-inch 2019 (`MacBookPro16,1`), 6-core Intel i7, Apple T2 Security Chip firmware `23P3120`, macOS 15.7.4 (`24G517`).

---

## Step 1: the famous fix fails, and the failure is the first clue

```
$ sudo killall ControlStrip
No matching processes were found
```

`killall` didn't fail to fix the Touch Bar. It failed to find anything to kill.

```bash
pgrep -lf TouchBarServer   # nothing
pgrep -lf ControlStrip     # nothing
```

Neither process was running. The restart trick assumes the Touch Bar daemons are alive but wedged: kill them, launchd respawns them, done. That's a real failure mode and the trick genuinely fixes it. But if the processes were never running, there is nothing to restart, and you will keep pasting that command into your terminal forever.

So before you restart a service, check that it is running. A lot of the fixes on the internet are for a different bug than the one you have.

---

## Step 2: launchd knows about them, they just won't stay up

```
$ launchctl list | grep -i -E "touchbar|controlstrip"
-	0	com.apple.controlstrip
-	0	com.apple.SpacesTouchBarAgent.app
```

Three columns: PID, last exit status, label. PID is `-`, so nothing is running. Exit status is `0`, so whatever ran last exited cleanly.

That combination is odd. A crash gives you a non-zero status or a signal. Exit `0` means the process started, decided it had nothing to do, and quit on purpose.

The services are registered. They launch. They immediately and deliberately give up.

---

## Step 3: read the plists, find the dependency

The launchd definitions live here:

```
/System/Library/LaunchDaemons/com.apple.touchbarserver.plist
/System/Library/LaunchAgents/com.apple.controlstrip.plist
/System/Library/LaunchAgents/com.apple.SpacesTouchBarAgent.plist
```

Two useful things fall out of reading them. First, the real binary paths:

```
/usr/libexec/TouchBarServer
/System/Library/CoreServices/ControlStrip.app/Contents/MacOS/ControlStrip
```

Second, the architecture. `ControlStrip.plist` has no `RunAtLoad`. It has a `LaunchEvents` block keyed on `com.apple.touchbar.matching`. ControlStrip does not start on its own. It waits for TouchBarServer to publish that Mach matching event. TouchBarServer registers four Mach services:

```
com.apple.touchbarserver
com.apple.touchbarserver.mig
com.apple.touchbarserver.plugin
com.apple.touchbarserver.render
```

ControlStrip is also fenced by `LimitLoadToSessionType` to `Aqua` and `LoginWindow`, so it only exists inside a logged-in GUI session.

The chain is strictly ordered. TouchBarServer publishes the event, ControlStrip wakes on it, ControlStrip draws the bar. Break any link and every link after it fails quietly. Restarting ControlStrip on its own can never fix this, which is the other half of why the popular fix is useless.

### False lead 1

I checked `/System/Library/PrivateFrameworks/TouchBarServer.framework/` here and found nothing. For about five minutes I was sure the system volume was damaged and the binary had been deleted, and I started sketching a macOS reinstall.

Wrong. `TouchBarServer` was never in PrivateFrameworks. It's at `/usr/libexec/TouchBarServer`, exactly where the plist I had just read said it was. I had guessed a path from memory instead of using the one in front of me.

An absence you didn't verify is not evidence. "I looked and it wasn't there" only means something if you looked in the right place.

---

## Step 4: does the hardware still exist?

Down a layer. Ask IOKit whether the machine still has a Touch Bar.

```bash
ioreg -l | grep -i -c -E "AppleDFR|DFRDisplay"   # 0
ioreg -l | grep -i -c "AppleMultitouchDevice"    # 5
```

DFR is Apple's internal name for the Touch Bar: Dynamic Function Row. Zero DFR entries, five multitouch entries.

### False lead 2

Those five matches nearly sent me off in the wrong direction. They're the trackpad. `AppleMultitouchDevice` is the trackpad digitizer and has nothing to do with the Touch Bar. A grep that matches five things tells you nothing if it matched five of the wrong thing. Look at what actually matched, not how many.

So, zero DFR devices. A useful signal but not a conclusive one, because the Touch Bar doesn't register under a class named after itself.

---

## Step 5: the USB bus, where the answer actually was

On a T2 Mac the Touch Bar is not some exotic private bus. It's USB, hanging off the T2 controller.

```bash
system_profiler SPUSBDataType
```

Under Apple T2 Bus:

| Device | Product ID |
|---|---|
| Apple T2 Controller | `0x8233` |
| Composite Device | `0x8104` |
| Touch Bar Backlight | `0x8102` |
| Apple Internal Keyboard / Trackpad | `0x0340` |
| Headset | `0x8103` |
| Ambient Light Sensor | `0x8262` |
| FaceTime HD Camera | `0x8514` |

Everything is there. Camera, mic, sensors, keyboard. And Touch Bar Backlight is present, which means the panel is physically attached and its backlight circuit is powered.

Now look at what is missing. There is a second Touch Bar device that belongs in that list: Touch Bar Display, product ID `0x8302`. The backlight and the display are two separate USB devices. Mine had exactly one of them.

Cross-checked at the HID layer:

```bash
ioreg -c IOHIDDevice -r -l | grep -E "Product|VendorID|ProductID"
```

`Touch Bar Backlight` (vendor `1452` for Apple, product `33026` which is `0x8102`) shows up twice, under HID usage pages `0xFF12` and `0xFF00`. That's why the DFR grep came back empty: the Touch Bar registers as a generic `IOHIDDevice`, not under any class with "DFR" in the name. My earlier query wasn't proving absence, it was asking the wrong question.

And still no display controller. Backlight enumerated, display not.

### False lead 3, the expensive one

I went to the boot logs next. On this machine an `rtk` shell hook shadows the system `log` binary, so `log show --predicate ...` dies with `(eval):log:1: too many arguments`. Use the absolute path:

```bash
/usr/bin/log show --last 30m --predicate 'process == "WindowServer"' --style compact
```

The boot log looked healthy:

```
WindowServer  loaded /System/Library/HIDPlugins/IOHIDDFREventFilter.plugin
WindowServer  loaded /System/Library/HIDPlugins/DFRTouchServiceFilter.plugin
com.apple.CoreBrightness.AppleUSBALS  found _hidDFRDisplay.displayStateElement page=0xFF12 usage=0x31
setting DFR state = 2
```

DFR plugins loading. CoreBrightness binding a DFR display element. "setting DFR state = 2". I concluded the hardware was fine and this was purely a software problem, and went off to restart daemons.

That was wrong, and it cost more time than any other mistake in this debug. Those HID plugins load on any Touch Bar-capable model whether or not a panel answers. `setting DFR state = 2` is WindowServer announcing what it intends for the DFR, not confirmation that a DFR received it. The log recorded an attempt and I read it as a result. A log line proves the code ran, not that it worked.

---

## Step 6: force the daemon up and make it say what's wrong

The system-domain kickstart needs a real TTY for the sudo prompt, so run this in your own terminal:

```bash
sudo launchctl kickstart -k system/com.apple.touchbarserver
```

TouchBarServer came up and stayed up, PID 11138. Touch Bar still dark. But ControlStrip's exit status changed from `0` to `1`. It had stopped politely declining to start and had begun genuinely failing, which meant it was finally getting far enough to produce a real error.

```bash
/usr/bin/log show --last 5m --predicate 'process == "TouchBarServer"' --style compact
```

There it is:

```
[com.apple.touchbarserver:hmd] SLSGetTouchBar returned nil
[com.apple.CoreBrightness:default] Failed to create log handle
```

`SLSGetTouchBar` is a SkyLight private API, SkyLight being the framework behind WindowServer. TouchBarServer calls it to get a handle to the Touch Bar display, and got back `nil`. WindowServer has no Touch Bar to hand out.

ControlStrip's own log:

```
Channel could not return listener port
TouchBarServer not running
```

That message is false, and it is the reason this bug is so badly misdiagnosed online. TouchBarServer was running. I could see the PID. ControlStrip doesn't check for a live process. It checks for a registered Touch Bar object, and prints "TouchBarServer not running" when it finds none.

Users read "not running", conclude the process is dead, and go restart a process that was never the problem. That is where `pkill TouchBarServer` comes from.

---

## The actual root cause

Reading the chain from the bottom up:

1. Touch Bar Display (`0x8302`) fails to enumerate on the T2 USB bus. The backlight (`0x8102`) enumerates fine, which is why the hardware looks present at a glance.
2. No USB device means no HID device, so nothing binds a driver.
3. WindowServer and SkyLight have no DFR display registered.
4. `SLSGetTouchBar` returns `nil`. TouchBarServer runs in a degraded no-op state.
5. TouchBarServer never publishes `com.apple.touchbar.matching`.
6. ControlStrip finds no registered Touch Bar, prints the misleading "TouchBarServer not running", and exits.

The break is at step 1, in T2 USB enumeration. Steps 2 through 6 are consequences. Every software-layer fix I found online operates at steps 5 and 6, which is about as useful as rebooting your monitor's on-screen menu to fix an unplugged cable.

---

## The fix: SMC reset

The T2 handles power and peripheral enumeration for the internal bus. If it comes up without enumerating the Touch Bar display, a normal reboot won't help, because the T2 holds its state across a warm restart. You have to reset the SMC.

For a T2 Mac with the power button in the Touch ID key:

1. Shut down fully. Apple menu, then Shut Down. Not Restart.
2. Hold left Control, left Option and right Shift for 7 seconds.
3. Keeping those three held, also hold the power button for 7 more seconds.
4. Release all four. Wait 10 seconds. Power on.

The exact keys matter: left Control, left Option, right Shift. Wrong side of the keyboard, no reset.

Apple Silicon Macs have no user-triggerable SMC reset; a full shutdown and 30 seconds unplugged is the closest equivalent. This bug is specific to the T2 generation anyway.

---

## Verification

Don't trust your eyes here, since the backlight can come on without the display working. Check the bus:

```bash
system_profiler SPUSBDataType | grep -A1 "Touch Bar"
```

Fixed:

```
        Touch Bar Backlight:
          Product ID: 0x8102
--
        Touch Bar Display:
          Product ID: 0x8302
```

Not fixed: only `Touch Bar Backlight` appears, so the display still isn't enumerating. Go to the escalation list below.

Then check that the software stack followed the hardware back up:

```bash
pgrep -l TouchBarServer   # 359 TouchBarServer
pgrep -l ControlStrip     # 453 ControlStrip
```

Both running, both started automatically at boot with no kickstart. That's the whole chain healthy.

One piece of noise worth naming, because it looks alarming and isn't:

```
system_profiler[7000:25202] SPUSBDevice: IOCreatePlugInInterfaceForService failed 0xe00002be
```

`0xe00002be` is `kIOReturnNoResources` in `IOReturn.h`. `system_profiler` prints it for a handful of USB devices it can't open a plugin interface on, every run, on a completely healthy machine. Nothing to do with the Touch Bar.

---

## If the SMC reset doesn't do it

In order, cheapest first.

NVRAM reset: power on holding Option, Command, P and R for about 20 seconds, then re-check the USB bus.

Firmware revive with Apple Configurator: needs a second Mac and a USB-C cable in the correct port. Revive preserves your data, Restore erases the machine, so pick carefully. Reviving rewrites bridgeOS, the firmware that owns this enumeration.

Hardware: if the display still won't enumerate after a firmware revive, the panel's connector or the panel itself has failed. The backlight and display are separate circuits on the same flex cable, and a partial cable failure produces exactly this signature, backlight yes and display no.

Reinstalling macOS isn't on that list. Nothing in this failure lives on the system volume.

---

## What I'd take from this

Most of my wasted time came from trusting reports instead of checking state. I believed a `killall` that had nothing to kill, a missing framework at a path that never held it, and a log line that announced an intention rather than a result. Each one felt like evidence and none of it was.

The other half is direction. Process, launchd, plist, IOKit, HID, USB. The answer was at the bottom of that stack and every fix circulating online lives at the top, which is why none of them work. If a service won't stay up, the interesting question is usually what it's waiting for, and that question keeps pointing downward until you hit hardware.

Underneath all of it: one USB device that didn't show up, and four layers of software reporting something else about it.
