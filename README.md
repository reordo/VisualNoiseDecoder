# Visual Noise Decoder

Hide images inside images. Decode and view them without CORS access.

Visual Noise Decoder is a browser-based experiment in image hiding and visual
decoding. An ordinary-looking cover PNG carries a masked secret image in its
least significant color bits. The viewer reconstructs the secret using SVG
filters and CSS blending, without reading the hosted image's pixels in JavaScript.

The project consists of three standalone HTML files, with no dependencies,
build step, or backend.

## Try the examples

These three pairs use different sizes and orientations. Each cover is twice
the width and height of its secret; all covers are below one megapixel.
The encoded PNGs, resized covers, and resized secrets are in [examples](examples/).

Seed phrase: `4-m88F7tBX13YQHZAU8wI8Rp3k7VG44S`

- [Open all three as a gallery][example-gallery]
- [Decode image 0: Moon hidden in Earth][example-0] (800 × 800 cover, 400 × 400 secret)
- [Decode image 1: Sun hidden in a hurricane view][example-1] (960 × 540 cover, 480 × 270 secret)
- [Decode image 2: aurora hidden in clouds][example-2] (600 × 900 cover, 300 × 450 secret)

Source images: NASA Goddard Space Flight Center —
[Earth](https://images.nasa.gov/details/GSFC_20171208_Archive_e001386),
[Moon](https://images.nasa.gov/details/GSFC_20171208_Archive_e000868),
[hurricane](https://images.nasa.gov/details/GSFC_20171208_Archive_e000525),
[Sun](https://images.nasa.gov/details/GSFC_20171208_Archive_e001435),
[clouds](https://images.nasa.gov/details/GSFC_20171208_Archive_e001403), and
[aurora](https://images.nasa.gov/details/GSFC_20171208_Archive_e001594).
NASA media are generally not subject to US copyright; see the
[NASA media usage guidelines](https://www.nasa.gov/nasa-brand-center/images-and-media/)
for conditions and exceptions. NASA does not endorse this project.

## Getting started

Download or clone [the repository](https://github.com/reordo/VisualNoiseDecoder),
then serve this directory over HTTPS or a local HTTP server. For example, with
Python installed:

```sh
python -m http.server 8080
```

Open `http://localhost:8080/encode.html`. Web Crypto and file system pickers require
a secure context; browsers treat localhost as a secure context.

## Encode images

Use [encode.html](https://htmlpreview.github.io/?https://github.com/reordo/VisualNoiseDecoder/blob/main/encode.html) to select or drop secret images and cover images.
Folder selection is available in supported browsers.

- Files are sorted by filename. Covers cycle through the secret images in that order.
- Choose a seed phrase and a first image index. Each output receives the next index.
- The cover is scaled and center-cropped to twice the secret's width and height.
- An optional maximum output side reduces the secret before encoding, preserving
  its aspect ratio apart from whole-pixel rounding. The hidden image is half the
  output size on each axis. Transparency is flattened onto white.
- Save individual PNGs, write them to a selected folder, or download a ZIP archive.

Output names include their indices: `name.vnd-0.png`. Keep these indices when
viewing the images. Folder output replaces existing files with matching names.

ZIP output stores PNGs without recompressing them. Where supported, it streams
directly to disk; otherwise, it collects the archive in memory before downloading.
Archives must be smaller than 4 GiB. Use folder output for larger collections.
Browsers may ask permission to download multiple individual files.

## View an image

[image.html](https://htmlpreview.github.io/?https://github.com/reordo/VisualNoiseDecoder/blob/main/image.html) takes its parameters from the URL fragment:

```text
image.html#url=https%3A%2F%2Fexample.com%2Fphoto.vnd-0.png&seed=your%20seed%20phrase&index=0
```

`url` is the direct URL of the original encoded PNG, `seed` is the encoding seed
phrase, and `index` is its image index (defaults to `0`). URL-encode parameter
values, especially URLs containing their own query parameters or fragments.

The image fits the viewport and stays centered. If it is larger than the viewport,
click to zoom toward the clicked point; click again to fit it. The same page can
be embedded in an iframe:

```html
<iframe
  src="image.html#url=https%3A%2F%2Fexample.com%2Fphoto.vnd-0.png&amp;seed=your%20seed%20phrase&amp;index=0"
  title="Decoded image"
  style="width: 100%; height: 600px; border: 0;"
></iframe>
```

For a circular image, such as an avatar, give the iframe a square size and round
its corners. `pointer-events: none` disables pointer interaction, including
click-to-zoom. Use a square secret image to fill the circle without empty space.

```html
<iframe
  src="image.html#url=https%3A%2F%2Fexample.com%2Favatar.vnd-0.png&amp;seed=your%20seed%20phrase&amp;index=0"
  title="Decoded avatar"
  tabindex="-1"
  style="width: 256px; height: 256px; border: 0; border-radius: 100%; pointer-events: none;"
></iframe>
```

## View a gallery

Use [gallery.html](https://htmlpreview.github.io/?https://github.com/reordo/VisualNoiseDecoder/blob/main/gallery.html) with the seed phrase and first image index.
Choose either **One URL per line** or **Extract PNG URLs**. Extraction accepts
HTML embed snippets, BBCode, or text containing direct HTTP(S) PNG URLs.

You can also open a gallery directly from its URL fragment. Repeat `url` for
each image, in order; `index` is the first image index and defaults to `0`:

```text
gallery.html#url=https%3A%2F%2Fexample.com%2Ffirst.png&url=https%3A%2F%2Fexample.com%2Fsecond.png&seed=your%20seed%20phrase&index=0
```

Alternatively, `urls` can contain URL-encoded input text. Add `mode=extract`
to extract PNG links from pasted HTML, BBCode, or other text; without it, the
input is treated as one URL per line. Opening either form fills the fields and
loads the gallery automatically.
Clicking **Show** after changing the fields updates the fragment to a shareable
link, including the input mode and original text. Each image also has an
**open separately** link.
The seed phrase is visible to anyone who receives the complete link.

Links retain their input order, including duplicates. Indices increase in that
order; the gallery does not infer indices from filenames. Put the links in the
same order as the encoded images.

Each image occupies its own row. Viewers load progressively as they approach the
viewport, and their height adjusts to the decoded image's proportions.

Use **original encoded PNGs**, not hosting pages, thumbnails, resized copies, or
recompressed images. Any change to the embedded color bits can destroy the secret.

## How it works

1. The encoder derives a key with SHA-256 from a fixed domain string, the image
   index, and the seed phrase. AES-CTR generates a deterministic mask stream.
2. Each secret RGB channel contains eight bits. Four quadrants of the cover each
   store two of those bits, XORed with mask bits, in their two least significant bits.
   The cover's upper six bits remain unchanged.
3. The viewer uses discrete SVG component-transfer filters to extract the stored
   bit planes. CSS `difference` blending removes the mask, and weighted
   `plus-lighter` blending reconstructs the RGB image.

Canvas is used for local encoding and for generating the viewer's mask images.
The remote carrier is displayed as CSS backgrounds, not read back from canvas.
This avoids needing the image host's permission for cross-origin pixel reads;
it does **not** bypass access controls, hotlink protection, or browser content
policies that prevent the image from loading at all.

## Limitations and security

This is a research demonstration, **not an audited encryption system**.

- Never reuse a seed/index pair for different secret images. Reusing it repeats
  the mask and exposes the XOR relationship between their secret pixels.
  Continue with unused indices when encoding another batch with the same seed.
- Use a long, random seed phrase. SHA-256 is not a password-hardening function;
  weak phrases are vulnerable to guessing.
- There is no authentication or integrity check. Wrong seeds, wrong indices, or
  modified PNGs can produce noise rather than a clear error.
- Dimensions remain visible, and the embedded data may be statistically detectable.
- A shared viewer link contains the seed. URL fragments are not included in the
  HTTP request, but are accessible to scripts on the viewer page and anyone who
  receives the link. Host the viewer somewhere you trust.
- Large images require substantial memory and rendering work. Gallery lazy loading
  reduces the initial load, but does not unload images already viewed.

Folder access and direct-to-disk export depend on File System Access API support,
primarily available in Chromium-based browsers. File selection and ordinary
downloads are the fallback. Decoding relies on browser SVG/CSS rendering behavior;
the viewer includes a raster refresh workaround for Chromium when reduced images
are restored after visibility or scale changes.

## Author

Keishin Senzaki (`reordo`)<br>
[riodaa@proton.me](mailto:riodaa@proton.me)<br>
[github.com/reordo](https://github.com/reordo)

[example-gallery]: https://htmlpreview.github.io/?https://github.com/reordo/VisualNoiseDecoder/blob/main/gallery.html#url=https%3A%2F%2Fraw.githubusercontent.com%2Freordo%2FVisualNoiseDecoder%2Fmain%2Fexamples%2Fgallery-0-earth.vnd-0.png&url=https%3A%2F%2Fraw.githubusercontent.com%2Freordo%2FVisualNoiseDecoder%2Fmain%2Fexamples%2Fgallery-1-hurricane.vnd-1.png&url=https%3A%2F%2Fraw.githubusercontent.com%2Freordo%2FVisualNoiseDecoder%2Fmain%2Fexamples%2Fgallery-2-clouds.vnd-2.png&seed=4-m88F7tBX13YQHZAU8wI8Rp3k7VG44S&index=0
[example-0]: https://htmlpreview.github.io/?https://github.com/reordo/VisualNoiseDecoder/blob/main/image.html#url=https%3A%2F%2Fraw.githubusercontent.com%2Freordo%2FVisualNoiseDecoder%2Fmain%2Fexamples%2Fgallery-0-earth.vnd-0.png&seed=4-m88F7tBX13YQHZAU8wI8Rp3k7VG44S&index=0
[example-1]: https://htmlpreview.github.io/?https://github.com/reordo/VisualNoiseDecoder/blob/main/image.html#url=https%3A%2F%2Fraw.githubusercontent.com%2Freordo%2FVisualNoiseDecoder%2Fmain%2Fexamples%2Fgallery-1-hurricane.vnd-1.png&seed=4-m88F7tBX13YQHZAU8wI8Rp3k7VG44S&index=1
[example-2]: https://htmlpreview.github.io/?https://github.com/reordo/VisualNoiseDecoder/blob/main/image.html#url=https%3A%2F%2Fraw.githubusercontent.com%2Freordo%2FVisualNoiseDecoder%2Fmain%2Fexamples%2Fgallery-2-clouds.vnd-2.png&seed=4-m88F7tBX13YQHZAU8wI8Rp3k7VG44S&index=2
