# Bloom Studio

A single-page 3D web app. You arrange a bouquet in a pot, then tap it and the flowers fly into place to form a **scannable QR code** for any link. Tap the QR code and it turns back into the bouquet.

It is inspired by Ann Nguyen's [Bloom QR](https://www.bubbbly.com/bloom) and extends it with more flowers, pots, painted designs, colors, a quantity control, butterflies, and download and share options.

- **Live page:** https://claude.ai/artifact/BnvsJ5Yfji9HpZfGx5FEpd
- **Source:** one file, `bloom-studio.html` (about 60 KB, no build step)

---

## 1. Features

| Area | What you can do |
|---|---|
| **Flowers** | Rose, Tulip, Daisy, Sunflower, Lily, Peony, Lotus, or **Mixed** (a random assortment) |
| **Flower colors** | Original (each flower's natural colors), Blush, Marigold, Sunbeam, Poppy, Azure dusk, **Mixed colors** |
| **Quantity** | 3 to 40 flowers. The ball grows with the count and each flower resizes so it always looks full |
| **Pots** | Pot, Tall vase, Jar, Mug, Bowl, Bottle, Matka, Egg planter, Planter, Round bowl, Fluted |
| **Pot prints** | 9 painted designs, 6 Rajasthani and Gujarati prints, 4 modern art prints, 3 aesthetic prints, 5 classic prints (see section 5) |
| **Pot colors** | 10 swatches (terracotta, cream, indigo, madder red, saffron, bottle green, blush, charcoal, teal, sage) |
| **Stems** | Green or Dark green |
| **Bouquet shape** | Round ball of flowers, facing outward on all sides, slowly turning and following the pointer |
| **Butterflies** | 4 small butterflies fly around the bouquet and leave when the QR code forms |
| **QR code** | Tap the bouquet (or press **Bloom QR**) to turn a link into flowers. Tap the code to go back |
| **Share and save** | Download image (PNG), Share image (device share sheet), Share link (opens the same design) |
| **Other** | Shuffle arrangement, light and dark mode, works on desktop and mobile |

---

## 2. How to use it

1. **Paste a link** in the box at the top of the side panel. It starts with a sample link, `https://claude.ai`.
2. **Pick your flower, colors, stems, pot, print, pot color and quantity.** Everything updates live.
3. **Tap the bouquet**, or press **Bloom QR**. The flowers fly into a QR code.
4. **Tap the QR code**, or press **Back to bouquet**, to return.
5. **Download image**, **Share image** or **Share link** to send your creation.

> **Scanning tip:** when the flowers form the QR code, they automatically darken so the code has enough contrast. Always test with your phone camera before printing or sharing.

Links up to 90 characters work. If you type a domain such as `example.com`, `https://` is added for you.

---

## 3. Tech stack

| Piece | Details |
|---|---|
| 3D engine | [three.js r128](https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js) from cdnjs |
| QR generation | [qrcode-generator 1.4.4](https://cdnjs.cloudflare.com/ajax/libs/qrcode-generator/1.4.4/qrcode.min.js) from cdnjs (error correction level M) |
| Font | Bricolage Grotesque from Google Fonts (falls back to the system font) |
| Everything else | Plain HTML, CSS and JavaScript in one file |

Both scripts are pinned to exact versions. There is no build step and no other dependency.

---

## 4. How it works

### 4.1 Rendering with instancing

Every flower is made of many small parts. To keep this fast, each kind of part is drawn with a three.js `InstancedMesh` (one draw call for thousands of copies):

| Mesh | Used for |
|---|---|
| `bm.cup`, `bm.strap`, `bm.lance`, `bm.lily`, `bm.lotus` | Petals (one mesh per petal shape) |
| `pet` | Small ellipsoids: stamens, anthers, sunflower seeds |
| `cen` | Flower centers |
| `stem` | Stems |

Each flower in the scene is a **head** with its own position, rotation, scale and growth values. Every frame, each head eases toward its target, then its petal matrices are written to the instanced meshes. When nothing is moving (for example, a finished QR code), the page skips the work and does not re-render.

### 4.2 Petal shapes

`SH` defines five curved petal shapes, built by `bladeGeo()`:

| Shape | Look | Used by |
|---|---|---|
| `cup` | Wide, rounded, cupped petal | Rose, Peony, Tulip |
| `strap` | Long, narrow, rounded tip | Daisy |
| `lance` | Pointed, slightly curved | Sunflower |
| `lily` | Pointed, curls back at the tip | Lily |
| `lotus` | Broad, boat shaped, pointed | Lotus |

Each shape has functions for the bend angle (`th`), half width (`w`), edge curl (`cup`), a base tint and the number of instance slots per flower.

### 4.3 Flower definitions

Flowers live in the `FL` object. Petals are listed as rings:

```js
// bl: [count, tilt, length, widthScale, roll, baseRadius, lift, phase?]
rose:{ n:'Rose', sh:'cup', cols:['#d63a4f','#e0556b','#c92d45'],
       dk:.3, sp:1.1, ex:1.1,
       bl:[[3,.05,.6,.9,.6,0,0], [4,.18,.78,.95,.5,.04,0], ...] }
```

| Field | Meaning |
|---|---|
| `n` | Display name |
| `sh` | Petal shape key from `SH` |
| `cols` | Natural colors, used when the palette is "Original" |
| `bl` | Petal rings (see above). Total petals must fit the shape's slot count |
| `el` | Optional ellipsoid parts (stamens, seeds): `[count, tilt, len, wid, thick, r, lift, roll, [radius,y]?, phase?, color?]` |
| `c`, `cs`, `cf` | Center color, size and flatness (leave out `c` for no center) |
| `dk` | Darkens inner petals for depth |
| `sp` | Rotation offset between rings (makes the spiral) |
| `ex` | Visual radius, used to size the flower consistently |

`bl` and `el` are turned into transform matrices once at startup (`BL`, `EL`, `CEN`, `EXT`).

### 4.4 Bouquet layout

Flower heads are placed on a sphere using a **Fibonacci spiral**, covering most of the ball (the lowest part is left open so stems can enter the pot). Each flower faces straight out from the center of the ball. The ball radius `R` and flower radius `rad` are computed from the quantity `N` so the surface always looks full. A short stem runs from the pot mouth to each flower.

### 4.5 QR code

1. The link goes to `qrcode(0,'M')`, which picks the smallest size that fits.
2. Every dark module becomes the target position of one flower head, on a flat grid facing the camera.
3. The list of cells is shuffled so the flowers fly in from varied places.
4. Extra flowers beyond `N` are created as needed (up to `MAXH = 1000`). Flowers not needed shrink away.
5. For contrast, petal colors are clamped to a dark lightness while the code is showing. A cream rounded card fades in behind as the light background.

### 4.6 Pots

Pots are lathe (spun) shapes defined in the `POTS` array:

```js
[name, topHeight, profile[[radius,height], ...], hasHandle?, 'flute'?]
```

- The profile goes up the outside, over the rim, and down the inside.
- Texture coordinates are rewritten so the vertical texture position follows real height, so painted bands line up on every shape.
- The `'flute'` flag adds vertical ridges.
- The mug's handle is a separate torus mesh sharing the pot material.

### 4.7 Pot prints

Prints are drawn onto a canvas and used as the pot texture.

| Type | Function | Canvas | Notes |
|---|---|---|---|
| Small tiled prints | `styleMat()` | 128 × 128 | Use the selected pot color plus derived ink and accent colors. Includes Rajasthani, modern art, aesthetic and classic prints |
| Painted designs | `bigTex()` | 256 × 256, wraps once around the pot | Keep their own palettes (except Banded motifs and Chevron, which use the pot color) |
| Sunset | `sunDraw()` | 128 × 128 | The gradient animates slowly from peach to night blue |

Picking a painted design also switches to a matching pot shape (`SUG` map), which you can change afterwards.

### 4.8 Butterflies

Four butterflies are small groups: a body plus two mirrored wing planes with a painted canvas texture. `bfPos()` moves each one along a looping path around the bouquet. Wings flap by rotating around the body axis. They fade out when the QR code is shown.

### 4.9 Share, download and design links

- **Download image** and **Share image** render the current view to a PNG. Inside a hosted page, saving uses the platform's `downloads` capability. Outside it, a normal download link is used.
- **Share link** builds a link to the page with the current design in the address hash:

```
#d-<flower>-<palette>-<potIndex>-<print>-<potColorIndex>-<stem>-<quantity>
# example: #d-rose-orig-6-matka-0-g-14
```

The page reads this hash on load and applies the settings. It is a single plain name, so it works in hosted pages. The QR link itself is **not** stored in the hash.

| Part | Values |
|---|---|
| flower | rose, tulip, daisy, sunflower, lily, peony, lotus, mixed |
| palette | orig, blush, marigold, sunbeam, poppy, azure, mixed |
| potIndex | 0 Pot, 1 Tall vase, 2 Jar, 3 Mug, 4 Bowl, 5 Bottle, 6 Matka, 7 Egg planter, 8 Planter, 9 Round bowl, 10 Fluted |
| print | any print key (see section 5) |
| potColorIndex | 0 terracotta, 1 cream, 2 indigo, 3 madder red, 4 saffron, 5 bottle green, 6 blush, 7 charcoal, 8 teal, 9 sage |
| stem | g (green) or d (dark green) |
| quantity | 3 to 40 |

> **Update `BASE`** in the script (`const BASE='https://claude.ai/artifact/...'`) to the address where you host the page, so shared links point to the right place.

---

## 5. Pot print keys

| Group | Keys |
|---|---|
| Painted pots | `matka`, `lotus`, `daisy`, `fruit`, `mughal`, `bands`, `chevron`, `tropical`, `golddrip` |
| Rajasthani and Gujarati | `bandhani`, `paisley`, `block`, `mirror`, `jaali`, `patola` |
| Modern art | `memphis`, `bauhaus`, `cutouts`, `terrazzo` |
| Aesthetic | `sunset`, `boho`, `minimal` |
| Classic | `plain`, `stripes`, `dots`, `checks`, `waves` |

---

## 6. Customizing

**Add a pot shape:** add a row to the `POTS` array. It appears automatically as a chip. Remember the new index when using design links.

**Add a pot color:** add a hex value to `POTCOLS`.

**Add a flower color palette:** add an entry to `PAL` (an array of hex colors) and a matching chip in the Colors row.

**Add a flower:** add an entry to `FL` using an existing shape from `SH`. Keep the total petal count within that shape's `slots` and the ellipsoid count under `MAXE` (40). Add `ex` so it sizes correctly.

**Add a petal shape:** add an entry to `SH` with `nt`, `th`, `w`, `cup`, `tint`, `tg` and `slots`. A new mesh is created for it automatically.

**Add a print:**
1. Add a `case` in `styleMat()` (small tile) or `bigTex()` (painted design, also add its key to `BIGP`).
2. Add a chip in the `G` list inside the print chip builder.
3. Add the key to `PATS` so design links accept it.

**Change the default look:** edit the state line near the top of the script (`sel`, `pal`, `N`, `potIdx`, `potPat`).

---

## 7. Limits and performance

| Setting | Value |
|---|---|
| Maximum flower heads | 1000 (`MAXH`) |
| Longest link | 90 characters |
| Petal instance slots | cup 36, strap 32, lance 30, lily 6, lotus 24, ellipsoids 40 per flower |
| Quantity range | 3 to 40 |

A long link makes a bigger QR code with more flowers, which is heavier on older phones. The page stops drawing when the scene is still, which keeps it light once the QR code has formed.

---

## 8. Running it yourself

1. Save `bloom-studio.html` anywhere.
2. Open it in a modern browser (Chrome, Edge, Safari, Firefox). It needs an internet connection for the two scripts and the font.
3. To host it, upload the single file to any static host. Update `BASE` in the script to your hosted address.

When opened as a normal file, **Download image** saves through a regular download link.

---

## 9. Known limitations

- The prints and flowers are procedural drawings, so they are simpler than real photographs or hand-painted pots.
- Several visuals (flower shapes, painted pot designs, butterflies) were tuned without a live browser preview, so some may need small adjustments.
- Copying the share link can be blocked by some browsers. The link also appears in a box so you can copy it by hand.
- Shared links only open for people who have been given access to your hosted page.
- Flowers are darkened in QR mode for scanning, so colors there are deeper than in the bouquet.

---

## 10. Credits

- Concept inspired by **Bloom QR** by Ann Nguyen (bubbbly.com/bloom).
- Painted pot designs are inspired by hand-painted Rajasthani, Mughal and modern planter photos supplied by the project owner.
- Built with three.js and qrcode-generator.
- Created in a conversation with Claude (Anthropic).
