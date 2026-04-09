# CrossPoint PXC Converter

**[crosspoint-pxc-converter.pages.dev](https://crosspoint-pxc-converter.pages.dev)**

A browser-based image converter for creating sleep screen wallpapers for the [CrossPoint](https://github.com/crosspoint-reader/crosspoint-reader) open-source e-reader firmware running on the XTEink X4 device.

Converts any image (PNG, JPG, WebP, BMP, GIF) into the **PXC format** — a compact 2-bit-per-pixel bitmap used by CrossPoint for sleep screens and wallpapers. No upload, no server — all processing happens locally in your browser.

---

## PXC Format

PXC is CrossPoint's native wallpaper format. It stores a 480×800 image at 2 bits per pixel, giving four gray levels that map directly to the e-ink display's physical states:

| Value | Display level  | sRGB  |
|-------|---------------|-------|
| `0`   | Black          | 0     |
| `1`   | Dark grey      | 85    |
| `2`   | Light grey     | 170   |
| `3`   | White          | 255   |

**File layout:**

```
Bytes 0–1   Width  (uint16 LE)
Bytes 2–3   Height (uint16 LE)
Bytes 4+    Pixel data — 2 bits per pixel, MSB first, row-major
            Each byte holds 4 pixels: [p0 p1 p2 p3] in bits [7:6 5:4 3:2 1:0]
```

One full 480×800 wallpaper = 4 + (480/4 × 800) = **96,004 bytes**.

---

## Where to put the file

| Path on SD card        | Effect                                      |
|------------------------|---------------------------------------------|
| `/sleep.pxc`           | Single static sleep screen                  |
| `/.sleep/name.pxc`     | Rotated wallpaper pool (multiple files)     |

CrossPoint picks from `/.sleep/` at random if multiple files are present.

---

## Features

### Image adjustment
- **Crop mode** — drag to reposition the crop window over the source image; snap guides appear when the image edge aligns with the crop boundary
- **Fit mode** — letterbox with configurable alignment (top / center / bottom × left / center / right)
- **Rotation** — 90° CW / CCW steps
- **Contrast** — linear-space contrast adjustment with a ±100 slider, pivot at midpoint
- **Invert** — invert luminance in linear space before dithering

### Dithering algorithms

All error-diffusion algorithms operate in **linear light** — luminance is extracted via BT.601 and linearised from sRGB before dithering, so quantisation errors are perceptually uniform rather than biased toward shadows or highlights.

| Algorithm | Notes |
|-----------|-------|
| **Floyd-Steinberg** | Classic 4-neighbour kernel (`7/5/3/1 ÷ 16`). Good general balance. |
| **Atkinson** | Distributes only 6/8 of the error across 6 neighbours. Preserves highlights, produces lighter images — popular for manga. |
| **Jarvis (JJN)** | 3-row, 12-neighbour kernel (`÷ 48`). Spreads error wider, reduces banding at the cost of softer edges. |
| **Stucki** | JJN-family with adjusted weights (`÷ 42`). Slightly sharper than Jarvis. |
| **Burkes** | 2-row version of Stucki (`÷ 32`). Faster and still sharp. |
| **Bayer** | 4×4 ordered (threshold) dithering. No error bleeding, produces a regular crosshatch pattern. Good for flat illustrations with hard edges. |
| **Zhou-Fang** | JJN kernel with **serpentine scanning** — rows alternate left→right and right→left. Eliminates the directional "worm" artifacts that single-direction JJN produces, giving more neutral, grain-like noise. Generally the best choice for photographic content on e-ink. |

### Preview
- Live 480×800 preview updates on every change
- 3.5× zoom loupe follows the cursor for pixel-level inspection

---

## Technical notes

**Linear-space processing.** The converter linearises each pixel from sRGB before dithering and maps quantised levels back to sRGB only for the preview display. This means quantisation thresholds are placed at the perceptual midpoints between gray levels, not at the naive sRGB midpoints (which would push too many pixels toward black).

The four linear reference values are `[0, 23, 103, 255]`, which are the linearised equivalents of sRGB `[0, 85, 170, 255]`. Error diffusion computes error against these linear targets.

**Zhou-Fang serpentine detail.** On each even row the kernel propagates forward (right); on each odd row it propagates backward (left). The two lower kernel rows are symmetric around the current pixel so they are unaffected by direction. This halves the horizontal bias accumulated over a full image.

---

## Related

- [CrossPoint firmware](https://github.com/crosspoint-reader/crosspoint-reader) — the e-reader firmware this tool targets
- [xtcjs](https://xtcjs.pages.dev) — CBZ/PDF to XTC comic converter for CrossPoint
