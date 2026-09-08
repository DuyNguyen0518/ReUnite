# ReUnite Mobile — the Flutter app

The app is a **shell over the Rust mesh core**. Routing, encryption, peer ranking, SOS,
panic codes, ghosting and zone consensus all happen in `crates/meshcore`; this layer
starts that core, pushes commands into it, drains its events, and draws the result.

**Nothing here may grow protocol logic.** Anything that looks like a rule about the mesh
belongs in Rust, where the terminal client gets it for free too.

Full setup — laptop, Android, iPhone, and the two manual Xcode steps iOS needs — is in
[`../docs/MOBILE.md`](../docs/MOBILE.md).

---

## Structure

```text
mobile/
├── pubspec.yaml
└── lib/
    ├── main.dart                    entry point; picks the radios for this platform
    ├── app.dart                     startup gate, SOS banner, bottom navigation
    ├── bridge/
    │   ├── mesh_ffi.dart            raw dart:ffi bindings to libmeshffi (JSON in/out)
    │   └── ble_radio.dart           MethodChannel/EventChannel to the native radio
    ├── services/
    │   └── mesh_service.dart        the app's single connection to the core
    ├── models/
    │   └── mesh_models.dart         Dart mirrors of crates/meshffi/src/dto.rs
    ├── features/
    │   ├── chat/                    network chat, hops and delivery status
    │   ├── map/                     peer radar (default) + interactive OSM map
    │   ├── emergency/               slide-to-SOS, panic codes, zone reporting, heatmap
    │   └── networks/                private networks, invites, kick, radio diagnostics
    └── shared/
        └── theme.dart               dark emergency-mode theme
```

Native Bluetooth lives outside `lib/`, because mobile operating systems will not let a
Rust library own their Bluetooth stack:

| Platform | File | Roles |
| :--- | :--- | :--- |
| Android | `android/.../BleMesh.kt`, `FrameCodec.kt` | peripheral **and** central |
| iOS | `ios/Runner/BleMesh.swift` | peripheral **and** central |
| macOS | `macos/Runner/BleMeshCentral.swift` | central only — a Mac cannot advertise |

All three share the GATT service UUID `a1b2c3d4-e5f6-7890-1234-56789abcdef0` with
`crates/meshcore/src/transport/ble_linux.rs`, so a Linux laptop running
`meshnet --transport ble` is just another peer.

---

## Run it

The app will not start without the native core built for the platform you are running
on — it shows a startup-error screen naming every path it tried.

```bash
# from the repository root
./scripts/build_ffi.sh macos      # or: android | ios

cd mobile
flutter pub get
flutter run -d macos              # or: flutter run -d <device>
```

Seed peers can be baked in at build time, which is the only route available before the
app has ever run on a device:

```bash
flutter run --dart-define=MESH_PEERS=10.17.158.195:47474
```

Addresses typed into the Radio panel on the Networks tab are saved on the device and
merged in behind these.

---

## Test it

```bash
../scripts/check.sh dart          # analyze + test, building the core if it is missing
```

The Dart tests are not mocked: `app_test.dart` and `peers_test.dart` load the same
`libmeshffi` the shipped app loads and drive a **real Rust node** over FFI. That is the
check that the UI is wired to the core rather than to a stub, which is what Phase 2 step
2.1 was about. They are `@TestOn('mac-os')` and need `./scripts/build_ffi.sh macos` first.

`widget_test.dart` is pure unit tests over the model and configuration layer, and runs
anywhere.
