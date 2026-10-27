# Downloadable app ideas

Status, 2026-10-27: not planned. 0SFS is a web app, and nobody should have to
download anything to fly it. This page collects what only a downloadable app
could do, in case one is made. Nothing here is worth a download on its own.

## Turn and zoom at once on a Mac trackpad

**What it gives.** Two fingers on a Mac trackpad would turn the view and zoom
it in one gesture, as they do on a touchscreen and on the phone's camera pad.
In a browser they cannot: macOS decides whether a gesture is a scroll or a
pinch before the page sees it, and keeps that one kind until the fingers lift
([Camera gestures](https://github.com/0SFS/controls/blob/main/docs/camera-gestures.md)).

**How.** A native process can read the raw contacts before macOS classifies
them, through the private `MultitouchSupport.framework`. The app would pass
them to the flight camera's touch path (`controls/src/flightCameraInput.ts`
and `createTouchSurfaceGlide` in `controls/src/cameraGlide.ts`) as fingers, the
same way as a touchscreen's.

**What a probe found**, on macOS 27, 2026-10-27, from a ~50-line Swift program
that loads the framework with `dlopen`:

- `MTDeviceCreateList` lists the devices. `MTRegisterContactFrameCallback`
  and `MTDeviceStart(device, 0)` deliver one frame per callback: the contacts,
  their count, a timestamp and a frame number. `MTDeviceStop` ends it.
- Frames come at 125 Hz, and macOS asked for no permission.
- One contact is 96 bytes:

  | Offset | Type | Meaning |
  | --- | --- | --- |
  | 16 | Int32 | finger id, kept while the finger stays down |
  | 20 | Int32 | state: 1 start, 3 begin, 4 touching, 5 breaking, 7 left |
  | 32, 36 | Float | position on the surface, 0 to 1, y up |
  | 48 | Float | contact size |
  | 68, 72 | Float | position in millimetres |

- The trackpad's sensor reported 12194 × 7408 in hundredths of a millimetre.

**Costs.** The framework is private: Apple documents none of it, may change it
in any macOS release, and does not allow it in the Mac App Store. The browser
still receives the same gesture as a scroll or a pinch, so the app would have
to stop the page from acting on those while it feeds the contacts itself. It
covers Macs only; Windows and Linux touchpads would each need their own way in.
