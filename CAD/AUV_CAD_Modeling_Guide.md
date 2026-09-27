# AUV Top-Down CAD Build Guide

**For:** Nate Hasty — CAD lead, WPI AUV MQP 2026–2027
**Written:** 2026-09-22
**Purpose:** a click-level modeling guide for the 880 mm torpedo AUV brief, so you can build it
yourself and compare against the ChatGPT Astra run.

> Everything numeric in this guide is either (a) copied from the brief, (b) arithmetic on brief
> values with the arithmetic shown, or (c) explicitly labelled **ASSUMED**. Nothing is quietly
> invented. If a number here has no source tag, it came from the brief.

---

## Part 0 — "How will you drive SolidWorks?"

Answering this the way I'd answer it if I were executing the brief, so you have a baseline to
judge Astra's answer against.

**The honest answer for any AI agent:** I am running in a Linux cloud container with no Windows
and no SolidWorks licence, so I cannot touch SolidWorks from here. Two routes exist on *your*
laptop: the **SolidWorks API** (`ISldWorks` / `IModelDoc2` / `IFeatureManager`) driven from a VBA
macro or Python via `pywin32` COM, which is the only sane programmatic route; or **screen-level
UI automation** (screenshot, click, type), which I technically have access to on `nates-laptop`
but which is far too slow and brittle for a 30-part parametric assembly — one mis-registered click
and you get silent geometry corruption rather than an error.

**What no agent can execute, regardless of route:**

- Retrieving vendor CAD that sits behind a login or a JS-gated download button.
- Verifying the mass of a part the team has not bought and weighed.
- Any structural validation beyond closed-form hand calcs (no FEA judgement).
- Deciding whether a joint is manufacturable in the WPI machine shop.
- Resolving the four spec conflicts listed in Part 3 — those need a human decision.

**Use this as your rubric on Astra's output.** An agent that claimed it could do the whole brief
without naming a single limitation was not being careful. There's a full comparison checklist at
the end of this document (Part 12).

---

## Part 1 — Before you open SolidWorks

### 1.1 What you already have

Last year's team left usable vendor geometry in your repo:

```
CAD/V2 SolidWorks Files/Newest 4 Thruster Design/
  BROV 4 inch dome.SLDPRT
  BROV 4 inch enclosure tube 12 inch length.SLDPRT
  BROV 4 inch end cap.SLDPRT
  BROV 4 inch flange.SLDPRT
  BROV 12 inch by 4 inch enclosure.SLDASM
  T100-ASM-CW-THRUSTER-R1.SLDPRT        <- T100, NOT T200. Do not reuse.
```

**Do this first:** open each of those four BR parts, run `Tools > Evaluate > Measure`, and write
the actual dimensions into your assumptions log. Do **not** assume they are correct — verify the
tube OD reads 112.000 and the ID reads 103.000. If they don't, they were modelled by a student and
you re-download from Blue Robotics. The T100 file is the wrong thruster; you need a T200.

### 1.2 Vendor CAD to fetch

| Part | Where | Note |
|---|---|---|
| 4" Series Dome End Cap | bluerobotics.com product page → Technical Details → CAD | **STEP, not SLDPRT** (see 1.3) |
| 4" Series End Cap (blank + hole patterns) | same | get the pattern you'll actually use |
| 4" Series Flange / O-ring flange | same | |
| T200 Thruster | bluerobotics.com or GrabCAD | |
| Bar02 / Bar30 depth sensor | bluerobotics.com | tiny, envelope is fine |
| WetLink Penetrator (M10) | bluerobotics.com | |

### 1.3 The SLDPRT version trap

SolidWorks **cannot open a part saved by a newer version of SolidWorks.** If BR publishes SLDPRT
from SW 2025 and WPI's lab is on 2023, the file simply refuses. **Default to STEP for everything
imported.** A STEP import is a dumb solid with no features — which is exactly what you want for a
COTS part you must not modify anyway.

After importing a STEP: `Right-click the part > Material > Edit Material` to assign 6061-T6 (or
whatever it is), or use the mass override in 1.6.

### 1.4 Folder structure

Build this under `CAD/2026-27_AUV/` in the repo:

```
CAD/2026-27_AUV/
  00_Library/
      AUV_Materials.sldmat          custom materials (syntactic foam, water)
      Vendor/                       imported STEP/SLDPRT, untouched originals
      Templates/                    AUV_Part.prtdot, AUV_Asm.asmdot, AUV_Drw.drwdot
  AUV_GLOBALS.txt                   the shared equation file
  AUV-100-SKELETON.SLDPRT
  AUV-2xx_Camera/
  AUV-3xx_Electronics/
  AUV-4xx_Ballast/
  AUV-5xx_Tail/
  AUV-6xx_BlankSpacer/
  AUV-900-REF-VOLUMES.SLDPRT
  AUV-000-TOP.SLDASM
  Drawings/
  Exports/
  Docs/    ICD.md, assumptions_log.md, mass_buoyancy_report.md
```

### 1.5 Git LFS — you have a gap

Your `.gitattributes` currently tracks `*.stl`, `*.SLDPRT`, `*.SLDASM`. Git attribute patterns are
**case-sensitive**, and SolidWorks writes lowercase extensions in some save paths. You are also
missing drawings and STEP files. Fix it before you commit any CAD:

```gitattributes
*.stl  filter=lfs diff=lfs merge=lfs -text
*.STL  filter=lfs diff=lfs merge=lfs -text
*.sldprt filter=lfs diff=lfs merge=lfs -text
*.SLDPRT filter=lfs diff=lfs merge=lfs -text
*.sldasm filter=lfs diff=lfs merge=lfs -text
*.SLDASM filter=lfs diff=lfs merge=lfs -text
*.slddrw filter=lfs diff=lfs merge=lfs -text
*.SLDDRW filter=lfs diff=lfs merge=lfs -text
*.step filter=lfs diff=lfs merge=lfs -text
*.STEP filter=lfs diff=lfs merge=lfs -text
*.stp  filter=lfs diff=lfs merge=lfs -text
*.STP  filter=lfs diff=lfs merge=lfs -text
```

Also add `~$*` (already there) and `*.bak`, `*.SLDPRT.bak` to `.gitignore`.

**Warning:** SolidWorks assemblies store *paths* to their components. If two people clone the repo
to different absolute paths, references still resolve because they're relative within the folder —
but only if everyone keeps the whole `2026-27_AUV/` tree together. Never commit a part outside it.

### 1.6 Document setup (do this once, then save as a template)

1. New part → `Tools > Options > Document Properties`
2. `Units` → **MMGS**, and set **Decimals = 3** for *Length*, *Angle* (2 is fine), *Mass/Section
   Properties*. The brief requires three decimals; SolidWorks defaults to two.
3. `Drafting Standard` → ANSI or ISO (pick one, tell the team, never change it)
4. `Image Quality` → nudge shaded resolution up one notch; imported STEP looks faceted otherwise
5. `File > Save As` → type **Part Templates (*.prtdot)** → `00_Library/Templates/AUV_Part.prtdot`
6. Repeat for an assembly template and a drawing template
7. `Tools > Options > System Options > File Locations > Document Templates` → Add the Templates
   folder

**Custom materials.** `Right-click Material > Edit Material`. Right-click the SolidWorks Materials
tree → you cannot edit it. Instead: right-click any close material → **Copy**; right-click in the
lower pane → **New Library**, save as `00_Library/AUV_Materials.sldmat`; paste into it. Create:

| Material | Density | Source |
|---|---|---|
| `Acrylic (PMMA)` | 1180 kg/m³ | **ASSUMED** — generic PMMA. SW's built-in "Acrylic (Medium-high impact)" is 1200. |
| `Al 6061-T6` | 2700 kg/m³ | SW built-in, use as-is |
| `Delrin / POM` | 1410 kg/m³ | SW built-in "POM" |
| `Syntactic Foam (ASSUMED)` | 500 kg/m³ | stated in the brief |
| `WATER_FRESH_1000` | 1000 kg/m³ | stated in the brief |

Then `Tools > Options > System Options > File Locations > Material Databases` → Add
`00_Library/`.

**Mass override for COTS and envelope parts.** Open the part →
`Tools > Evaluate > Mass Properties > Override Mass Properties...` → tick *Mass* → enter the
value. For the T200: **427.000 g** (brief, in air). Every override goes in the assumptions log.

**Custom properties for traceability.** In every part: `File > Properties > Custom` tab, add:

- `SOURCE` = `Vendor CAD` / `Envelope` / `Designed`
- `MASS_SOURCE` = `Datasheet` / `Material` / `ASSUMED`
- `REF` = the datasheet URL or log entry ID

These show up in the BOM, which lets you mechanically prove the assumptions log is complete
(acceptance check 7).

---

## Part 2 — The two decisions that will wreck the model if you get them wrong

### 2.1 Coordinate system mapping

The brief says: **origin at nose tip, +X aft, +Y starboard, +Z up.** That is a right-handed frame
(check: X×Y = (−fwd)×(stbd) = up ✓).

SolidWorks' default planes are: **Front = XY**, **Top = XZ**, **Right = YZ**.

So if you set the hull axis along model X and "up" along model Z:

| What you're drawing | Sketch plane | Why |
|---|---|---|
| **Side elevation** (nose→tail, up/down) — the master layout | **Top plane** (XZ) | Its axes are X and Z |
| **Plan view** (nose→tail, port/starboard) — thruster offsets, dive plane span | **Front plane** (XY) | Its axes are X and Y |
| **Hull cross-section** — the 112 circle, vent/flood holes | **Right plane** (YZ) | |

**The consequence:** pressing "Front" on the view toolbar shows you the *plan* view, and "Top"
shows you the *side elevation*. It feels wrong for the first hour and then you stop noticing.

**Do it anyway.** The alternative — modelling in SolidWorks' natural orientation with Y up — means
the Mass Properties dialog reports your CG as `(x, y, z)` where `y` is the vertical, and every
buoyancy number in your report has to be mentally transposed. That is exactly how a BG separation
sign error gets into a design review.

**Belt and braces:** add `Insert > Reference Geometry > Coordinate System` at the origin, name it
`CSYS_VEHICLE`, X along the hull axis pointing aft, Z up. In the Mass Properties dialog there is an
**"Output coordinate system"** dropdown — select `CSYS_VEHICLE` for every mass report. Now the
numbers are stated in the brief's frame no matter what.

**Make the views bearable:** orient the model to the side elevation, then
`View > Modify > Orientation` (or spacebar) → click the **New View** icon → name it
`SIDE_ELEVATION`. Do the same for `PLAN` and `BOW`. These named views appear in the drawing View
Palette later, which is how you guarantee your drawing sheet shows the right thing.

### 2.2 Origin scheme — this is the top-down decision

Two workable schemes. Pick one *now*; switching later means remodelling.

**Option A — Common origin / master-model (RECOMMENDED for this build).**
Every section part's origin is the **vehicle nose tip**. The tail fairing sits 620–880 mm from its
own origin inside its own part file. Every part inserts the skeleton origin-to-origin. Every
assembly mate is origin-to-origin: three coincident mates (Front↔Front, Top↔Top, Right↔Right) and
the component is fully defined.

- *Pro:* dead simple, impossible to mis-position, the blank-spacer swap is trivially identical,
  and changing `L_ballast` moves the tail geometry *inside the tail part* so the assembly never
  has to do anything clever.
- *Con:* parts aren't reusable standalone; a part-level drawing has a datum 700 mm away.
- This is a standard aero/auto skeleton technique. It is defensible at a review.

**Option B — Local origins + station-plane mates.**
Each section part's origin is its own forward interface face. In the assembly you mate the part's
Right plane to the skeleton's `PLANE_STn`, plus Top↔Top and Front↔Front.

- *Pro:* parts are reusable and self-contained; the "change a length, watch it move" demo is
  visible in the assembly, which reviewers like.
- *Con:* any part that needs the outer mould line (the tail fairing taper) must insert the skeleton
  with a `Move/Copy` translate of `-1 * ("L_nose"+"L_elec"+"L_ballast")`, which is one more thing
  to break.

**Go with A.** You are rusty and on a schedule; A has fewer failure modes. If a reviewer asks,
say you used a common-origin master model and that station positions are carried by the equation
set rather than by mate offsets.

Either way the rule from the brief holds: **only `AUV-100-SKELETON` is fixed. Everything else is
mated. Nothing is dragged into place.**

---

## Part 3 — Six things in the brief that do not close

The brief says "if you cannot obtain a dimension you need, stop and ask." These are the places
where you should stop. Raise all six with your advisor and the team in one go so you aren't blocked
twice.

### 3.1 The axial stack-up does not close at 420 mm

Station 1→2 is 130→420, i.e. 290 mm of "electronics section", and the brief specifies a **300 mm**
tube. But stations 0–420 are **one continuous dry volume**, so the real constraint is:

```
L_dome_protrusion  +  300 (tube)  +  L_aftcap_protrusion  =  420
```

which leaves **120.000 mm** for the two end caps' protrusion beyond the tube ends. A 4" dome end
cap protrudes roughly a hemisphere's worth (≈ 56 mm radius → order 60–80 mm) and a flat aluminium
end cap protrudes maybe 10–20 mm. That's ~70–100 mm, not 120.

**Action:** import the dome cap and end cap STEP, `Measure` the axial protrusion of each past the
tube end face, and write:

```
X_tube_fwd  = L_dome_protrusion                (derived, not free)
X_tube_aft  = X_tube_fwd + 300
L_dry_total = X_tube_aft + L_aftcap_protrusion  (derived)
```

Then compare `L_dry_total` to 420. **Expect 420 to move — probably down, to somewhere near
390–400.** Tube stock is 200/300/400 only, so the tube length is the hard number and 420 is the
soft one. Do not force 420 by adding a spacer.

*This is the single best test of whether Astra actually reasoned about the geometry or just typed
the brief's numbers into sketches.*

### 3.2 An 80 mm thruster offset does not physically fit

T200 body diameter is 100 mm → radius 50 mm. At `Y_thruster = 80`:

- Clearance to the tail fairing requires `D_fairing_at_thruster ≤ 2 × (80 − 50) = 60.000 mm`
- Prop tip clearance (76 mm prop → 38 mm radius) requires `D_fairing ≤ 2 × (80 − 38) = 84.000 mm`

The **body**, not the prop, is binding: the fairing can be no more than **60 mm diameter** at the
thruster station, and realistically ≤ 50 mm once you allow 5 mm clearance a side. That is a very
thin tail to route a structural spine, two servo harnesses and six 16 AWG motor leads through.

Put it in the skeleton as a hard equation and let SolidWorks enforce it:

```
"Y_thruster"  = 80
"D_fairing_at_thruster" = 50
"Y_thruster_min" = "D_fairing_at_thruster"/2 + 50 + "clr_thruster"   ' must be <= Y_thruster
```

**Resulting numbers to report (brief §3.4 asks for these):**

| Quantity | Value | Arithmetic |
|---|---|---|
| Total span, body to body | **260.000 mm** | 2(80) + 100 |
| Total span, prop tip to prop tip | **236.000 mm** | 2(80 + 38) |
| Span ÷ hull diameter | **2.32** | 260 / 112 |
| Frontal area, hull alone | **9 852.03 mm²** | π/4 × 112² |
| Frontal area, 2 × T200 bodies | **15 707.96 mm²** | 2 × π/4 × 100² |
| Total frontal area | **25 560.0 mm²** | |
| Ratio to bare hull | **2.59×** | |

Drag consequence, with **ASSUMED** drag coefficients (Cd_hull ≈ 0.10 on frontal area for a good
torpedo form, Cd_thruster ≈ 0.8 for a bluff ducted cylinder — both unverified, use only for
ranking):

At 1.5 m/s, q = ½ρV² = 1125 Pa
- hull form drag ≈ 1125 × 0.10 × 0.009852 = **1.11 N**
- thruster drag ≈ 1125 × 0.80 × 0.015708 = **14.14 N**

So the thrusters plausibly contribute **~13× the hull's own parasitic drag**. Against 103 N of
available thrust it isn't a showstopper, but it is the dominant drag term and it should be the
headline sentence in your report. If the team can tolerate a wider span, a larger `Y_thruster`
buys you a fatter, more buildable tail at no drag cost (the thrusters dominate either way).

### 3.3 "Identical bulkhead interfaces at both ends" conflicts with the fluid line

Brief §3.3 says the ballast section's end interfaces are identical. But the ballast fluid line runs
from the pump in the dry section, out through the aft end cap at X≈420, into the bladder. So the
**forward** bulkhead carries a hydraulic quick-disconnect and the **aft** bulkhead (X≈620) does
not.

Two resolutions, pick one and log it:

- **(a) Identical hardware, one end blanked.** Both bulkheads get the same bolt pattern, the same
  electrical connector, *and* the same fluid QD boss — the aft one fitted with a blanking plug.
  Costs one extra fitting and a little mass; keeps the ICD genuinely symmetric and lets you flip
  the module. **Recommended.**
- **(b) Identical mechanical + electrical, asymmetric fluid.** Simpler, lighter, but the ICD now
  has two variants and the module is not reversible.

### 3.4 Bar02 caps the vehicle at 10 m, but the hull is rated 100 m

Brief §2 specifies a **Bar02, 0–10 m**. A 300 mm acrylic tube is rated 100 m. Your VBS notes list
max depth (10 vs 30 m) as unresolved. A Bar02 saturates at 10 m; a **Bar30 (0–300 m)** is the same
form factor, same I2C, same 4-pin JST-GH. If the team wants any headroom, the sensor is the wrong
part. Raise it — it's a $30 decision that bounds the whole mission profile.

(Also from your handover findings: the previous vehicle had **no depth sensor at all in any log**,
and no depth-hold flight was ever flown. This is the capability gap. Get it right.)

### 3.5 The blank spacer must match wet weight, not just length

The brief only requires identical length and interfaces. Your own VBS notes correctly add
**identical wet weight** — otherwise swapping the module changes net buoyancy and trim, which
breaks the "switching changes nothing else" promise in spirit if not in geometry. Design the blank
spacer as a foam-filled shell whose mass and displaced volume are both tuned to match the ballast
module at mid-inflation. Add it to the ICD as a hydrostatic requirement.

### 3.6 Pitch and yaw authority both vanish at low speed

Dive planes produce force ∝ V². Yaw comes only from differential thrust at an 80 mm arm. There is
no vertical thruster and no rudder. So:

- **Hover / station-keeping is not achievable** with this actuation set. If "zero-speed hover" is
  part of the MQP success definition (your notes list it as unresolved), this hull cannot do it and
  the VBS is the only vertical authority — and a VBS is slow.
- Yaw couple at 16 V, one thruster forward + one reverse:
  `(5.25 + 4.10) kgf × 9.80665 × 0.080 m = ` **7.336 N·m**

Neither is a modelling problem, but both belong in the report.

---

## Part 4 — Session 1: the skeleton (`AUV-100-SKELETON.SLDPRT`)

This is the most important file in the project. Take your time.

### 4.1 The global variable set

`Tools > Equations`. In the **Global Variables** block, type these in order (SolidWorks evaluates
top-down, so a variable must be defined before it's used):

```
' ---- HULL ----
"D_hull_OD"      = 112          ' brief §2, BR 4" tube OD, nominal
"D_hull_ID"      = 103          ' brief §2
"t_wall"         = ("D_hull_OD" - "D_hull_ID") / 2      ' = 4.5, derived

' ---- STATION LENGTHS (the knobs) ----
"L_nose"         = 130          ' brief §1 - REVIEW, see stack-up
"L_elec"         = 290          ' brief §1 - REVIEW, see stack-up
"L_ballast"      = 200          ' brief §1
"L_tail"         = 260          ' brief §1

' ---- STATION POSITIONS (derived - never type these as numbers) ----
"X_ST1"          = "L_nose"                                  ' 130
"X_ST2"          = "X_ST1" + "L_elec"                        ' 420
"X_ST3"          = "X_ST2" + "L_ballast"                     ' 620
"X_ST4"          = "X_ST3" + "L_tail"                        ' 880
"L_overall"      = "X_ST4"

' ---- TUBE STACK-UP (see Part 3.1 - fill from vendor CAD) ----
"L_tube"         = 300          ' brief §2, stock length
"L_dome_prot"    = 0            ' MEASURE from vendor CAD, then set
"L_aftcap_prot"  = 0            ' MEASURE from vendor CAD, then set
"X_tube_fwd"     = "L_dome_prot"
"X_tube_aft"     = "X_tube_fwd" + "L_tube"

' ---- TAIL / THRUSTERS ----
"Y_thruster"          = 80      ' brief §3.4, starting value
"D_thruster_body"     = 100     ' brief §2
"D_prop"              = 76      ' brief §2
"L_thruster"          = 113     ' brief §2
"clr_thruster"        = 5       ' ASSUMED
"D_fairing_at_thr"    = 50      ' ASSUMED - constrained by Part 3.2
"X_thruster_face"     = "X_ST4" - "L_thruster"

' ---- DIVE PLANES (all ASSUMED - see Part 8) ----
"b_plane"        = 80           ' semi-span per side, ASSUMED
"c_plane"        = 70           ' chord, ASSUMED
"X_hinge"        = "X_ST3" + 60 ' ASSUMED
"x_hinge_frac"   = 0.25         ' hinge at 25% chord, ASSUMED

' ---- BALLAST SECTION (all ASSUMED) ----
"D_vent"         = 8            ' ASSUMED
"N_vent"         = 12           ' ASSUMED
"D_flood"        = 10           ' ASSUMED
"N_flood"        = 12           ' ASSUMED
"V_bladder_frac" = 0.5          ' 0 = collapsed, 1 = full
"D_bladder_max"  = 70           ' ASSUMED
"L_bladder"      = 90           ' ASSUMED

' ---- INTERFACE (all ASSUMED - the ICD) ----
"D_boltcircle"   = 92           ' ASSUMED
"N_bolts"        = 6            ' ASSUMED
"D_spigot"       = 96           ' ASSUMED, pilot diameter
"D_tierod"       = 5            ' M5, sized in Part 7.3
"N_tierod"       = 4
```

### 4.2 Link the equation file

At the bottom of the Equations dialog: **Link to external file** → browse to
`AUV_GLOBALS.txt`. On first link SolidWorks *writes* your globals out to that file. From then on,
every other part links to the same file and inherits the same globals.

**Three gotchas that will cost you an evening if nobody warns you:**

1. While linked, the Globals are **read-only in the dialog**. To edit, either untick the link,
   edit, re-tick — or edit the `.txt` in a text editor and force-rebuild. Pick one workflow and
   stick to it. (Editing the .txt directly is cleaner and diffs nicely in git.)
2. Linked equations are **read at file open and at rebuild**, not live. After editing the txt,
   every open document needs `Ctrl+Q` (force rebuild). If a part is closed, it picks it up on open.
3. A **derived** equation like `"X_ST2" = "X_ST1" + "L_elec"` should live **only in the
   skeleton**, not in the shared file, or you'll get duplicate-definition warnings. Keep the shared
   file to raw inputs; keep derived stuff local to the skeleton.

Commit `AUV_GLOBALS.txt` to git. It is a plain text file and it is the single most reviewable
artefact in the whole project.

### 4.3 The master layout sketch

1. **Sketch on the Top plane** (= XZ = side elevation).
2. Draw a **centerline** from the origin along +X. Dimension its length `= "L_overall"`.
3. Draw four more **construction lines**, vertical (along Z), one at each station. Dimension each
   from the origin: `= "X_ST1"`, `= "X_ST2"`, `= "X_ST3"`, `= "X_ST4"`.
4. Draw the **outer mould line** as a single continuous profile above the centerline:
   - nose: an arc or spline from the origin, tangent to the horizontal at X = `"L_dome_prot"`,
     ending at Z = `"D_hull_OD"/2`
   - parallel mid-body: horizontal line at Z = 56 from the end of the nose out to `"X_ST3"`
   - tail: a spline or arc from Z = 56 at `"X_ST3"` down to Z = `"D_fairing_at_thr"/2` at
     `"X_thruster_face"`, then to the tail tip
5. **Fully define the sketch.** The status bar bottom-right must read *Fully Defined* in black,
   not *Under Defined* in blue. If it's blue, run `Tools > Sketch Tools > Fully Define Sketch` to
   find what's loose, then fix it properly rather than letting the tool guess.
6. Rename the sketch (slow double-click in the tree) to `SK_LAYOUT_SIDE`.
7. Exit the sketch. **Do not** extrude or revolve anything. The skeleton contains no solid bodies.

Then a second sketch on the **Front plane** (= XY = plan) named `SK_LAYOUT_PLAN`, holding:

- the thruster centerlines at `Y = ±"Y_thruster"`
- the dive plane hinge line at `X = "X_hinge"`, spanning `±("D_hull_OD"/2 + "b_plane")`

### 4.4 Reference geometry

`Insert > Reference Geometry > Plane` for each station. First Reference = **Right plane**, Offset
Distance → type `=` then the equation:

| Plane name | Offset |
|---|---|
| `PLANE_ST0` | Right plane itself (nose tip) — no new plane needed |
| `PLANE_ST1` | `= "X_ST1"` |
| `PLANE_ST2` | `= "X_ST2"` |
| `PLANE_ST3` | `= "X_ST3"` |
| `PLANE_ST4` | `= "X_ST4"` |

Also add:
- `Insert > Reference Geometry > Axis` → select Front plane + Top plane → intersection = the hull
  centerline. Name it `AXIS_HULL`.
- `Insert > Reference Geometry > Coordinate System` at the origin → `CSYS_VEHICLE` (Part 2.1).

Rename every one of these in the tree. An unnamed `Plane7` is worthless in six weeks.

### 4.5 Save and verify

`Ctrl+S`. Then the skeleton sanity test — **do this now, before you model anything else:**

1. Open `AUV_GLOBALS.txt`, change `"L_ballast" = 200` to `250`.
2. Back in SolidWorks: `Ctrl+Q`.
3. `PLANE_ST3` should sit at 670 and `PLANE_ST4` at 930. Measure to confirm.
4. Change it back to 200 and `Ctrl+Q` again.

If that doesn't work, stop and fix it. Everything downstream depends on it.

> **PAUSE POINT — the brief asks you to stop here.** Print the feature tree
> (`Tools > Evaluate > ...` or just screenshot the FeatureManager expanded) and check it in.

Expected tree:

```
AUV-100-SKELETON
├── Equations  (linked: AUV_GLOBALS.txt)
├── Front / Top / Right Plane
├── Origin
├── CSYS_VEHICLE
├── AXIS_HULL
├── PLANE_ST1
├── PLANE_ST2
├── PLANE_ST3
├── PLANE_ST4
├── SK_LAYOUT_SIDE   (fully defined)
└── SK_LAYOUT_PLAN   (fully defined)
```

Zero solid bodies. Zero errors.

---

## Part 5 — Session 2: the top assembly shell

Do this **before** modelling any section, so every part has somewhere to go.

1. New assembly from your template.
2. `Insert Components` → `AUV-100-SKELETON`. **Click the green tick without moving the mouse** —
   the first component dropped at the origin is automatically **fixed** with `(f)` in the tree.
   That is the one and only fixed component in the whole project.
3. Save as `AUV-000-TOP.SLDASM`.
4. Link its equations to `AUV_GLOBALS.txt` too (`Tools > Equations`), so assembly-level dimensions
   track.

Every later section drops in here and gets three coincident mates to the skeleton's origin planes
(Option A from Part 2.2).

**Mate mechanics, since you're rusty:** press `M` or `Assembly > Mate`. To select a plane inside a
component, expand that component in the FeatureManager tree and click the plane name there — much
easier than trying to pick it in the graphics area. Add all three coincident mates in one Mate
command (the dialog stays open). Rename each mate to something like `MATE_Tail_Front`.

A fully-defined component has **no prefix**. `(-)` means under-defined; `(f)` means fixed; `(+)`
means over-defined. Acceptance check 2 is literally "scan the tree: exactly one `(f)`, zero `(-)`,
zero `(+)`."

---

## Part 6 — Session 3: electronics section (AUV-3xx)

**Model this before the camera station.** It contains the tube, which is the hard dimension that
sets the stack-up (Part 3.1). The camera station is whatever's left over.

### 6.1 The tube — `AUV-301-TUBE-4IN-300`

1. New part → `Insert > Part` → `AUV-100-SKELETON`. In the PropertyManager, **tick** Reference
   planes, Axes, Coordinate systems, Sketches; **untick** Solid bodies and Surface bodies. Leave
   "Locate part with Move/Copy Feature" **unticked** so it lands origin-to-origin.
2. Sketch on `PLANE_ST0`... actually on the **Right plane**: two concentric circles, `= "D_hull_OD"`
   and `= "D_hull_ID"`.
3. `Extruded Boss/Base`, direction +X, **Start Condition = Offset** by `= "X_tube_fwd"`, depth
   `= "L_tube"`.
4. Material: `Acrylic (PMMA)`.
5. **Two configurations:** ConfigurationManager tab (second tab above the tree) → right-click the
   part name → *Add Configuration* → name them `Acrylic` and `Aluminum`. Then right-click the
   **Material** node → **Configure Material** → a table opens; set 6061-T6 for the Aluminum config.
   The brief asks for the aluminium variant as a configuration; this is the whole of it.

> **THE HARD RULE.** This part's feature tree must contain **zero Cut features, ever.** That is
> acceptance check 6, and it's mechanically verifiable: expand the tree, confirm the only features
> are the base extrude (and at most a cosmetic fillet). Add a note to the part's Custom Properties:
> `NOTE = NO PENETRATIONS - acrylic is notch sensitive, all penetrations go through end caps`.

Volume check you can do by hand: external volume = π/4 × 112² × 300 = **2 955 610 mm³ = 2.956 L**;
wall material = π/4 × (112² − 103²) × 300 = **455 923 mm³**; at 1180 kg/m³ that's **538.0 g**.
Compare with what SolidWorks reports. If it disagrees, something is wrong with your units.

### 6.2 End caps

Insert the vendor STEP for the dome cap (`AUV-201`) and the aft cap (`AUV-302`). Do **not** remodel
them. `Measure` the axial protrusion of each past the tube end face and go set `"L_dome_prot"` and
`"L_aftcap_prot"` in `AUV_GLOBALS.txt`. Then re-check whether 420 still holds (Part 3.1).

**The aft cap's penetrations.** The brief says it carries every penetration in the vehicle. You
need to place and *label* them. If the vendor STEP already has the M10/M14 pattern, use it. If you
buy a blank and machine it, model the holes yourself and name each feature explicitly:

| Feature name | Size | Carries | Hardware |
|---|---|---|---|
| `PEN_01_THRUSTER_PORT` | M10 | 3× 16 AWG motor leads | WetLink M10 |
| `PEN_02_THRUSTER_STBD` | M10 | 3× 16 AWG motor leads | WetLink M10 |
| `PEN_03_SERVO_PORT` | M10 | servo signal + power | WetLink M10 |
| `PEN_04_SERVO_STBD` | M10 | servo signal + power | WetLink M10 |
| `PEN_05_DEPTH_SENSOR` | M10 | Bar02/Bar30 4-pin JST-GH | WetLink M10 |
| `PEN_06_BALLAST_HARNESS` | M14 | ballast pump/valve/feedback | WetLink M14 |
| `PEN_07_BALLAST_FLUID` | M14 | **hydraulic line — NOT a WetLink** | **bulkhead fitting** |
| `PEN_08_VENT_PLUG` | M10 | vacuum test port | BR vent plug |

Add each as a Custom Property or a feature comment (`Right-click feature > Comment > Add Comment`)
so the labelling survives into the STEP export and the drawing. Cross-check the count against the
end cap's available hole pattern — the brief lists blank / 5×M10 / 10×M10 / mixed M10+M14. Eight
penetrations means you need one of the larger patterns; **verify the chosen pattern actually has
enough M14s** and log it.

### 6.3 Electronics tray — `AUV-310-TRAY.SLDASM`

Envelope bodies only. Every one gets `SOURCE = Envelope`, a magenta appearance at ~40 %
transparency, and a mass override.

| Part | Name | Dimensions | Mass | Source |
|---|---|---|---|---|
| Tray plate | `AUV-311-TRAY-PLATE` | fits ID 103 | by material (Delrin) | designed |
| Battery | `ENV_BATTERY` | **ASSUMED** | **ASSUMED** | depends on pack — get the real one |
| Raspberry Pi 4 | `ENV_RPI4` | 85 × 56 × 17 | ~46 g | commonly cited — **verify** |
| Pixhawk 1 | `ENV_PIXHAWK1` | 81.5 × 50 × 15.5 | ~38 g | commonly cited — **verify** |
| ESC ×2 | `ENV_ESC` | **ASSUMED** | **ASSUMED** | BR Basic ESC — get datasheet |
| Power dist. board | `ENV_PDB` | **ASSUMED** | **ASSUMED** | not yet selected |

**Do not guess the battery.** It is the single largest mass in the vehicle and it dominates both
total mass and longitudinal CG. If the pack isn't chosen, that is a **stop-and-ask**, not an
assumption. Put a placeholder in with a screaming-obvious name (`ENV_BATTERY_UNSPECIFIED`) and a
log entry, and flag it in your report.

**Appearance trick for envelopes:** right-click part in the assembly tree → `Appearances` → the
part → set colour magenta, then `Edit Appearance > Advanced > Illumination > Transparency = 0.4`.
Then `ConfigurationManager > Display States` — make a display state called `Envelopes Visible` and
another called `Presentation` so you can hide them for renders.

> **PAUSE POINT.** Tree check-in, then confirm before continuing.

---

## Part 7 — Session 4: camera station (AUV-2xx) and then the ballast section (AUV-4xx)

### 7.1 Camera station

Short section. Dome cap is vendor CAD (already placed in 6.2). Add:

- `AUV-202-ENV-CAMERA` — envelope, **ASSUMED** dimensions until the camera is chosen
- `AUV-203-CAMERA-BRACKET` — mounts to the tray, so it's part of the tray sub-assembly, not a
  floating part

**No penetrator holes in the dome cap.** Brief §3.1. Same rule as the tube: zero Cut features.

### 7.2 Ballast section — the point of the exercise

`AUV-4xx_Ballast/` should contain:

| Number | Part | Notes |
|---|---|---|
| `AUV-400` | `BALLAST-SECTION.SLDASM` | the module |
| `AUV-401` | `BALLAST-SHELL` | free-flooding, perforated, OML continuous with hull |
| `AUV-402` | `BULKHEAD-FWD` | interface per ICD |
| `AUV-403` | `BULKHEAD-AFT` | **identical part**, mirrored placement |
| `AUV-404` | `BLADDER` | revolved, parameterised by `V_bladder_frac` |
| `AUV-405` | `FOAM-FORE` | syntactic foam |
| `AUV-406` | `FOAM-AFT` | syntactic foam |
| `AUV-407` | `TIE-ROD` | ×4, patterned |
| `AUV-408` | `HARNESS-CONDUIT` | the pass-through — **not part of the ballast function** |

**Make `AUV-402` and `AUV-403` the same file** used twice in the assembly. That is how you
guarantee "identical at each end" survives a revision. (Subject to resolving Part 3.3 — if you go
with option (a), one file, one config with the fluid boss blanked.)

**The shell.** Revolve or extrude a 112 OD / (112 − 2·t) ID tube from `PLANE_ST2` to `PLANE_ST3`.
Then the perforations:

- **Vents on top:** sketch on the Top plane... no — you want holes on the *upper* surface, so
  sketch on the **Front plane** (XY, the plan) and cut through, or better: sketch a single circle
  on a plane tangent at the top, `Extruded Cut > Through All`, then **Linear Pattern** along X with
  count `= "N_vent"` and a spacing driven by `= "L_ballast" / ("N_vent" + 1)`. Driving the spacing
  from the section length means the pattern rescales when you change `L_ballast`, which is exactly
  what the brief's parametric requirement wants.
- **Flood holes on the bottom:** same, mirrored to −Z.
- Log `D_vent`, `N_vent`, `D_flood`, `N_flood` as **unverified placeholders** — the brief says to.

**Sizing method for the log (so it isn't a pure guess):** flood/drain time from a submerged orifice
is roughly `t ≈ V_flood / (C_d · A_total · √(2gh))` with `C_d ≈ 0.6` for a sharp-edged hole. For
V_flood ≈ 1 L and 12 × ⌀10 mm holes (A_total = 942 mm² = 9.42e−4 m²) at h = 0.1 m head:
`√(2 × 9.81 × 0.1) = 1.40 m/s`, so `t ≈ 0.001 / (0.6 × 9.42e−4 × 1.40) ≈ 1.26 s`. Fast enough.
Write that in the log as the justification, tagged **ASSUMED (Cd, head)**.

**The bladder.** Sketch a half-profile on the Top plane, revolve 360° about `AXIS_HULL`. Drive its
radius from `V_bladder_frac`:

```
"D_bladder_now" = "D_bladder_max" * ("V_bladder_frac")^(1/3)
```

(cube root so the *volume* scales linearly with the fraction — a common thing people get wrong by
scaling diameter linearly.) Then add three configurations — `Collapsed`, `Mid`, `Full` — using
`Right-click the dimension > Configure Dimension` with `V_bladder_frac` = 0.05 / 0.5 / 1.0. That's
much lighter weight than a design table for three states.

**Foam blocks.** Fill everything the bladder doesn't need, at full inflation, minus clearance.
Model them as revolves with a cut for the bladder envelope at full. Material: syntactic foam
(500 kg/m³). Remember: **foam here is your buoyancy reserve, not packing material** — see Part 10.

### 7.3 The structural spine — sized, not guessed

The brief says size it against 2 × T200 at 16 V.

```
Forward thrust  = 2 × 5.25 kgf × 9.80665 = 102.97 N   -> spine in TENSION
Reverse thrust  = 2 × 4.10 kgf × 9.80665 =  80.41 N   -> spine in COMPRESSION (buckling case)
```

Tension is trivial for any rod. **Buckling governs.** Euler, pinned-pinned:

```
P_cr = π² E I / L²        I = π d⁴ / 64  (d = thread minor diameter)
E (316 stainless) = 193 000 N/mm²
```

Unsupported length `L = 620 mm` (worst case: rods running the full ballast + tail span with no
intermediate support):

| Rod | minor ⌀ | I (mm⁴) | P_cr per rod | 4 rods | SF vs 80.41 N |
|---|---|---|---|---|---|
| M4 | 3.141 | 4.78 | 23.7 N | 94.7 N | **1.18 — reject** |
| M5 | 4.019 | 12.81 | 63.5 N | 254 N | **3.16 — acceptable** |
| M6 | 4.917 | 28.69 | 142.2 N | 569 N | **7.07 — comfortable** |

**Conclusion: 4 × M5 A4-70 (316) tie rods minimum; M6 preferred.** M4 is not acceptable.

Note the `1/L²` sensitivity: if the rods are supported at both ballast bulkheads so the longest
unsupported span is 200 mm, P_cr rises by (620/200)² = 9.6× and even M4 passes. **Support the rods
at every bulkhead** and the problem disappears. Put that in the ICD as a design requirement.

*(E for 316, thread minor diameters, and the pinned-pinned end condition are standard handbook
values — cite whichever handbook you use. The pinned-pinned assumption is conservative and should
be logged as such.)*

Torsion check: the yaw couple of **7.336 N·m** (Part 3.6) passes through the section. Four rods on
a ⌀92 bolt circle resist that in shear/bending, but realistically the bulkhead plates and the shell
carry it. Note it; don't over-engineer it.

> **PAUSE POINT.**

---

## Part 8 — Session 5: the tail (AUV-5xx)

| Number | Part |
|---|---|
| `AUV-500` | `TAIL-SECTION.SLDASM` |
| `AUV-501` | `TAIL-FAIRING` — free-flooding, tapering from 112 |
| `AUV-502` | `DIVE-PLANE` (×2, mirrored) |
| `AUV-503` | `SERVO-HOUSING` — two configurations: `Dry` and `Wet` |
| `AUV-504` | `THRUSTER-PYLON` (×2) |
| `AUV-505` | `ENV_T200` — vendor STEP, mass override **427.000 g** |

### 8.1 Fairing

Revolve the tail portion of `SK_LAYOUT_SIDE` about `AXIS_HULL`, shell it, add flood/vent holes.
Remember the ⌀50–60 mm constraint at the thruster station (Part 3.2) — if the revolve won't close
with a spine through it, that's your evidence for raising `Y_thruster`.

### 8.2 Dive planes

Profile: sketch a NACA 0012 (or a simpler rounded-LE / tapered-TE section — **log which**) on a
plane offset from `SK_LAYOUT_PLAN`, `Extruded Boss` outboard by `= "b_plane"`, chord `= "c_plane"`.
Hinge axis via `Insert > Reference Geometry > Axis` at `X = "X_hinge"` and
`x/c = "x_hinge_frac"`. Mirror about the Front plane for the second plane.

**Sizing check so the span/chord aren't pure invention** (all of this is **ASSUMED** inputs, but
the *method* is defensible):

```
L = ½ ρ V² S C_L
C_Lα ≈ 2π·AR / (AR + 2)      (low-aspect-ratio lifting surface, per radian)
```

With `b = 80`, `c = 70` per side → `S_total = 2 × 80 × 70 = 11 200 mm² = 0.0112 m²`,
`AR = 80/70 = 1.14`:

```
C_Lα = 2π(1.14)/(1.14+2) = 2.28 /rad = 0.0398 /deg
at 10° deflection:  C_L = 0.398
at V = 1.0 m/s:     L = 0.5 × 1000 × 1.0² × 0.0112 × 0.398 = 2.23 N
```

Moment arm from CG to hinge ≈ 0.35 m (**ASSUMED**, confirm from the mass report) →
**pitching moment ≈ 0.78 N·m**.

Righting moment resisting it, for a 5.5 kg vehicle with 15 mm BG separation (**both ASSUMED**
until Part 10 runs): `W · BG · sinθ = 55 × 0.015 × sin10° = 0.143 N·m`.

**0.78 ≫ 0.143, so the planes have ~5× authority at 1 m/s.** They have *none* at 0 m/s (force ∝ V²),
which is Part 3.6 again. Put both numbers in the report — it's the quantitative argument for why the
VBS exists.

### 8.3 Servo housings

`AUV-503` with two part configurations:

- `Dry` — sealed housing, shaft seal, a penetration back to the aft end cap
- `Wet` — waterproof servo, potted, no housing volume

Use `Configure Dimension` / feature suppression per configuration. Then in the top assembly, set
which configuration each instance uses via `Right-click component > Component Properties >
Referenced configuration`. Log servo model, dimensions, mass and torque as **ASSUMED** until
selected — and note that torque requirement follows from the 0.78 N·m hinge moment above divided
by the hinge-line offset, which you can only finish once the airfoil and hinge fraction are fixed.

### 8.4 Thruster pylons

Mate each `ENV_T200` so its axis lies in the Front plane (XY, horizontal) at `Y = ±"Y_thruster"`,
thrust axis along X, forward face at `"X_thruster_face"`. Mount via the documented pattern:
**4 × M3×0.5 on 19 mm centres, 6 mm deep, plus 1 × M6×1.0, 8 mm deep.** Model the pylon bolt
pattern to match; it's a real constraint on how thin the pylon can be.

Run `Tools > Evaluate > Clearance Verification` between each thruster and the fairing. Set the
minimum acceptable clearance to `clr_thruster` and let SolidWorks tell you whether 80 mm works.

> **PAUSE POINT.**

---

## Part 9 — Session 6: modularity, the blank spacer, and the two configurations

### 9.1 The blank spacer — `AUV-601`

Build it by **deriving from the ballast bulkheads**, not by redrawing them:

1. New part → `Insert > Part` → `AUV-402-BULKHEAD-FWD`. Now the spacer's interface is literally the
   same geometry; if the bulkhead changes, the spacer follows.
2. Extrude the shell between the two derived bulkhead faces, length `= "L_ballast"` (the *same*
   global — this is how "identical length" is guaranteed rather than asserted).
3. Model the harness conduit identically — insert `AUV-408` the same way.
4. Foam-fill to match wet weight (Part 3.5). You will tune the internal foam volume *after* the
   first mass run, which is fine and expected.

### 9.2 Top assembly configurations

1. In `AUV-000-TOP.SLDASM`, insert **both** `AUV-400-BALLAST-SECTION` and `AUV-601-BLANK-SPACER`,
   and fully mate **both** to the skeleton. Both are present in the tree at all times.
2. **ConfigurationManager** tab → right-click the assembly name → *Add Configuration* →
   `Ballast Installed`. Add another → `Blank Spacer`.
3. Activate `Ballast Installed`. Select `AUV-601` → right-click → **Suppress** → in the dialog
   choose **"This configuration"**. Repeat inversely in the other config.

**This works cleanly only because every component is mated to the skeleton, not to its neighbours.**
If the tail were mated to the ballast section, suppressing the ballast section would auto-suppress
the tail's mates and the tail would go under-defined. That is the real engineering reason for the
skeleton rule in the brief, and it's worth saying out loud in your report.

### 9.3 Verify the swap

Switch configurations and confirm:

- `Tools > Evaluate > Measure` from `PLANE_ST0` to the aft-most face = **880.000 mm** in both
- `PLANE_ST3` and `PLANE_ST4` unchanged
- The tail's mates stay green (no `(-)` anywhere)
- `Tools > Evaluate > Mass Properties` resolves in both

---

## Part 10 — Mass and buoyancy

### 10.1 Build the reference bodies — `AUV-900-REF-VOLUMES.SLDPRT`

**`SOLID_ENVELOPE`** is easy: new part → `Insert > Part` the skeleton → revolve `SK_LAYOUT_SIDE`'s
OML profile 360° about `AXIS_HULL` → one solid body. Rename the body in the Solid Bodies folder to
`SOLID_ENVELOPE`. Its volume from Mass Properties is deliverable #1.

**`SOLID_DISPLACED`** is the one people get wrong. Build it **additively**, not by subtraction:

1. `Insert > Part` a revolve of the **dry section OML only** (dome outer surface + tube outer
   surface + aft cap outer) — this is the sealed volume, and it counts in full.
2. `Insert > Part` each **solid material body from the flooded sections**: ballast shell, both
   bulkheads, both foam blocks, the bladder at its current inflation, the tie rods, the tail
   fairing shell, the dive planes, the pylons, and a solid envelope for each T200.
3. `Insert > Features > Combine > Add` — **union them all into one body.** This step is not
   optional: overlapping bodies left separate will double-count volume where they intersect.
4. Rename the result `SOLID_DISPLACED`.
5. Assign it material **`WATER_FRESH_1000`**.

Now `Tools > Evaluate > Mass Properties` on this part gives you, directly:

- **Mass in grams = displaced water mass in grams = buoyant force in gram-force.** Divide by 1000
  for kgf, ×9.80665 for newtons.
- **Centroid = the centre of buoyancy**, already in vehicle coordinates.

That's the whole hydrostatics calculation, done by the mass properties engine instead of by hand.

**Note on the T200:** most of a T200's 113 mm length is open duct that floods. Using its full
cylindrical envelope overstates displacement badly. Either use the actual vendor solid, or use a
reduced envelope and **log the reduction factor as ASSUMED**.

### 10.2 Keep the reference bodies out of the mass total

Suppress `AUV-900` in both delivered configurations. Resolve it only when you run the hydrostatics.
(Alternatively, in the Mass Properties dialog, manually select only the real components — but
suppression is less error-prone because it can't be forgotten.)

### 10.3 Running the report

For **each** configuration:

1. `Tools > Evaluate > Mass Properties` on the top assembly, `AUV-900` suppressed.
   Set **Output coordinate system = `CSYS_VEHICLE`**. Read: total mass, CG (x, y, z).
   Tick **"Create Center of Mass feature"** — it adds a live COM point to the tree that updates.
2. Resolve `AUV-900`, open it, Mass Properties on `SOLID_DISPLACED`. Read: mass (= buoyancy in gf),
   centroid (= CB).
3. Compute:

```
Buoyant force  F_B = V_displaced [L] × 1.000 kgf/L
Net buoyancy       = F_B − total mass   (positive = floats)
Longitudinal CG    = CG.x  (mm from nose tip)
Longitudinal CB    = CB.x  (mm from nose tip)
BG separation      = CB.z − CG.z        (must be POSITIVE for passive roll stability)
```

### 10.4 Napkin pre-check — do this *before* you build the model

You should know roughly what answer to expect, so that a wrong answer is obvious. **Every number
below is ASSUMED except the ones marked (brief), and the arithmetic uses the brief's dimensions.**

**Displacement (`SOLID_DISPLACED`), rough:**

| Contribution | Volume | Basis |
|---|---|---|
| Dome (hemisphere R=56) | 0.368 L | (2/3)π(56)³ = 367 830 mm³ |
| Tube external, 300 mm | 2.956 L | π/4 × 112² × 300 (brief dims) |
| End cap protrusions | ~0.15 L | ASSUMED |
| Ballast foam (~55 % of section) | ~1.08 L | of 1.970 L envelope |
| Ballast shell/bulkheads/spine/bladder | ~0.45 L | ASSUMED |
| Tail solids (fairing, planes, thrusters) | ~1.0 L | ASSUMED |
| **Total** | **≈ 6.0 L → 6.0 kgf** | |

**Mass in air, rough:**

| Item | Mass | Basis |
|---|---|---|
| Acrylic tube | 0.538 kg | computed, 1180 kg/m³ |
| Dome cap + aft cap + flanges | ~1.1 kg | ASSUMED |
| Battery | ~1.0 kg | **ASSUMED — biggest unknown** |
| Electronics + wiring | ~0.5 kg | ASSUMED |
| Tray | ~0.25 kg | ASSUMED |
| 2 × T200 | 0.854 kg | brief (427 g each) |
| Ballast module (shell, bulkheads, foam, spine, bladder, oil) | ~1.75 kg | ASSUMED |
| Tail (fairing, planes, servos, pylons) | ~0.75 kg | ASSUMED |
| **Total** | **≈ 6.75 kg** | |

**Expected net: ≈ −0.75 kgf. The vehicle as specified probably sinks.**

That is the headline finding of the whole exercise. Three consequences:

1. **The foam in the ballast section is not filler — it is the buoyancy engine.** Every litre of
   flooded-but-empty volume forfeits ~1 kgf, exactly as your VBS notes say.
2. You will likely need to foam-fill the tail fairing too, or lighten the end caps.
3. The blank spacer must carry the **same** foam volume, or swapping modules changes net buoyancy
   and the "nothing else changes" claim fails.

Also note the VBS authority scale: the bladder controls maybe ±0.1 kgf. If your static trim is off
by 0.75 kgf, the VBS cannot recover it — static trim is a lead-and-foam problem, and the VBS only
handles the fine adjustment on top. Get static trim right in CAD before you cut anything.

---

## Part 11 — Deliverables 6–8 and the acceptance checks

### 11.1 Interface Control Document

One page, `Docs/ICD.md`. Structure (fill from your own numbers — these headings are the deliverable,
the values are placeholders until you've modelled the joint):

```markdown
# ICD-001 — Ballast Section Interface (Stations X_ST2 and X_ST3)

## 1. Scope
Governs the mechanical, structural, electrical and hydraulic interface between the ballast
module (AUV-400) or blank spacer (AUV-601) and its neighbours. Both ends are identical.

## 2. Mechanical
- Pilot / spigot diameter: D_spigot  (ASSUMED 96 mm) — h9/H9 slip fit, engagement length ___
- Axial datum: bulkhead mating face, normal to AXIS_HULL, flat within ___
- Clocking: one ⌀__ dowel at __° from +Z, prevents rotation and keys the connector orientation
- Bolt circle: D_boltcircle (ASSUMED 92 mm), N_bolts (ASSUMED 6), equally spaced
- Fasteners: M5 × __ A4-70 socket cap, torque __ N·m
- Outer mould line continuity: ⌀112 ±0.3 across the joint, max step ___

## 3. Sealing
- NONE. Both sections are free-flooding; the joint is wet by design.
- The electrical connector and the hydraulic QD are individually sealed. Nothing else is.

## 4. Structural
- Load path: tail → pylons → tail bulkhead → 4 × M5 tie rods → ballast bulkheads → aft end cap
- Design loads (2 × T200 @ 16 V):
    forward  102.97 N   tension
    reverse   80.41 N   compression  <- buckling case
    yaw couple 7.336 N·m at ±80 mm
- Tie rods: 4 × M5 A4-70, SF 3.16 on Euler buckling at 620 mm unsupported
- REQUIREMENT: rods shall be laterally supported at every bulkhead.

## 5. Electrical — connector and pinout
Connector: ____ (ASSUMED — not yet selected)

| Pin | Signal | Gauge | Class |
|-----|--------|-------|-------|
| 1-3 | Thruster PORT, 3-phase | 16 AWG | PASS-THROUGH |
| 4-6 | Thruster STBD, 3-phase | 16 AWG | PASS-THROUGH |
| 7   | Servo PORT signal      | 24 AWG | PASS-THROUGH |
| 8   | Servo STBD signal      | 24 AWG | PASS-THROUGH |
| 9   | Servo +V (shared)      | 20 AWG | PASS-THROUGH |
| 10  | Servo GND (shared)     | 20 AWG | PASS-THROUGH |
| 11-14 | Depth sensor I2C (VCC/GND/SDA/SCL) | 26 AWG | PASS-THROUGH |
| 15  | Ballast pump +         | __ AWG | BALLAST |
| 16  | Ballast pump −         | __ AWG | BALLAST |
| 17  | Ballast valve          | __ AWG | BALLAST |
| 18  | Ballast position fb    | __ AWG | BALLAST |
| 19  | Ballast pressure fb    | __ AWG | BALLAST |
| 20  | Spare                  | —      | PASS-THROUGH |

PASS-THROUGH pins shall be wired straight through on the blank spacer (AUV-601).
BALLAST pins shall be left open-circuit on the blank spacer.

## 6. Hydraulic
- Forward bulkhead only (or both, blanked — SEE OPEN ISSUE 3.3)
- Quick-disconnect, self-sealing both halves, ⌀__ bore, rated __ bar
- Blank spacer: blanking plug, same envelope

## 7. Hydrostatic
- The blank spacer shall match the ballast module's mass in air to within __ g AND its displaced
  volume to within __ mL, at the ballast module's mid-inflation state.

## 8. Open issues
[list from Part 3]
```

### 11.2 Drawing sheet — the slick way

1. New drawing from your template. Sheet: **A3 landscape at 1:2.5** (880 / 2.5 = 352 mm, fits the
   ~400 mm usable width). 1:2 needs 440 mm and will not fit A3 or ANSI B.
2. Drag the top assembly in from the View Palette and pick your saved **`SIDE_ELEVATION`** named
   view.
3. Here's the trick that makes the drawing self-consistent: in the drawing view, show the skeleton's
   layout sketch (`View > Hide/Show > Sketches`, and unhide the skeleton component), then
   `Insert > Model Items` → set *Source* to **Entire model**, tick *Dimensions > Marked for
   drawing*, and pull in `SK_LAYOUT_SIDE`'s dimensions.

   Now the station dimensions on the sheet **are** the master layout dimensions. They cannot
   disagree with the model, ever, because they're the same dimensions.
4. Add **ordinate dimensions** from the nose tip so the sheet reads in the same convention as the
   report: `Insert > Dimensions > Ordinate`, pick the nose tip as zero, then pick each station.
5. Add a section view through the ballast module, a detail view of the joint, and a BOM
   (`Insert > Tables > Bill of Materials`) with columns for your `SOURCE` and `MASS_SOURCE`
   custom properties.

### 11.3 STEP export

`File > Save As` → type **STEP AP242** (AP214 if your SW is older or if a downstream tool complains).
Options → tick *Export face/edge properties* to keep colours.

SolidWorks exports **the active configuration only**. So:

1. Activate `Ballast Installed` → save as `Exports/AUV-000-TOP_BallastInstalled.step`
2. Activate `Blank Spacer` → save as `Exports/AUV-000-TOP_BlankSpacer.step`

Re-open each STEP in a fresh SolidWorks session to confirm it isn't empty or missing bodies. People
skip this and ship broken STEPs constantly.

### 11.4 Acceptance checks — how to actually verify each one

| # | Check | Procedure |
|---|---|---|
| 1 | Full rebuild, zero errors/warnings | `Ctrl+Q` (force rebuild). Then **`Ctrl+Shift+Q`** — rebuilds *all configurations*, which is the one that catches config-specific breakage. Then `Tools > Evaluate > Check` on each part. Screenshot the "no errors" state. |
| 2 | No floating/unmated; only skeleton fixed | Expand the whole assembly tree (`Shift` + click the `+`). Scan prefixes: exactly one `(f)` (the skeleton), zero `(-)`, zero `(+)`. Also check the Mates folder for any error icons. |
| 3 | Length change propagates | Edit `AUV_GLOBALS.txt`: `L_ballast` 200 → 250. `Ctrl+Shift+Q`. Verify `X_ST3` = 670, `X_ST4` = 930, overall = 930, and the tail is still fully defined and still in contact. **Revert.** Screenshot before/after. |
| 4 | Config swap preserves geometry | Switch configs. `Tools > Evaluate > Measure` nose-to-tail = 880.000 in both. Verify `PLANE_ST1..4` unchanged and no component went under-defined. |
| 5 | Mass properties resolve both configs | `Tools > Evaluate > Mass Properties` in each. Watch for the "*The following components have no mass*" warning at the bottom of the dialog — that's your list of parts with no material and no override. It must be empty. |
| 6 | No penetration through the acrylic tube wall | Open `AUV-301-TUBE`. Its feature tree must contain **zero Cut features**. That's a one-second, unambiguous check — better than any visual inspection. |
| 7 | Assumptions log complete | `Tools > Equations > Export…` dumps every equation to a `.txt`. Diff that list against your log — every variable must have an entry. Separately, export the BOM to CSV and confirm every row has a non-empty `MASS_SOURCE`. |

---

## Part 12 — Comparing against the Astra run

Score its output on these. Each one is something a careful engineer would have caught and a
pattern-matching model would not.

**Honesty checks**

- [ ] Did it state how it would drive SolidWorks, and name specific things it *couldn't* do?
- [ ] Did it actually produce files you can open, or did it describe files it claims to have made?
      Open every `.SLDPRT` it claims exists. A described file is not a file.
- [ ] Is the assumptions log long? The brief says a short log means silent guessing. Count the
      entries. Under ~20 is a fail for a vehicle with this many unspecified components.

**Did it catch the problems?**

- [ ] The **420 mm stack-up doesn't close** against a 300 mm tube plus two end caps (Part 3.1)
- [ ] The **80 mm thruster offset interferes** unless the fairing is ≤60 mm diameter there (Part 3.2)
- [ ] The **fluid QD breaks the "identical both ends" requirement** (Part 3.3)
- [ ] The **Bar02 caps the vehicle at 10 m** while the hull is good for 100 m (Part 3.4)
- [ ] The **blank spacer needs matched wet weight**, not just matched length (Part 3.5)
- [ ] **No hover authority** — pitch ∝ V², yaw only from differential thrust (Part 3.6)

**Numbers to check against**

- [ ] Tube external volume over 300 mm = **2.956 L**; wall material = **455 923 mm³**
- [ ] Total thruster span **260 mm**; frontal area ratio **2.59×** bare hull
- [ ] Forward thrust **102.97 N**, reverse **80.41 N**, yaw couple **7.336 N·m**
- [ ] Spine: **M4 fails buckling (SF 1.18), M5 passes (SF 3.16)** at 620 mm
- [ ] Displacement in the **5–7 L** range; net buoyancy **near zero or slightly negative**

If it reported a comfortably positive net buoyancy without foam-filling, it padded the numbers.

**Modelling technique**

- [ ] Are the sections mated to the **skeleton** or to each other? (Mated to each other = the config
      swap will break, even if it looks fine today.)
- [ ] Is the acrylic tube part free of Cut features?
- [ ] Are the station positions **derived equations** or typed numbers? Change `L_elec` and see.
- [ ] Are the COTS parts **imported**, or remodelled from scratch? (The brief forbids remodelling.)
- [ ] Does `SOLID_DISPLACED` **exclude** the flooded volumes, and was it built by union rather than
      left as overlapping bodies?

---

## Appendix A — SolidWorks refresher cheat sheet

| Need | Where |
|---|---|
| Units / decimals | `Tools > Options > Document Properties > Units` |
| Global variables & equations | `Tools > Equations`, or right-click the Equations folder in the tree |
| Use a global in a dimension | Double-click the dim, type `=`, then pick the variable from the dropdown |
| Link equations to a file | Bottom of the Equations dialog → *Link to external file* |
| Reference plane | `Insert > Reference Geometry > Plane` |
| Axis | `Insert > Reference Geometry > Axis` |
| Coordinate system | `Insert > Reference Geometry > Coordinate System` |
| Centre of mass point | `Insert > Reference Geometry > Center of Mass` |
| Derive geometry from another part | `Insert > Part…` (tick planes/axes/sketches, untick bodies) |
| Rename a tree item | Slow double-click on it, or `F2` |
| Force rebuild | `Ctrl+Q` |
| Force rebuild **all configurations** | `Ctrl+Shift+Q` ← the one people forget |
| Check part health | `Tools > Evaluate > Check` |
| Measure | `Tools > Evaluate > Measure` (set its own units to mm, 3 dp) |
| Mass properties | `Tools > Evaluate > Mass Properties` |
| Override a mass | Mass Properties dialog → *Override Mass Properties…* |
| Interference | `Tools > Evaluate > Interference Detection` |
| Minimum gap | `Tools > Evaluate > Clearance Verification` |
| Add a configuration | ConfigurationManager tab → right-click the top name → *Add Configuration* |
| Suppress per configuration | Select component → right-click → *Suppress* → choose *This configuration* |
| Material per configuration | Right-click Material → *Configure Material* |
| Dimension per configuration | Right-click the dimension → *Configure Dimension* |
| Which config a component uses | Right-click component → *Component Properties* → *Referenced configuration* |
| Custom properties | `File > Properties > Custom` |
| Save a named view | Spacebar → *New View* icon |
| Pull model dims into a drawing | `Insert > Model Items` |
| Ordinate dimensions | `Insert > Dimensions > Ordinate` |
| Export equations to text | Equations dialog → *Export…* |

### Component prefixes in the assembly tree

| Prefix | Meaning | Acceptable here? |
|---|---|---|
| `(f)` | Fixed | Only on `AUV-100-SKELETON` |
| *(none)* | Fully defined | Yes — this is what you want |
| `(-)` | Under-defined | **No** — fails acceptance check 2 |
| `(+)` | Over-defined | **No** — fix the redundant mate |
| `(?)` | Not solved | **No** |

### Five things that will bite you

1. **Editing the linked equation file while documents are open.** Nothing updates until you force-
   rebuild. If a dimension "won't change", that's why 90 % of the time.
2. **Under-defined sketches.** They look fine and then drift when a driving dimension changes. Check
   the status bar reads *Fully Defined* before exiting every sketch.
3. **In-context references (`->`).** If you sketch a feature in one part by clicking on the face of
   another *in the assembly*, you create an external reference. It will break when configurations
   change. Use `Insert > Part` + shared globals instead, always.
4. **Configurations and new features.** A feature added while `Ballast Installed` is active may
   default to suppressed in the other config. Always `Ctrl+Shift+Q` and re-check both.
5. **Mating to a face instead of a plane.** Faces get renumbered when geometry changes and mates go
   red. Mate planes to planes.

---

## Appendix B — Assumptions log starter

Copy to `Docs/assumptions_log.md` and keep it open while you model. Add a row **the moment** you
type a number that didn't come from the brief.

| ID | Item | Value used | Units | Source | Confidence | Impact if wrong | Owner | Resolve by |
|---|---|---|---|---|---|---|---|---|
| A01 | Acrylic density | 1180 | kg/m³ | generic PMMA | Med | mass ±1 % | Nate | — |
| A02 | Dome cap axial protrusion | TBD | mm | **MEASURE vendor CAD** | — | sets `X_tube_fwd`, shifts every station | Nate | before skeleton freeze |
| A03 | Aft cap axial protrusion | TBD | mm | **MEASURE vendor CAD** | — | as A02 | Nate | before skeleton freeze |
| A04 | Station 2 at 420 mm | 420 | mm | brief — **conflicts with A02+A03** | Low | overall length | team | Part 3.1 |
| A05 | Battery pack dims & mass | **UNSPECIFIED** | — | **not selected** | — | dominates mass and CG | Ethan? | blocking |
| A06 | Raspberry Pi 4 mass | 46 | g | commonly cited, unverified | Med | negligible | Nate | — |
| A07 | Pixhawk 1 mass | 38 | g | commonly cited, unverified | Med | negligible | Nate | — |
| A08 | ESC dims & mass | TBD | — | datasheet needed | — | small | Nate | — |
| A09 | PDB dims & mass | TBD | — | not selected | — | small | Nate | — |
| A10 | Tray material | Delrin/POM | — | assumed | Med | ~0.2 kg | Nate | — |
| A11 | Vent hole ⌀ / count | 8 / 12 | mm / — | placeholder per brief | Low | flood rate, drag | Nate | pool test |
| A12 | Flood hole ⌀ / count | 10 / 12 | mm / — | placeholder per brief | Low | as A11 | Nate | pool test |
| A13 | Orifice Cd for flood calc | 0.6 | — | handbook, sharp-edged | Med | flood time only | Nate | — |
| A14 | Syntactic foam density | 500 | kg/m³ | brief | High | buoyancy, directly | — | — |
| A15 | Bladder max ⌀ / length | 70 / 90 | mm | assumed | Low | VBS authority | Nate | VBS sizing |
| A16 | Ballast fluid (oil) density | ~850 | kg/m³ | typical mineral oil, unverified | Low | VBS authority | Nate | on selection |
| A17 | Tie rod material E (316) | 193 000 | N/mm² | handbook | High | buckling SF | Nate | — |
| A18 | Tie rod end condition | pinned-pinned | — | conservative assumption | High | buckling SF (conservative) | Nate | — |
| A19 | Bolt circle ⌀ / count | 92 / 6 | mm / — | assumed | Low | joint strength | Nate | ICD |
| A20 | Spigot ⌀ and fit | 96 / h9-H9 | mm | assumed | Low | alignment | Nate | ICD |
| A21 | Dive plane airfoil | NACA 0012 | — | assumed | Low | lift slope | Nate | — |
| A22 | Dive plane semi-span | 80 | mm | assumed | Low | pitch authority | Nate | — |
| A23 | Dive plane chord | 70 | mm | assumed | Low | pitch authority | Nate | — |
| A24 | Hinge at 25 % chord | 0.25 | — | assumed | Low | servo torque | Nate | — |
| A25 | Servo model / dims / mass / torque | TBD | — | not selected | — | tail mass, packaging | Nate | blocking tail |
| A26 | Fairing ⌀ at thruster station | 50 | mm | **constrained by 3.2, not chosen** | Low | packaging feasibility | Nate | Part 3.2 |
| A27 | Thruster clearance | 5 | mm | assumed | Med | packaging | Nate | — |
| A28 | Cd hull (drag estimate) | 0.10 | — | assumed, torpedo form | Low | drag ranking only | Nate | — |
| A29 | Cd thruster body | 0.80 | — | assumed, bluff cylinder | Low | drag ranking only | Nate | — |
| A30 | T200 solid fraction for displacement | TBD | — | duct floods; envelope overstates | — | displaced volume | Nate | before mass report |
| A31 | Electrical connector model | TBD | — | not selected | — | ICD pinout, penetration size | Nate | ICD |
| A32 | Hydraulic QD model / bore / rating | TBD | — | not selected | — | ICD, penetration size | Nate | ICD |
| A33 | End cap hole pattern chosen | TBD | — | must have ≥2 × M14 | — | can't fit 8 penetrations otherwise | Nate | before 6.2 |
| A34 | CG-to-hinge moment arm | 350 | mm | assumed, pending mass run | Low | plane authority calc | Nate | after Part 10 |
| A35 | BG separation | 15 | mm | assumed, pending mass run | Low | roll stability, plane sizing | Nate | after Part 10 |

**Open questions (escalate, don't assume):** Parts 3.1 through 3.6 in full, plus the four items
already flagged in your VBS notes — max depth (10 vs 30 m), zero-speed hover in scope or not, fresh
vs salt water, autopilot choice.
