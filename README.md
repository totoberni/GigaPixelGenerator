# Gigapixel Mathematical Graph Generator

A high-performance pipeline for generating massive, gigapixel-scale images from mathematical functions. This project leverages a hybrid GPU/CPU rendering approach, tiled processing, and efficient storage backends to create stunningly detailed visualizations that can scale to terapixel sizes.
Why? **Because I CAN!**

## Table of Contents
1.  [Features](#features)
2.  [How It Works: The Pipeline](#how-it-works-the-pipeline)
3.  [The Mathematical Kernels](#the-mathematical-kernels)
4.  [System Requirements](#system-requirements)
5.  [How to Use](#how-to-use)
6.  [Configuration Parameters](#configuration-parameters)
7.  [Code Structure](#code-structure)
8.  [Validation System](#validation-system)

---

## Features

-   **Hybrid Rendering Engine**: Utilizes both GPU (via CUDA/CuPy) and CPU (via Numba) for parallel processing, balancing the load and maximizing throughput.
-   **Tiled Architecture**: Renders massive images in smaller, manageable tiles, allowing generation of images far larger than available RAM.
-   **Efficient Zarr Backend**: Uses the Zarr library for intermediate storage of computed tiles, which is optimized for chunked, compressed, and parallel read/write operations.
-   **Checkpoint & Resume**: Automatically saves progress and can resume an interrupted render, saving hours or even days of computation.
-   **Seamless Imaging**: Implements a tile overlap system to eliminate grid artifacts and ensure a perfectly stitched final image.
-   **Robust Validation**: Includes built-in checks to ensure numerical consistency between GPU and CPU outputs and to verify that tile boundaries are seamless.
-   **Industry-Standard Output**: Converts the final Zarr dataset into a pyramidal BigTIFF file using `pyvips`, ideal for deep-zoom viewers and professional use.
-   **Automated Previews**: Generates multiple down-scaled PNG versions for easy sharing and viewing.

---

## How It Works: The Pipeline

The notebook executes a multi-stage pipeline to go from a mathematical formula to a final set of image files.

1.  **Setup & Configuration**: The user defines all parameters in **Cell 11**. This includes the final image dimensions, the mathematical coordinates (`center`, `scale`), the function to use (`kernel_type`), and output paths.
2.  **Validation (Optional)**: If enabled, the pipeline first runs a smaller-scale generation. It validates GPU-CPU consistency, checks coordinate systems, and verifies that the tile overlap logic produces seamless boundaries.
3.  **Tile Generation**: The `GigapixelGenerator` calculates the grid of tiles required for the final image. It checks the checkpoint file to see which tiles have already been completed.
4.  **Hybrid Rendering**: The generator processes the list of remaining tiles in batches. It strategically alternates between the GPU and CPU renderers (`HybridGPUCPURenderer`) to compute the pixel data for each tile.
5.  **Zarr Storage**: As each tile is rendered, it's immediately written to a Zarr datastore on disk (`image_output.zarr`). This minimizes memory usage and secures the rendered data.
6.  **Checkpointing**: After a set number of tiles, the generator updates a `progress.json` file in the checkpoint directory. If the script is stopped, it can resume from this point.
7.  **TIFF Conversion**: Once all tiles are stored in the Zarr dataset, the `ZarrToTIFFConverter` reads the entire dataset and uses the high-performance `pyvips` library to create a single, tiled, pyramidal BigTIFF file (`image_output.tiff`).
8.  **PNG Preview Generation**: Finally, the script reads the high-resolution TIFF and generates several smaller PNG previews at resolutions defined in the configuration.

---

## The Mathematical Kernels

At the heart of the generator are the mathematical functions, or "kernels," that define the visual output. The pipeline includes two primary kernels, written in both CUDA C++ for the GPU and Numba-accelerated Python for the CPU. The final value is then mapped to a custom Blue-Black-Red colormap.

### The 'Complex' Kernel (Main Function)
This is the primary function designed to create intricate and visually rich patterns. Its formula is:

$$ f(x, y) = \sin(1000xy) + \left(\cos(x^2 + y^2)\right)^{\sin\left(100\sqrt{x^2+y^2}\right)} - 0.5 $$

Because raising a negative number to a non-integer power is undefined in real numbers, the second term, $(\cos(r^2))^{\sin(100r)}$, requires special conditional logic. The kernel implements this logic to avoid mathematical errors and produce a continuous, well-defined plot across the entire coordinate space.

### The 'Simple' Kernel (Validation Function)
This kernel is used primarily for validation and testing. It's a straightforward function that produces a predictable, wavy pattern, making it ideal for verifying that the rendering pipeline, tiling, and coordinate systems are all working correctly. Its formula is:

$$ f(x, y) = \sin(xy) $$

---

## System Requirements

-   **Environment**: Google Colab is the primary target. It can also be adapted for any Linux-based system with the required hardware.
-   **GPU**: An NVIDIA GPU with CUDA support is required for accelerated rendering. The code is tested on a T4 GPU in Colab.
-   **Storage**: Sufficient Google Drive (or local disk) space is crucial. A gigapixel image can easily consume hundreds of gigabytes for the Zarr, TIFF, and checkpoint files.

---

## How to Use

This notebook is designed to be run cell by cell in a Google Colab environment.

### Step 1: Open in Google Colab
-   Upload the `.ipynb` file to your Google Drive.
-   Open the notebook with Google Colab.
-   Ensure the runtime is set to use a **GPU accelerator** (`Runtime` -> `Change runtime type` -> `T4 GPU`).

### Step 2: Initial Setup
-   Run **Cell 0 (`Installs`)** to install all required Python libraries.
-   Run **Cell 1 (`Imports`)** to import the libraries.
-   Run **Cell 1.5 (`Mount Google Drive`)** to connect your Google Drive. You will need to authorize this connection. This is where your output files will be saved.

### Step 3: Configure Your Render (Cell 11)
This is the main control panel for your image. Before running the full pipeline, modify the parameters in **Cell 11** to define your desired output.

-   **`IMAGE_WIDTH` / `IMAGE_HEIGHT`**: Set the dimensions of your final image.
-   **`MAIN_KERNEL`**: Choose `'complex'` or `'simple'` for the mathematical function.
-   **`CENTER_X`, `CENTER_Y`, `SCALE`**: Define the coordinate space to explore. `SCALE` acts as the zoom level (smaller values = more zoom).
-   **`BASE_DIR`**: This is automatically set to a unique directory in `/content/gigapixel_output/` for each run. You can change this to a persistent Google Drive path like `/content/drive/MyDrive/GigaPixelRuns/`.

### Step 4: Execute the Pipeline
-   Run **Cell 12 (`Execute Pipeline`)** to start the generation process.
-   The script will print detailed logs, including progress bars for tile generation and file conversion.

### Step 5: Retrieve Your Files
-   Once the pipeline is complete, your files will be located in the directory specified by `BASE_DIR`.
-   You will find:
    -   `checkpoints/`: The progress and recovery data.
    -   `image_output.zarr/`: The raw, tiled pixel data.
    -   `image_output.tiff`: The final, high-resolution BigTIFF image.
    -   `png_previews/`: A folder containing the smaller PNG versions.

---

## Configuration Parameters

All key settings are located in **Cell 11**.

| Parameter               | Description                                                                                             | Example                    |
| ----------------------- | ------------------------------------------------------------------------------------------------------- | -------------------------- |
| `RUN_ID`                | A unique identifier for the current run, used for the output directory name.                              | `run_20250902_231751`      |
| `MAIN_KERNEL`           | The mathematical function to use for the main render: `'complex'` or `'simple'`.                          | `'complex'`                |
| `IMAGE_WIDTH`           | The final width of the image in pixels.                                                                 | `169420`                   |
| `IMAGE_HEIGHT`          | The final height of the image in pixels.                                                                | `169420`                   |
| `TILE_SIZE`             | The dimension (in pixels) of each square tile used for rendering.                                       | `1000`                     |
| `TILE_OVERLAP`          | The number of pixels to overlap between adjacent tiles to prevent seams.                                | `1`                        |
| `CENTER_X`, `CENTER_Y`  | The mathematical coordinates for the center of the image.                                               | `0.0`, `0.0`               |
| `SCALE`                 | The zoom factor. Smaller numbers mean higher magnification.                                             | `0.001`                    |
| `BASE_DIR`              | The root directory where all outputs for the current run will be saved.                                 | `/content/gigapixel_output/` |
| `CHECKPOINT_INTERVAL`   | Save progress after this many tiles have been rendered.                                                 | `600`                      |
| `VAL_ENABLED`           | Set to `True` to run a smaller validation render before the main one.                                   | `True`                     |
| `TIFF_COMPRESSION`      | The compression method for the final TIFF file. `'deflate'` is a good lossless option.                  | `'deflate'`                |
| `PNG_SIZES`             | A list defining the resolutions for the PNG previews.                                                   | `[('thumbnail', 1024, 1024)]` |

---

## Code Structure

The notebook is organized into several key classes, each with a specific responsibility:

-   **`HybridGPUCPURenderer` (Cell 3)**: The core engine. It contains both the GPU and CPU logic for computing the pixel values for a given tile. It also houses the validation kernel.
-   **`ZarrTiledStorage` (Cell 4)**: Manages the creation and writing of tiles to the Zarr on-disk array structure.
-   **`GigapixelGenerator` (Cell 6)**: The orchestrator. It manages the overall process, including tile generation, checkpointing, calling the renderer, and writing to storage.
-   **`MemoryPoolManager` (Cell 7)**: A helper class to monitor system and GPU memory, ensuring the process doesn't crash from out-of-memory errors.
-   **`ZarrToTIFFConverter` (Cell 8)**: Handles the final conversion from the Zarr dataset to the BigTIFF file format using `pyvips`.

---

## Validation System

To ensure correctness at massive scales, the pipeline includes a robust, multi-step validation process (configured in Cell 11 and run if `VAL_ENABLED = True`).

1.  **Coordinate Pattern Generation**: The renderer can generate a simple gradient pattern instead of the mathematical function. This is used to visually confirm that the coordinate system is correct and that tiles are being placed in the right locations.
2.  **GPU-CPU Consistency Check**: The `validate_gpu_cpu_consistency` method renders the same random tiles on both the GPU and the CPU. It then compares the results pixel by pixel to ensure they are numerically identical within a small tolerance (`VAL_GPU_CPU_EPSILON`). This guarantees that the complex CUDA C++ kernel and the Numba Python kernel are perfect matches.
3.  **Tile Boundary Verification**: After a validation image is generated, the `verify_tile_boundaries` method analyzes the pixels along the edges where tiles meet. It checks for any sharp discontinuities, ensuring that the `TILE_OVERLAP` mechanism is working correctly to produce a seamless image.
