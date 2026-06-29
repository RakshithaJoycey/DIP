# DIP Color Quantization System — Project Explanation

---

## What is Color Quantization?

Color quantization is the process of reducing the number of colors in an image.
Instead of using millions of possible RGB colors, we restrict the output image
to only use colors from a fixed palette — in this case, the Munsell color system.

The goal is: output image should look as close as possible to the original,
but only use colors that are human-visible and perceptually meaningful.

---

## Dataset

File: `anku_munsell_390.ods`

- Contains 1269 Munsell colors
- Each color has spectral reflectance data from 390nm to 800nm (411 wavelength bands)
- Also contains XYZ tristimulus values, CIE xy chromaticity, and Lab values
- These are real, physically measured human-visible colors

---

## Project Structure

```
color_quantization/
    color_utils.py       — color space math (conversions)
    palette_builder.py   — builds the 1269 and 3000 color palettes
    quantizer.py         — the main 6-step quantization pipeline
    run_comparison.py    — runs the pipeline and saves comparison images
    visualizer.py        — plotting and visualization functions
    main.py              — command line entry point
    app.py               — Streamlit web UI
    palette_lab_1269.npy — saved 1269 Lab palette
    palette_rgb_1269.npy — saved 1269 RGB palette
    palette_lab_3000.npy — saved 3000 Lab palette
    palette_rgb_3000.npy — saved 3000 RGB palette
```

---

## File-by-File Explanation

---

### 1. color_utils.py

This file contains all the color space conversion math.
It is used by every other file in the project.

Functions inside:

- `xyz_to_lab(X, Y, Z)`
  Converts XYZ tristimulus values to CIE L*a*b* color space.
  Uses D65 illuminant (standard daylight reference white).
  Formula: L = 116*f(Y/Yn) - 16, a = 500*(f(X/Xn) - f(Y/Yn)), b = 200*(f(Y/Yn) - f(Z/Zn))

- `lab_to_xyz(L, a, b)`
  Reverse of above. Converts Lab back to XYZ.

- `xyz_to_srgb_float(X, Y, Z)`
  Converts XYZ to sRGB in float range [0.0 to 1.0].
  Applies gamma correction (sRGB standard).

- `xyz_to_srgb_uint8(X, Y, Z)`
  Same as above but multiplies by 255 and rounds to integer.
  This is the format used for saving images (0 to 255 per channel).

- `srgb_to_linear(rgb)`
  Removes gamma from sRGB to get linear light values.

- `srgb_to_xyz(rgb)`
  Converts sRGB image pixels to XYZ.

- `srgb_to_lab(rgb)`
  Converts sRGB image pixels directly to Lab.
  Used in the quantization pipeline (Step 2).

- `delta_e_cie76(lab1, lab2)`
  Computes CIE76 Delta-E — the perceptual color difference between two Lab colors.
  Formula: sqrt((L1-L2)^2 + (a1-a2)^2 + (b1-b2)^2)

Why Lab space?
  Lab is perceptually uniform — equal numerical distances = equal perceived differences.
  This makes it the best space for color matching and interpolation.

---

### 2. palette_builder.py

This file builds the color palettes from the Munsell dataset.
It runs once and saves the palettes as .npy files for reuse.

What it does step by step:

Step 1 — Load dataset
  Reads anku_munsell_390.ods using pandas.
  Extracts X, Y, Z columns (tristimulus values).
  Converts XYZ → CIE Lab using color_utils.xyz_to_lab()
  Converts XYZ → sRGB uint8 using color_utils.xyz_to_srgb_uint8()
  Result: 1269 Lab colors and 1269 RGB colors

Step 2 — Sort for smooth spline traversal
  Function: sort_lab_for_spline()
  Uses a greedy nearest-neighbor chain algorithm.
  Starts from the darkest color (lowest L*).
  Always picks the next closest unvisited color.
  This ensures the sequence travels smoothly through color space
  so the spline does not jump between distant colors.

Step 3 — Catmull-Rom spline interpolation (1269 → 3000)
  Function: catmull_rom_interpolate()
  Uses scipy CubicSpline with 'not-a-knot' boundary condition.
  This is the mathematical definition of Catmull-Rom spline.
  Runs independently on each channel: L, a, b.
  Input: 1269 sorted Lab points
  Output: 3000 smoothly interpolated Lab points
  Clamps result to valid Lab range: L [0,100], a [-128,127], b [-128,127]

Step 4 — Build Global Palette
  Function: build_global_palette()
  Samples a uniform 22x22x22 grid in Lab space.
  Keeps only grid points inside the convex hull of the Munsell gamut
  (using scipy Delaunay triangulation).
  Combines grid points with Catmull-Rom interpolated colors.
  Converts all candidates to sRGB and removes out-of-gamut colors.
  Uses farthest-point sampling to select exactly 3000 colors
  that maximise coverage across the full Munsell color space.
  Converts final 3000 Lab colors → XYZ → sRGB uint8.

Saves:
  palette_lab_1269.npy  — 1269 x 3 float64 (Lab values)
  palette_rgb_1269.npy  — 1269 x 3 uint8   (RGB values)
  palette_lab_3000.npy  — 3000 x 3 float64 (Lab values)
  palette_rgb_3000.npy  — 3000 x 3 uint8   (RGB values)

---

### 3. quantizer.py

This is the core of the project. It runs the 6-step quantization pipeline.

The Pipeline:

  Step 1: Input RGB image (H x W x 3 uint8)

  Step 2: Convert RGB → Lab
    Uses srgb_to_lab() from color_utils.
    Every pixel gets converted from RGB to CIE Lab.
    Result: H x W x 3 float64 array of Lab values.

  Step 3: KDTree nearest neighbor mapping
    Builds a KDTree on the palette Lab colors (1269 or 3000).
    For every pixel's Lab value, finds the nearest palette color.
    Result: every pixel is now snapped to the closest Munsell color.
    This is the actual quantization step.

  Step 4: (Optional) Catmull-Rom smoothing
    If smooth=True, applies spline smoothing on the mapped Lab sequence.
    This reduces abrupt color transitions between adjacent pixels.
    smooth=False by default — NOT used for error metric evaluation.
    Only used as a visual enhancement option.

  Step 5: Convert Lab → RGB
    Converts the mapped Lab values back to XYZ then to sRGB uint8.
    This is the final output image.

  Step 6: Compute error metrics
    Function: compute_metrics()
    MSE   — Mean Squared Error between original and quantized pixel values
    PSNR  — Peak Signal-to-Noise Ratio (higher = better quality)
    SSIM  — Structural Similarity Index (1.0 = identical, 0 = no similarity)
    Delta-E — Perceptual color difference in Lab space (lower = better)

Key design: KDTree is built on Lab palette (not RGB) so nearest neighbor
search is perceptually accurate — it finds the color that looks closest
to the human eye, not just the numerically closest RGB value.

---

### 4. run_comparison.py

This file runs the full pipeline on input images and saves comparison outputs.

What it does:
  Loads both palettes (1269 and 3000 Lab colors).
  For each input image:
    Runs quantize() with 1269-color palette → computes metrics
    Runs quantize() with 3000-color palette → computes metrics
  Saves a comparison figure with 4 columns:
    Col 1: Original image
    Col 2: Quantized with 1269 colors
    Col 3: Quantized with 3000 colors
    Col 4: Grouped bar chart comparing MSE, PSNR, SSIM for both palettes

Output: output_1269_vs_3000_comparison.png

---

### 5. visualizer.py

Contains all the plotting functions used for analysis and visualization.

Functions:
  plot_lab_distribution()     — 3D scatter plot of colors in Lab space
  plot_palette_swatches()     — grid of color swatches for a palette
  plot_comparison()           — original vs quantized side by side
  plot_error_metrics()        — bar chart of MSE, PSNR, SSIM
  plot_lightness_histogram()  — histogram of L* values for 1269 vs 3000

---

### 6. main.py

Command line entry point for the project.

Usage:
  python main.py your_image.png
  python main.py your_image.png --no-dither
  python main.py your_image.png --no-plots

What it does:
  Loads or builds the palette (builds once, reuses after).
  Runs the quantization pipeline on the input image.
  Saves the quantized image as your_image_quantized.png
  Generates and saves all visualization plots.
  Prints error metrics to terminal.

---

### 7. app.py

Streamlit web UI for the project.

Run with:
  streamlit run color_quantization/app.py

Features:
  Upload any image from your browser.
  Toggle Floyd-Steinberg dithering on/off.
  View original vs quantized image side by side.
  See MSE, PSNR, SSIM, Delta-E metrics instantly.
  View 3D Lab distribution of palette colors.
  View lightness histogram.
  Download the quantized image.

---

## Output Files Generated

| File | Description |
|---|---|
| xyz_spectrum_analysis.png | Spectral reflectance curves (390-800nm) + mean spectrum |
| cie_xy_1269_vs_3000.png | CIE xy chromaticity diagram for 1269 vs 3000 colors |
| palette_1269_vs_3000.png | Color swatches of both palettes |
| graph_comparison.png | MSE, PSNR, SSIM bar charts for all test images |
| output_1269_vs_3000_comparison.png | Full quantization comparison with metrics |
| img1_original_vs_quantized.png | Image 1: original vs 1269 vs 3000 quantized |
| img2_original_vs_quantized.png | Image 2: original vs 1269 vs 3000 quantized |
| img3_original_vs_quantized.png | Camera photo: original vs 1269 vs 3000 quantized |

---

## Results Summary

| Image | MSE 1269 | MSE 3000 | PSNR 1269 | PSNR 3000 | SSIM 1269 | SSIM 3000 |
|---|---|---|---|---|---|---|
| Color grid 1 | 1987.26 | 1965.64 | 15.15 dB | 15.20 dB | 0.8053 | 0.8069 |
| Color grid 2 | 1877.03 | 1867.96 | 15.40 dB | 15.42 dB | 0.8450 | 0.8429 |
| Camera photo | 400.40  | 384.60  | 22.11 dB | 22.28 dB | 0.7578 | 0.8061 |

3000 colors consistently gives lower MSE and higher PSNR than 1269.
The camera photo shows the biggest improvement because natural photo colors
fall within the Munsell gamut. Neon color grids have higher error because
pure saturated neon colors do not exist in the Munsell system.
