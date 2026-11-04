# Kenter case: 3D print files

Kenter's version of the Shell case. The original parts are the Shell serial-run CAD in
`hardware/shell/serial_run/MECHANICAL/3Ddrawings/` (STEP, plus 2D drawings as PDF one level up).
That folder is excluded by this checkout's sparse-checkout; read it with
`git show HEAD:<path>` or `git sparse-checkout add hardware`.

## Layout

| Folder | Contents |
|---|---|
| `source/` | Blender sources (`kenter-case.blend`, `kenter-wallet.blend`) and the glTF exports of the edited parts (`*-KNTR.glb`) |
| `print/` | STL files to send to the print shop: millimetres, Z up, same coordinates as the original CAD |

`*.blend1` (Blender's automatic backup) is ignored by git.

## Parts

| STL | Part | Origin | Print in |
|---|---|---|---|
| `KNTR-001-TOP.stl` | Top panel (screen side) | Edited from `2282-001-TOP-V06` | SLA resin (or MJF nylon with extra key clearance) |
| `2282-002-MIDDLEFRAME-V09.stl` | Middle frame | Unchanged | SLA resin / MJF nylon |
| `2282-003-SIDEBUTTON-V06.stl` | Side button (Cancel) | Unchanged | SLA resin |
| `2282-004-BOTTOMCOVER-V04.stl` | Back cover | Unchanged | SLA resin / MJF nylon |
| `KNTR-005-KEYPAD.stl` | Keypad sleeve, one piece | Edited from `2282-005-KEYBOARDSLEEVE-V07` | Elastic resin ~50A or TPU for fit tests; cast silicone (Shore 40-60A) for the real part |

Not printed: the windows `006`/`007` (clear acrylic or glass, laser cut) and the adhesives, foams
and protection films `008`-`013` (die-cut sheet). Use the original STEP files for those.

## Key grid

Blender/STL coordinates in mm, origin at the screen centre, +Y towards the screen.

- Key centres: X in {-18, 0, +18}, Y in {-28, -41, -54, -67}; the reference key "5" is at (0, -41).
- Legends, matching `app/keypad/keypad.h`, with the top row nearest the screen:

  | | | |
  |---|---|---|
  | 1 | 2 | 3 |
  | 4 | 5 | 6 |
  | 7 | 8 | 9 |
  | Back | 0 | Confirm |

- Panel top surface Z = 5.4. Original keys: 10.0 mm square, top at Z = 6.2, with about 0.25 mm gap per side to the opening.
- Plunger tip: bottom at Z = 0.4. The air pocket around it (about 3 mm wide, up to Z = 1.1 in the original) is what lets the key travel.

## Status (second check, 2026-11-04)

Draft: **not ready to order.**

- [x] All 12 keys and openings centred on the grid (within 0.01 mm).
- [x] Legends recessed 0.36 mm; arrows on 2/4/6/8 raised 0.31 mm (top Z = 6.44).
- [x] Square plungers clear the switch bodies; only the tip touches the actuator.
- [ ] Key-to-opening gap: 0.11 mm per side (keys 10.42, openings 10.64). Aim for 0.2-0.25 mm (openings 10.8-10.9).
- [ ] The "5" nub is recessed (pit 0.36 mm deep, Ø ≈ 2.8 mm), not raised. Union it instead of subtracting it if it should stand proud.
- [ ] Space under the keys: the key body's bottom edge is at Z ≈ 0.36 (originally ≈ 0.72) above the board. Check it against the switch travel.
- [ ] Plunger tip at Z = 0.38 against the switch actuator top at 0.42: 0.04 mm preload (originally 0.02). Raising it to about Z = 0.42 is safer.
- [ ] Mesh: the top panel has about 200 open edges and the keypad about 30 non-manifold edges near Z ≈ 1.1-1.3. Run 3D Print Toolbox → Make Manifold on both.

## Exporting STL from Blender

File → Export → STL, **Scale 1000** (the scene is in metres), Selection Only, Apply Modifiers.
Then run the 3D Print Toolbox check (Check All) before sending.
