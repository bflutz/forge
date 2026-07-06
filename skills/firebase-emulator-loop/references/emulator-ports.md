# Workspace Firebase Emulator Port Map

Each project gets a 100-port band so two emulators can run side-by-side without collision.

Default Firebase emulator ports (collision-prone): UI 4000, Functions 5001, Firestore 8080, Pub/Sub 8085, Database 9000, Auth 9099, Storage 9199, Hosting 5000, Eventarc 9299, Tasks 9499.

We offset every project off these defaults using a per-project band, and we use `singleProjectMode: true` to isolate state.

## Allocation table

| Project              | Path                                              | Band   | UI   | Auth | Firestore | Storage | Functions |
|----------------------|---------------------------------------------------|--------|------|------|-----------|---------|-----------|
| Atlas                | `<workspace>/Projects/Atlas`                      | 9000   | 9000 | 9099 | 9080      | 9199    | 9005      |
| Acme Dashboard       | `<workspace>/AcmeDashboard`                       | 9100   | 9100 | 9199 | 9180      | 9299    | 9105      |
| Acme.Demo.Dashboard  | `<workspace>/Projects/Acme.Demo.Dashboard`        | 9200   | 9200 | 9299 | 9280      | 9399    | 9205      |
| Firebase.Camper.Van  | `<workspace>/Projects/Firebase.Camper.Van`        | 9300   | 9300 | 9399 | 9380      | 9499    | 9305      |
| Proteus              | `<workspace>/Projects/Proteus`                    | 9400   | 9400 | 9499 | 9480      | 9599    | 9405      |

## Adding a new project

1. Pick the next free band (current next: 9500).
2. Add a row to this table.
3. Use the standard offsets: UI = band+0, Functions = band+5, Firestore = band+80, Auth = band+99, Storage = band+199.

## Why these specific offsets

- Firestore stays close to 8080 default (just shifted) — easy to recognise.
- Auth at +99 mirrors the upstream default's hundreds-digit pattern (9099).
- Storage at +199 mirrors 9199 default.
- UI at +0 makes the band number itself the obvious web entry.
- Functions at +5 mirrors 5001 default's ones-digit.

Anyone reading a `firebase.json` can infer the project from the band and the service from the offset.
