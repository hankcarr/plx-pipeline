# PCB Footprint Standardization Pipeline

## Overview

A tool to download PCB component footprints from various sources, synthesize a
standardized footprint using Brant and Graham's method, and import the result
into Pulsonix for use in PCB design.

## Team

- **Hank** — project lead, Pulsonix user
- **Brant (Frack)** — programming/code review; currently performs synthesis manually
- **Graham (Frick)** — hardware/schematics
- **Target fab** — PCBway

---

## Synthesis Method

### Brant's Rule
For each pad, take the **maximum extent** across all source footprints. The
result is a footprint that every source variant fits within.

### Constraints
Pad expansion is bounded by PCBway's manufacturing design rules. If expanding
a pad would violate minimum spacing, the conflict is flagged for manual review
rather than silently producing an unmanufacturable footprint.

---

## PCBway Design Rule Limits

| Parameter                  | Absolute Minimum  | Recommended       |
|----------------------------|-------------------|-------------------|
| Min trace width            | 0.1mm / 4mil      | 0.15mm / 6mil     |
| Min spacing                | 0.1mm / 4mil      | 0.15mm / 6mil     |
| Min drill size (CNC)       | 0.2mm             | ≥0.3mm            |
| Finished hole diameter     | 0.2mm – 6.2mm     | —                 |
| Drill size tolerance       | ±0.08mm           | —                 |
| Min annular ring width     | 0.15mm / 6mil     | —                 |
| Min silkscreen char height | 0.8mm             | —                 |
| Min silkscreen char width  | 0.15mm            | —                 |

> Holes smaller than 0.3mm or larger than 6.3mm incur extra charges.
> Strongly recommended to design traces and spacing at 0.15mm / 6mil or above.

---

## Pipeline Steps

1. **Input** — multiple PLX files for the same component from various sources
2. **Parse** — extract pad geometry from each source footprint
3. **Synthesize** — union/max extents per pad; flag any spacing violations
   against PCBway DRC limits for manual review
4. **Standardize** — apply Brant's standard layer numbers
5. **Output** — PLX file with three IPC-7351 density variants, ready for
   Pulsonix Library Manager import

---

## Target Format — PLX

### Why PLX
PLX is the correct and only viable target format. The Pulsonix Library
Integration Toolkit (`.fgf`/`.sgf`/`.pgf`) is wizard-driven and parametric —
it generates footprints via preset patterns (DIP, QUAD, RADIAL, AXIAL) and
cannot express arbitrary pad coordinates. It is not suitable for our pipeline.

PLX is used by all commercial footprint providers (SnapEDA, Ultra Librarian,
Accelerated Designs) and imports directly into Pulsonix Library Manager by
drag-and-drop with optional technology file for layer mapping.

Eagle `.lbr` remains a distant fallback only if PLX import proves problematic.

### PLX Format Overview
PLX is a **nested S-expression format** (parenthesis-based, similar to KiCad).
Text-based, human-readable, import-only in Pulsonix (no export).

Header line variants:
- SnapEDA: `"PULSONIX_LIBRARY_ASCII" "SnapEDA ...licence..."`
- Ultra Librarian: `PULSONIX_LIBRARY_ASCII "path/to/file.PLX"` (no outer quotes)

### Top-level structure
```
(library Library
  (padStyleDef ...)     ; named pad styles — defined once, referenced by name
  (textStyleDef ...)    ; named text styles
  (patternDef ...)      ; PCB footprint: pads + layer graphics
  (symbolDef ...)       ; schematic symbol with pins
  (compDef ...)         ; component: links symbol to pattern, pin mapping
)
```

### Units
Declared in `asciiHeader`: `(fileUnits MIL)` or `(fileUnits MM)`.
All coordinates and dimensions use the declared units throughout.

### padStyleDef
Defines a named pad style. Referenced by pads in `patternDef`.
```
(padStyleDef "STYLENAME"
  (holeDiam 39.3701)          ; drill diameter (0 for SMD)
  (isHolePlated False)        ; present only for NPTH mounting holes
  (startRange 1)              ; SMD only: layer span start
  (endRange 2)                ; SMD only: layer span end
  (padShape
    (padShapeType Rect)       ; Rect | Oval | Ellipse | etc.
    (layerNumRef 1)           ; layer number (see layer map below)
    (shapeWidth 59.3701)
    (shapeHeight 59.3701)
  )
  ; repeat padShape for each layer this pad appears on
)
```
Convention: pin 1 uses `Rect` shape, all others use `Oval`.
SMD bottom layer (16) gets `shapeWidth 0` / `shapeHeight 0` (not present on bottom).

### patternDef — pads
```
(patternDef "FOOTPRINT_NAME"
  (originalName "FOOTPRINT_NAME")
  (multiLayer
    (pad
      (padNum 1)
      (padStyleRef STYLENAME)
      (pt x, y)
      (rotation 0)
    )
    ; ... one pad entry per pin
  )
```

### patternDef — layer graphics
```
  (layerContents
    (layerNumRef 30)
    (line (pt x1 y1) (pt x2 y2) (width w))
  )
  (layerContents
    (layerNumRef 30)
    (arc (pt cx cy) (radius r) (startAngle 0) (sweepAngle 360) (width w))
  )
  (layerContents
    (layerNumRef 18)
    (poly (pt x y) (pt x y) ...)
  )
  (layerContents
    (layerNumRef 18)
    (text (pt x y) "string" (textStyleRef "STYLE") (isVisible True))
  )
  (layerContents
    (layerNumRef 18)
    (attr "RefDes" "RefDes"
      (pt x y) (rotation 0) (textStyleRef "STYLENAME") (isVisible True)
    )
  )
```

### Multiple patterns per file
One PLX file may contain multiple `patternDef` blocks — standard practice for
IPC-7351 density variants. One `compDef` references all three via multiple
`attachedPattern` blocks.

### `+vbcrlf` token
Seen in Ultra Librarian output as a line continuation artifact. Parser must
tolerate and ignore it.

### PLX Layer Number Map

Layer numbers are **fixed** across all observed sources (SnapEDA, Ultra
Librarian, Accelerated Designs). We can hardcode the layer map.

| layerNumRef | Confirmed use                          | Brant layer            |
|-------------|----------------------------------------|------------------------|
| 1           | Copper top                             | TOP                    |
| 16          | Copper bottom (zero size on SMD)       | BOTTOM                 |
| 18          | Assembly drawing top / RefDes / attrs  | Assembly Drawing Top   |
| 19          | Pin-1 indicator circle (one occurrence)| Silkscreen Top (tentative) |
| 20          | Solder mask top                        | Solder Mask Top        |
| 21          | Solder mask bottom                     | Solder Mask Bottom     |
| 22          | Paste mask top                         | Paste Mask Top         |
| 30          | Silkscreen top outlines / pin-1 dot    | Silkscreen Top         |
| 39          | Documentation / courtyard / hidden attrs | Documentation        |

> Bottom silkscreen, bottom paste mask, and placement area layer numbers
> not yet observed — all samples are top-side or through-hole components.
> A bottom-side SMD sample would confirm these.

---

## Brant's Standard Layer Set

25 layers total, symmetric top/bottom stack with core and documentation layers.

| #  | Layer Name              | Notes                          |
|----|-------------------------|--------------------------------|
| 1  | Holes Top               | Rare; multi-layer boards only  |
| 2  | Assembly Drawing Top    |                                |
| 3  | Placement Area Top      | Courtyard equivalent           |
| 4  | Pin Names Top           |                                |
| 5  | Wires Top               | Ratsnest/copper pour top       |
| 6  | Silkscreen Top          |                                |
| 7  | Glue Top                | SMT adhesive dots              |
| 8  | Paste Mask Top          | Solder paste apertures         |
| 9  | Finish Top              | Surface finish (HASL/ENIG)     |
| 10 | TOP                     | Primary copper layer           |
| 11 | Solder Mask Top         |                                |
| 12 | Outline                 | Board edge / courtyard outline |
| 13 | CORE                    | Board core (dielectric)        |
| 14 | Solder Mask Bottom      |                                |
| 15 | BOTTOM                  | Primary copper layer           |
| 16 | Finish Bottom           |                                |
| 17 | Paste Mask Bottom       |                                |
| 18 | Glue Bottom             |                                |
| 19 | Silkscreen Bottom       |                                |
| 20 | Wires Bottom            |                                |
| 21 | Pin Names Bottom        |                                |
| 22 | Placement Area Bottom   | Courtyard equivalent           |
| 23 | Assembly Drawing Bottom |                                |
| 24 | Holes Bottom            | Rare; multi-layer boards only  |
| 25 | Documentation           | General notes / fab drawings   |

### Source layer mapping (KiCad / Eagle → PLX layerNumRef)

| Source layer (KiCad / Eagle) | PLX layerNumRef | Brant layer            |
|------------------------------|-----------------|------------------------|
| F.Cu / tCu                   | 1               | TOP                    |
| B.Cu / bCu                   | 16              | BOTTOM                 |
| F.SilkS / tPlace             | 30              | Silkscreen Top         |
| B.SilkS / bPlace             | TBD             | Silkscreen Bottom      |
| F.Paste / tCream             | 22              | Paste Mask Top         |
| B.Paste / bCream             | TBD             | Paste Mask Bottom      |
| F.Mask / tStop               | 20              | Solder Mask Top        |
| B.Mask / bStop               | 21              | Solder Mask Bottom     |
| F.Fab / tDocu                | 18              | Assembly Drawing Top   |
| B.Fab / bDocu                | TBD             | Assembly Drawing Bottom|
| F.Courtyard / tRestrict      | TBD             | Placement Area Top     |
| B.Courtyard / bRestrict      | TBD             | Placement Area Bottom  |
| Edge.Cuts                    | TBD             | Outline                |
| Drills (PTH)                 | 1+16            | TOP + BOTTOM           |
| Drills (NPTH)                | isHolePlated=False | Holes Top/Bottom    |

---

## Footprint Conventions (Brant's Standard)

### Orientation
- Orient packages **horizontally** with **pin 1 in the lower left** wherever possible
- Follows IPC-7351 land pattern standard orientation

### Origin placement
- **SMT parts** — origin at the **centroid of the pad matrix**
- **THT parts** — origin at the **centroid of pin 1**

### Mounting holes
- Most components have no mounting holes
- When present, assign mounting holes the **highest pad numbers**
- Identify by `isHolePlated False` in the padStyleDef
- Examples: TO-220 laid flat, large connectors, modules

### IPC-7351 density variants
- Every output PLX contains three patternDef variants:
  - Nominal (no suffix) — Brant's synthesized "fits all" footprint
  - Maximum (`-M`) — expanded for less precise assembly
  - Least (`-L`) — tightest for high-density assembly
- All three in one PLX file under one library, one compDef referencing all three

### Acceptable automation boundary
- 90% automation is the goal; manual cleanup for edge cases is acceptable
- Mounting hole numbering and unusual orientations are acceptable manual steps

---

## Outstanding Items

- [x] Confirm Pulsonix import format — PLX confirmed; Library Toolkit ruled out
- [x] Understand PLX format — fully documented from 12 sample files, 3 sources
- [x] Document Brant's standard layer set — 25 layers, fully defined
- [x] PLX layer number mapping — fixed numbers confirmed across all sources
- [x] Document Brant's footprint conventions — orientation, origin, mounting holes
- [x] Pulsonix Library Toolkit manual — obtained; confirmed not relevant to PLX pipeline
- [ ] Confirm layer 19 purpose — single occurrence (RY611024); tentatively Silkscreen Top
- [ ] Confirm bottom-side layer numbers — need a bottom-side SMD PLX sample
- [ ] Confirm Brant's thinking on PLX layer numbers in his Pulsonix setup
- [ ] Decide whether synthesis step is fully automated or Brant reviews output
- [ ] Build PLX parser — handle SnapEDA, Ultra Librarian, Accelerated Designs variants
- [ ] Build synthesis engine — max-extent union per pad + PCBway DRC spacing check
- [ ] Build layer standardization pass — remap layerNumRef values to Brant's standard
- [ ] Build PLX generator — output valid PLX with three IPC-7351 density variants
