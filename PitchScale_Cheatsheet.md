# OpenGL Pitch Scale – Debug Cheatsheet
### Based on your `vsd_pitch_scale()` code

---

## 1. How OpenGL Draws Things (The Mental Model)

Think of OpenGL like a **camera on a tripod** looking at a canvas.

- `glVertex2f(x, y)` → place a point at position (x, y) on the canvas
- `glBegin(GL_LINES)` → start drawing lines (connect vertices in pairs)
- `glEnd()` → stop drawing
- `glColor3f(R, G, B)` → set pen color (values 0.0 to 1.0, not 0–255)

```
glColor3f(1, 1, 1)       → White
glColor3f(0, 0, 0)       → Black
glColor3f(0, 0.95, 1)    → Cyan (sky color in your code)
glColor3f(1, 0.59, 0.18) → Orange (ground color in your code)
```

---

## 2. The Matrix Stack – Most Important Concept

This is likely your bug source. Understand this fully.

### What is it?
OpenGL has a **coordinate system (matrix)**. When you draw, all points are placed in the current matrix's coordinate frame.

### Push / Pop
```cpp
glPushMatrix();   // SAVE current coordinate state (like bookmark)
    // anything drawn here is in the MODIFIED coordinate frame
glPopMatrix();    // RESTORE to saved state (like undo)
```

### Translate
```cpp
glTranslatef(x, y, z);
// Moves the ORIGIN of the coordinate system
// glTranslatef(0, 1.25, 0) → moves origin UP by 1.25 units
// After this, drawing at (0,0) appears at screen position (0, 1.25)
```

### Rotate
```cpp
glRotatef(angle, 0, 0, 1);
// Rotates the coordinate system around Z-axis by 'angle' degrees
// THIS IS THE DANGEROUS ONE for your bug
```

### ⚠️ Critical Rule
**Order matters. Translate then Rotate ≠ Rotate then Translate.**

And most importantly:
> If you draw `glVertex2f(0, level)` INSIDE a `glRotatef(angle)` block,
> the point does NOT go straight up by `level`.
> It goes up-and-sideways depending on `angle`.

---

## 3. Your Code's Structure – Mapped Out

```
glPushMatrix()
    glTranslatef(0, 1.25, 0)       ← move origin to center of attitude indicator
    glRotatef(rotate_PITCH, 0,0,1) ← ROTATE the whole frame by rotate_PITCH degrees

    vsd_circle2(... PITCH_mfd11 ...)  ← horizon fill arcs (blue sky / orange ground)

glPopMatrix()                      ← back to original frame

glPushMatrix()
    glTranslatef(0, 0, 0)          ← no movement (stays at screen center)

    // Flight symbol drawn here (fixed, doesn't move)

glPopMatrix()

glPushMatrix()
    glTranslatef(0, 1.25, 0)       ← move origin again
    glRotatef(0, 0, 0, 1)          ← zero rotation (this block is fine)

    // Pitch ladder lines drawn here using 'level'
    glVertex2f(-1.38, 2.74 + level)  ← +20° line
    glVertex2f(-1.38, 2.0  + level)  ← +10° line
    glVertex2f(-1.38, 0.47 + level)  ← -10° line
    glVertex2f(-1.38, -0.27 + level) ← -20° line

glPopMatrix()
```

---

## 4. What `level` Does

```cpp
level = (isis_pitch * 0.7) / 45.0;
```

`level` is a **vertical offset in OpenGL units** that shifts all pitch ladder lines up or down.

### Ladder line positions (from code):
| Pitch | Y position in code | Spacing |
|-------|-------------------|---------|
| +20°  | 2.74 + level      | —       |
| +15°  | 2.37 + level      | 0.37    |
| +10°  | 2.00 + level      | 0.37    |
| +5°   | 1.63 + level      | 0.37    |
| 0°    | 1.25 + level      | 0.38    |
| -5°   | 0.84 + level      | 0.41    |
| -10°  | 0.47 + level      | 0.37    |
| -15°  | 0.10 + level      | 0.37    |
| -20°  | -0.27 + level     | 0.37    |

So **every 5° = ~0.37 GL units** of vertical space.

For `level` to move lines correctly: **1° of APITCH should shift level by 0.37/5 = 0.074 GL units.**

### Correct formula (if APITCH is in degrees):
```cpp
level = APITCH * 0.074;
// or equivalently:
level = (APITCH * 0.7) / 45.0;  // same thing, already in your code
```
**This part of the code is already correct.**

---

## 5. What `PITCH_mfd11` Does

```cpp
PITCH_mfd11 = isis_pitch * (-1);
```

This is used in `vsd_circle2` to rotate the **horizon line arcs** (blue sky / orange ground fill):

```cpp
vsd_circle2(0, 0, 2.74, 180,   0 - PITCH_mfd11/divider,  181 + PITCH_mfd11/divider);
vsd_circle2(0, 0, 2.74, 180, 180 + PITCH_mfd11/divider,  362 - PITCH_mfd11/divider);
```

These are **angle arguments** to the circle function.  
`PITCH_mfd11 / divider` is being treated as **degrees of arc**.

`divider = 1.625` is fixed. So `PITCH_mfd11` directly controls how much arc shifts.

If `PITCH_mfd11` feeds into sin/cos inside `vsd_circle2` → non-linear output.

---

## 6. What `vsd_circle2` Likely Does Internally

You don't have the source but based on usage:
```cpp
vsd_circle2(cx, cy, radius, steps, start_angle, end_angle);
```
It draws a filled arc from `start_angle` to `end_angle`.

Internally it almost certainly does:
```cpp
for(angle = start_angle; angle <= end_angle; angle += step) {
    x = cx + radius * cos(angle * PI/180);
    y = cy + radius * sin(angle * PI/180);
}
```

This means `PITCH_mfd11/divider` is used as **angle in degrees** inside a sin/cos.
**This is non-linear by nature** — small angle changes at 0° have different effect than at 45°.

---

## 7. The `if(level ...)` Blocks – Visibility Logic

```cpp
if (level <= -0.7)
    if (level < -2.8f) { level = 1.0f; }   // wraparound (possibly buggy)

if (level >= -0.7) {
    if (!(level >= 0.5)) { /* draw +20° line */ }
    if (!(level >= 0.9)) { /* draw +15°, +10°, +5° lines */ }
    /* always draw -5°, -10° lines */
    if (level >= -0.9)   { /* draw -15° line */ }
    if (level >= -0.8)   { /* draw -20° line */ }
}
```

**Translation:** Lines only draw when they're within the visible window.
As `level` changes, lines scroll out of view at edges. This is correct behavior.

---

## 8. Debug Steps – Do These In Order

### Step 1 – Isolate `rotate_PITCH`
```cpp
// Temporarily hardcode:
glRotatef(0.0f, 0.0f, 0.0f, 1.0f);  // force no rotation
```
Vary APITCH. If ladder lines now move linearly → `rotate_PITCH` was the bug.

### Step 2 – Isolate `PITCH_mfd11`
```cpp
// Temporarily hardcode:
PITCH_mfd11 = 0;
```
Vary APITCH. Check if horizon fill (blue/orange) stays fixed but ladder moves correctly.

### Step 3 – Check which PushMatrix block the ladder lines are in
Look for the `glVertex2f(-1.38, 2.74 + level)` lines.
- If they're **inside** a `glRotatef(rotate_PITCH)` block → they're being drawn in rotated frame → bug
- If they're **outside** (in a block with `glRotatef(0.0f...)`) → frame is clean → look elsewhere

### Step 4 – Open `vsd_circle2` definition
Search for it in the project. Look for sin/cos inside.
If `PITCH_mfd11/divider` goes into sin/cos → horizon arc is non-linear. That's expected and fine.
The ladder lines (using `level`) should NOT go through any trig.

---

## 9. Quick Syntax Reference

```cpp
glPushMatrix()          // save coordinate state
glPopMatrix()           // restore coordinate state
glTranslatef(x, y, z)  // shift origin
glRotatef(deg, x,y,z)  // rotate (z=1 means spin on screen)
glBegin(GL_LINES)       // start drawing lines
glVertex2f(x, y)        // place a point
glEnd()                 // stop drawing
glColor3f(r, g, b)      // set color (0.0–1.0)
glLineWidth(n)          // set line thickness in pixels
```

---

## 10. The One-Line Summary of Your Bug

> The pitch ladder `level` offset is **linear and correct**.
> Something between `APITCH` and what gets **displayed** is passing through a **trig function or a rotated coordinate frame**, making it non-linear.
> Find which `glRotatef` block the ladder lines live inside, and whether `rotate_PITCH` is non-zero when it shouldn't be.
