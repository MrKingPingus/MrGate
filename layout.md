# layout.md — Mr. Gate Phase 4 UI Layout Specification

All design decisions for the `@gfx` implementation are made here. Implementation should be pure translation — no open choices remain.

---

## Window

Design size: **700 × 420 px**. Set in `@init`:

```
gfx_w = 700;
gfx_h = 420;
```

In `@gfx`, compute scale factors at the top of every frame before any drawing:

```
ui_sx = gfx_w / 700;
ui_sy = gfx_h / 420;
```

All pixel values in this document are **design pixels**. Multiply x-coordinates by `ui_sx`, y-coordinates by `ui_sy` before passing to any `gfx_*` call.

---

## Colors

Set `gfx_r`, `gfx_g`, `gfx_b` (and `gfx_a = 1`) before drawing. All values are 0..1 floats.

| Name | Usage | R | G | B |
|------|-------|-----|-----|-----|
| COL_BG | Window background | 0.102 | 0.102 | 0.118 |
| COL_TOPBAR | Top bar fill | 0.133 | 0.133 | 0.149 |
| COL_KNOB_BODY | Knob fill circle | 0.200 | 0.200 | 0.220 |
| COL_KNOB_BORDER | Knob circle border | 0.282 | 0.282 | 0.314 |
| COL_KNOB_TRACK | Dim full-sweep arc | 0.228 | 0.228 | 0.259 |
| COL_KNOB_FILL | Active fill arc | 0.228 | 0.565 | 0.878 |
| COL_KNOB_DOT | Indicator dot | 0.722 | 0.847 | 0.973 |
| COL_BTN_ACTIVE | Active button fill | 0.165 | 0.424 | 0.722 |
| COL_BTN_INACTIVE | Inactive button fill | 0.157 | 0.157 | 0.180 |
| COL_BTN_BORDER | Button border | 0.267 | 0.267 | 0.314 |
| COL_TEXT | Primary text | 0.847 | 0.847 | 0.863 |
| COL_TEXT_DIM | Dim / label text | 0.533 | 0.533 | 0.565 |
| COL_DISP_BG | Display area fill | 0.055 | 0.055 | 0.071 |
| COL_DISP_BORDER | Display area border | 0.228 | 0.228 | 0.259 |

---

## Fonts

Set once in `@init`:

```
gfx_setfont(1, "Arial", 11);   // knob labels, button text
gfx_setfont(2, "Arial", 10);   // value readouts
```

Use `gfx_setfont(1)` / `gfx_setfont(2)` in `@gfx` to switch between them.

---

## Top Bar (y = 0..40)

Draw a filled rect: `x=0, y=0, w=gfx_w, h=40*ui_sy`. Color: COL_TOPBAR.

### Mode Buttons

All at design `y=8, h=24`. Rounded corners `r=3`. Border: 1 px COL_BTN_BORDER.  
Active (matching `slider1`): COL_BTN_ACTIVE fill, COL_TEXT label.  
Inactive: COL_BTN_INACTIVE fill, COL_TEXT_DIM label. Font 1, centered inside button.

| Button | Design x | Design w | Label |
|--------|----------|----------|-------|
| Gate (mode 0) | 10 | 62 | "Gate" |
| Downward (mode 1) | 78 | 86 | "Downward" |
| Upward (mode 2) | 170 | 72 | "Upward" |

### Style Dropdown

- Label `"Style:"`: design `x=450, y=15`. Font 1. Color: COL_TEXT_DIM.
- Box: design `x=490, y=8, w=200, h=24`. Fill COL_BTN_INACTIVE, border COL_BTN_BORDER, corner r=3.
- Text `"General"` centered in box. Font 1. Color: COL_TEXT.

---

## Center Display Placeholder

This rect is where Phases 5+6 draw. In Phase 4, draw it as a solid fill.

`x=130, y=48, w=440, h=312` (design px).  
Fill: COL_DISP_BG. Border: 1 px COL_DISP_BORDER.

When `ui_disp_enabled = 0` (Phase 4d): skip drawing this rect entirely.

---

## Knob Layout

```
KNOB_R = 22       // visual radius, design px
KNOB_L_X = 65     // left column center x
KNOB_R_X = 635    // right column center x
```

Y centers (design px):

| Row | Design Y | Left knob | Right knob |
|-----|----------|-----------|------------|
| 0 | 87 | Threshold (slider3) | Attack (slider7) |
| 1 | 165 | Ratio (slider4) | Release (slider8) |
| 2 | 243 | Range (slider5) | Hold (slider9) |
| 3 | 321 | Knee (slider6) | Lookahead (slider10) |

---

## Knob Drawing: `function ui_draw_knob(cx, cy, r, t)`

Parameters: `cx`, `cy` are **design-pixel** centers; `r` is `KNOB_R`; `t` is normalized value 0..1.  
Scale to screen inside the function: `scx = cx*ui_sx; scy = cy*ui_sy; sr = r*min(ui_sx,ui_sy)`.

Draw in this order:

1. **Body:** `gfx_circle(scx, scy, sr, 1)` filled with COL_KNOB_BODY. Then `gfx_circle(scx, scy, sr, 0)` outline with COL_KNOB_BORDER.

2. **Track arc:** `gfx_arc(scx, scy, (sr+5), ANG_MIN, ANG_MIN+ANG_SWEEP, 1)` with COL_KNOB_TRACK. (The `1` enables antialiasing if supported.)

3. **Fill arc:** `gfx_arc(scx, scy, (sr+5), ANG_MIN, ANG_MIN + t*ANG_SWEEP, 1)` with COL_KNOB_FILL. Skip if `t <= 0`.

4. **Indicator dot:** filled circle of radius 3 at:
   ```
   ix = scx + (sr - 5) * cos(ANG_MIN + t * ANG_SWEEP)
   iy = scy + (sr - 5) * sin(ANG_MIN + t * ANG_SWEEP)
   gfx_circle(ix, iy, 3*min(ui_sx,ui_sy), 1)   // COL_KNOB_DOT
   ```

### Angle constants

```
ANG_MIN   = 2 * $pi / 3;    // 120° — 7 o'clock (lower-left)
ANG_SWEEP = 5 * $pi / 3;    // 300° sweep clockwise through 12 o'clock to 5 o'clock
```

Verification: at `t=0` the dot is lower-left; at `t=0.5` it is at 12 o'clock (top); at `t=1` it is lower-right. If the arcs draw the wrong direction in Reaper, swap `ang1` and `ang2` in the `gfx_arc` calls.

---

## Knob Labels (Phase 4a)

Font 1. Color: COL_TEXT_DIM.  
Centered below knob: measure string width with `gfx_measurestr`, then:

```
gfx_x = scx - str_w/2;
gfx_y = (cy + r + 5) * ui_sy;
gfx_drawstr("LABEL");
```

Label strings (all-caps):

| Slider | Label |
|--------|-------|
| slider3 | `"THRESHOLD"` |
| slider4 | `"RATIO"` |
| slider5 | `"RANGE"` |
| slider6 | `"KNEE"` |
| slider7 | `"ATTACK"` |
| slider8 | `"RELEASE"` |
| slider9 | `"HOLD"` |
| slider10 | `"LOOKAHEAD"` |

---

## Value Readouts (Phase 4d)

Font 2. Color: COL_TEXT.  
Same centering as labels but at `gfx_y = (cy + r + 19) * ui_sy`.

Format strings (use `sprintf(#str, fmt, val)` then `gfx_drawstr(#str)`):

| Slider | Format |
|--------|--------|
| slider3 | `"%.1f dB"` |
| slider4 | `"%.1f:1"` |
| slider5 | `"%.1f dB"` |
| slider6 | `"%.1f dB"` |
| slider7 | `"%.2f ms"` if value < 10, else `"%.1f ms"` |
| slider8 | `"%.0f ms"` |
| slider9 | `"%.0f ms"` |
| slider10 | `"%.1f ms"` |

---

## Slider Normalization

Convert slider value → t (0..1) for knob drawing and drag math.

| Index | Slider | t formula | min | max |
|-------|--------|-----------|-----|-----|
| 0 | slider3 | `(slider3 + 80) / 80` | -80 | 0 |
| 1 | slider4 | `(slider4 - 1) / 29` | 1 | 30 |
| 2 | slider5 | `slider5 / 60` | 0 | 60 |
| 3 | slider6 | `slider6 / 24` | 0 | 24 |
| 4 | slider7 | `(slider7 - 0.05) / 99.95` | 0.05 | 100 |
| 5 | slider8 | `(slider8 - 5) / 4995` | 5 | 5000 |
| 6 | slider9 | `slider9 / 250` | 0 | 250 |
| 7 | slider10 | `slider10 / 10` | 0 | 10 |

Clamp t to [0, 1] after computing. Inverse: `slider_value = min_val + t * (max_val - min_val)`, then clamp to [min_val, max_val].

Store as arrays in `@init` at memory addresses `SLIDER_MIN_TABLE = 6230` and `SLIDER_MAX_TABLE = 6240`:

```
SLIDER_MIN_TABLE = 6230;
SLIDER_MAX_TABLE = 6240;
SLIDER_MIN_TABLE[0] = -80; SLIDER_MAX_TABLE[0] = 0;
SLIDER_MIN_TABLE[1] = 1;   SLIDER_MAX_TABLE[1] = 30;
SLIDER_MIN_TABLE[2] = 0;   SLIDER_MAX_TABLE[2] = 60;
SLIDER_MIN_TABLE[3] = 0;   SLIDER_MAX_TABLE[3] = 24;
SLIDER_MIN_TABLE[4] = 0.05; SLIDER_MAX_TABLE[4] = 100;
SLIDER_MIN_TABLE[5] = 5;   SLIDER_MAX_TABLE[5] = 5000;
SLIDER_MIN_TABLE[6] = 0;   SLIDER_MAX_TABLE[6] = 250;
SLIDER_MIN_TABLE[7] = 0;   SLIDER_MAX_TABLE[7] = 10;
```

---

## Defaults Table (for ctrl+click reset)

Memory address `DEFAULT_TABLE = 6200`. 24 entries: 3 modes × 8 params.  
Index: `DEFAULT_TABLE[mode_id * 8 + knob_index]` where `knob_index` maps slider3→0 .. slider10→7.

```
DEFAULT_TABLE = 6200;
// Gate (mode 0):
DEFAULT_TABLE[0]=-40; DEFAULT_TABLE[1]=10;  DEFAULT_TABLE[2]=40;
DEFAULT_TABLE[3]=3;   DEFAULT_TABLE[4]=1;   DEFAULT_TABLE[5]=100;
DEFAULT_TABLE[6]=10;  DEFAULT_TABLE[7]=0;
// Downward (mode 1):
DEFAULT_TABLE[8]=-50;  DEFAULT_TABLE[9]=2;   DEFAULT_TABLE[10]=12;
DEFAULT_TABLE[11]=9;   DEFAULT_TABLE[12]=10; DEFAULT_TABLE[13]=250;
DEFAULT_TABLE[14]=0;   DEFAULT_TABLE[15]=0;
// Upward (mode 2):
DEFAULT_TABLE[16]=-20; DEFAULT_TABLE[17]=2;  DEFAULT_TABLE[18]=6;
DEFAULT_TABLE[19]=6;   DEFAULT_TABLE[20]=5;  DEFAULT_TABLE[21]=200;
DEFAULT_TABLE[22]=0;   DEFAULT_TABLE[23]=0;
```

Add `DEFAULT_TABLE`, `SLIDER_MIN_TABLE`, and `SLIDER_MAX_TABLE` to the memory layout comment in `@init`.

---

## Bottom Bar (design y = 362..420)

### Display Toggle Button

`x=10, y=370, w=120, h=30`. Corner r=3. Font 1.

- When `ui_disp_enabled = 1`: fill COL_BTN_ACTIVE, text `"Display ON"`, color COL_TEXT.
- When `ui_disp_enabled = 0`: fill COL_BTN_INACTIVE, text `"Display OFF"`, color COL_TEXT_DIM.
- Border always: COL_BTN_BORDER.

### Gain Reduction Placeholder

`x=545, y=370, w=145, h=30`. Fill COL_BTN_INACTIVE, border COL_BTN_BORDER, corner r=3.  
Text `"GR: --"` centered. Font 1. Color COL_TEXT_DIM.  
(Live GR display is Phase 6.)

---

## Mouse Interaction (Phase 4b)

### State variables — add to `@init`

```
ui_drag_knob    = -1;   // 0..7 = knob being dragged, -1 = none
ui_drag_start_y = 0;
ui_drag_start_v = 0;    // slider value (not normalized) at drag start
ui_prev_lmb     = 0;    // LMB state last frame (1 = was down)
ui_last_click_t = 0;    // time_precise() of last LMB-down
ui_disp_enabled = 1;
```

### Knob hit test

```
// For each knob k with design center (cx, cy):
hit_r = KNOB_R * 1.5 * min(ui_sx, ui_sy);
dx = mouse_x - cx * ui_sx;
dy = mouse_y - cy * ui_sy;
hit = (dx*dx + dy*dy) <= hit_r*hit_r;
```

The 1.5× factor gives a slightly larger target than the visible knob.

### LMB down detection

```
lmb_now = mouse_cap & 1;
lmb_just_down = lmb_now && !ui_prev_lmb;
```

### Drag-to-adjust

On `lmb_just_down` over knob k:
```
// Double-click check
time_precise() - ui_last_click_t < 0.4 ? (
    // reset — see below
) : (
    ui_drag_knob    = k;
    ui_drag_start_y = mouse_y;
    ui_drag_start_v = slider_value_of(k);   // read slider3+k
);
ui_last_click_t = time_precise();
```

Each frame while `lmb_now && ui_drag_knob >= 0`:
```
k = ui_drag_knob;
t_start = (ui_drag_start_v - SLIDER_MIN_TABLE[k]) / (SLIDER_MAX_TABLE[k] - SLIDER_MIN_TABLE[k]);
delta_y  = ui_drag_start_y - mouse_y;           // positive = dragged up = increase
sensitivity = (mouse_cap & 8) ? 0.1 : 1.0;     // bit 3 = Shift
t_new = max(0, min(1, t_start + delta_y * sensitivity / 200));
new_val = SLIDER_MIN_TABLE[k] + t_new * (SLIDER_MAX_TABLE[k] - SLIDER_MIN_TABLE[k]);
// assign to slider3+k, call sliderchange()
```

On LMB release (`!lmb_now && ui_drag_knob >= 0`): `ui_drag_knob = -1`.

### Ctrl+click or double-click reset

```
reset_val = DEFAULT_TABLE[mode_id * 8 + k];
// assign to slider3+k, call sliderchange()
```

Ctrl is bit 4 of `mouse_cap`: `mouse_cap & 16`.

### Scroll-wheel nudge

Each frame, if `mouse_wheel != 0` and cursor is over knob k:
```
t_cur = (current_val - SLIDER_MIN_TABLE[k]) / (SLIDER_MAX_TABLE[k] - SLIDER_MIN_TABLE[k]);
t_new = max(0, min(1, t_cur + mouse_wheel * 0.005));
new_val = SLIDER_MIN_TABLE[k] + t_new * (SLIDER_MAX_TABLE[k] - SLIDER_MIN_TABLE[k]);
// assign, call sliderchange()
mouse_wheel = 0;
```

### Mode button and display toggle clicks

On `lmb_just_down`, check rects before checking knobs (buttons take priority):
- Mode button hit: `slider1 = mode_index; sliderchange();`
- Display toggle hit: `ui_disp_enabled = 1 - ui_disp_enabled;`
- Style dropdown hit: no-op.
- If none of the above hit: check knobs.

---

## `@gfx` Frame Order

Run in this sequence every frame:

1. `ui_sx = gfx_w/700; ui_sy = gfx_h/420;`
2. Clear background (COL_BG, full window).
3. Draw top bar: background rect → mode buttons → style dropdown.
4. If `ui_disp_enabled`: draw center display placeholder rect.
5. Draw 8 knobs (left column then right column), each with label (Phase 4a) and value readout (Phase 4d).
6. Draw bottom bar: display toggle button, GR placeholder.
7. Process mouse: check buttons first (on `lmb_just_down`), then check drag, then scroll wheel.
8. `ui_prev_lmb = lmb_now;`
