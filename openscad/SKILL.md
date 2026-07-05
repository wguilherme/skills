---
name: openscad
description: Use this skill when working with OpenSCAD files — creating parametric 3D models, FDM-ready parts, modular design systems, or automating exports via CLI. Triggers on requests like "create a scad file", "parametric 3D part", "OpenSCAD module", "export STL", "openscad CLI", "FDM design", or any mention of .scad files.
---

# OpenSCAD Skill

Guide for writing parametric OpenSCAD models — with focus on FDM-printable parts and modular design systems.

---

## File Structure Pattern

Every `.scad` file follows this order:

```openscad
// 1. Parameters (user-facing, with Customizer annotations)
atom_size = 20; // [5:1:100]
hole_d    = 10; // [0.5:0.1:15]
$fn       = 32;

// 2. Derived / calculated values
relief_d = hole_d + 4;
safe_r   = min(border_radius, atom_size / 2 - 0.01);

// 3. Modules
module my_part() { ... }

// 4. Render call (always last)
part_color = [0.35, 0.38, 0.42];
color(part_color) my_part();
```

`part_color` declared before render allows `-D "part_color=[r,g,b]"` CLI override without breaking the file when opened in the GUI.

---

## Parametric System (Customizer)

Inline comments after parameter declarations drive the OpenSCAD Customizer and MakerWorld PMM:

```openscad
size          = 20;    // [5:1:100]      — slider: min:step:max
mode          = "a";   // ["a", "b", "c"] — dropdown
border_radius = 0;     // [0:0.1:5]
```

---

## Key Modules

### Rounded cube via Minkowski

```openscad
module rounded_cube(size, r) {
    if (r <= 0) {
        cube([size, size, size], center = true);
    } else {
        inner = size - 2 * r;
        minkowski() {
            cube([inner, inner, inner], center = true);
            sphere(r = r);
        }
    }
}
```

`r` must be `< size/2`. Always clamp: `safe_r = min(r, size/2 - 0.01)`.

### Hole + countersink on both faces

```openscad
module through_hole(thickness, hole_d, relief_d, relief_depth) {
    cylinder(d = hole_d, h = thickness + 1, center = true);
    translate([0, 0,  thickness / 2 - relief_depth])
        cylinder(d = relief_d, h = relief_depth + 0.01);
    translate([0, 0, -thickness / 2])
        cylinder(d = relief_d, h = relief_depth + 0.01);
}
```

The `+0.01` prevents z-fighting on coplanar faces.

---

## Axis Rotation Quirk (Face Placement)

Placing cylinders on the ±Y faces of a centered body:

```openscad
// Face +Y
translate([0, size/2, 0]) rotate([90, 0, 0])
    cylinder(d = hole_d, h = relief_depth + 0.01);

// Face -Y  ← rotate NEGATIVE, translate NEGATIVE
translate([0, -size/2, 0]) rotate([-90, 0, 0])
    cylinder(d = hole_d, h = relief_depth + 0.01);
```

`rotate([90,0,0])` points the cylinder toward −Y. Use `rotate([-90,0,0])` to point toward +Y. This is counterintuitive — always verify with F5 preview.

---

## FDM Design Rules

| Rule | Value | Reason |
| --- | --- | --- |
| Press-fit tolerance | `0.2mm` subtracted from pin/flange | Most FDM printers. Adjust ±0.05 per printer. |
| Minimum wall | `1.2mm` (= 3× 0.4mm nozzle) | Below this walls may not print solid |
| Countersink min depth | `relief_depth ≥ 1.5mm` | Less won't seat connector flange flat |
| Plate min thickness | `(relief_depth × 2) + 1mm` | Prevents relief overlap between faces |
| Horizontal hole | Use teardrop shape | Avoids supports on horizontal bores |
| 45° chamfer | Replace 90° overhangs | Eliminates supports on edges |

### Safe guards pattern

```openscad
safe_relief = min(relief_depth, (thickness / 2) - 0.5);
safe_radius = min(border_radius, thickness / 2 - 0.01, cell_size / 2 - 0.01);
```

Always guard derived values — prevents invalid geometry when user inputs extreme parameters.

---

## Modular System Parameters

When building a system where parts must interconnect, declare a shared parameter contract:

```openscad
// All parts must use these same values to guarantee hole alignment
cell_size     = 20;   // base grid unit
hole_d        = 10;   // through-hole diameter
relief_depth  = 2;    // countersink depth
relief_margin = 2;    // countersink margin (relief_d = hole_d + margin*2)
tolerance     = 0.2;  // FDM press-fit tolerance
$fn           = 32;
```

Compatibility rule: **if `cell_size`, `hole_d`, and `relief_*` are equal across parts, holes align on any face combination.**

---

## Preview — Software vs CLI

### Via GUI (OpenSCAD app)

- `F5` — preview rápido (CSG, usa `color()`, não valida manifold)
- `F6` — render completo (CGAL, ignora `color()`, mais lento, correto para export)
- Painel lateral: Customizer — edita parâmetros em tempo real se o arquivo usa anotações `// [min:step:max]`

### Via CLI

Preferir CLI em projetos com automação ou CI. `--preview` equivale ao F5 — respeita `color()`:

```bash
openscad \
  --export-format png \
  --colorscheme Monotone \
  --imgsize 800,600 \
  --autocenter --viewall \
  --projection p \
  --preview \
  -o output.png \
  file.scad
```

`--render` (sem `--preview`) equivale ao F6 — ignora `color()`, mais lento, geometria validada.

### Instalar OpenSCAD

**Download direto:** https://openscad.org/downloads.html

**Via asdf** — recomendado se o projeto tem `.tool-versions` ou se o usuário já usa asdf:

```bash
asdf plugin add openscad https://github.com/gabrielelana/asdf-openscad
asdf install openscad 2021.01   # ou a versão em .tool-versions
asdf set openscad 2021.01       # define para o projeto
```

Se o projeto tem `.tool-versions` com `openscad X.Y`, basta:

```bash
asdf install   # instala tudo que está em .tool-versions
```

> Tag format do plugin: `openscad-YYYY.MM` → versão declarada sem prefixo: `2021.01`

---

## CLI Usage

### Export STL

```bash
openscad --export-format binstl -o output.stl file.scad
```

### Export PNG preview

```bash
openscad \
  --export-format png \
  --colorscheme Monotone \
  --imgsize 800,600 \
  --autocenter --viewall \
  --projection p \
  --preview \
  -D "part_color=[0.35, 0.38, 0.42]" \
  -o output.png \
  file.scad
```

### Inject variable at render time

```bash
openscad -D "atom_size=30" -D "part_color=[0.8,0.1,0.1]" -o out.stl file.scad
```

`-D` overrides any variable declared in the file. Values are raw OpenSCAD expressions.

### Colorschemes

```
Cornfield | Metallic | Sunset | Starnight | BeforeDawn | Nature
DeepOcean | Solarized | Tomorrow | Tomorrow Night | Monotone
```

- `Monotone` — grayscale, no color artifacts, best for documentation renders
- `Metallic` — dark body but pink/magenta on interior cut faces (avoid for docs)

`color()` in `.scad` only affects `--preview` mode. `--render` (F6) ignores it — use `--preview` flag in CLI for colored exports.

### Post-process: transparent background

```bash
# Sample background color from top-left pixel, then remove it
BG=$(magick render.png -format "%[pixel:u.p{0,0}]" info:)
magick render.png -fuzz 2% -transparent "$BG" output.png
```

### Post-process: gradient background

```bash
magick \
  \( -size 800x600 gradient:"rgb(208,208,208)-rgb(49,49,49)" \) \
  \( render.png -fuzz 2% -transparent "$BG" \) \
  -composite output.png
```

---

## Makefile Integration

Dynamic part list from filesystem — no hardcoded list:

```makefile
PARTS := $(basename $(notdir $(wildcard impl/openscad/*.scad)))
PART_COLOR := [0.35, 0.38, 0.42]

preview:
	@for part in $(PARTS); do \
		openscad --export-format png --preview \
			-D "part_color=$(PART_COLOR)" \
			-o renders/img/$$part.png impl/openscad/$$part.scad; \
	done

build:
	@for part in $(PARTS); do \
		openscad --export-format binstl \
			-o renders/stl/$$part.stl impl/openscad/$$part.scad; \
	done
```

Override color at call time: `make preview PART_COLOR="[0.8,0.1,0.1]"`

---

## asdf Install

```bash
# .tool-versions
openscad 2021.01

asdf plugin add openscad
asdf install
```

Plugin: https://github.com/gabrielelana/asdf-openscad (non-standard tag format: `openscad-YYYY.MM`)

---

## Reference

- Language reference: https://openscad.org/documentation.html
- Cheatsheet: https://openscad.org/cheatsheet/
- Colorschemes: https://en.wikibooks.org/wiki/OpenSCAD_User_Manual/Preferences#Color_Schemes
- MakerWorld PMM (parametric): https://makerworld.com/en/makerlab/parameterizedModel
- FusionBrick — real modular system built with this skill: https://github.com/wguilherme/FusionBrick
