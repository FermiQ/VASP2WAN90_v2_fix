# Documentation for `mlwf.patch`

## Overview

This patch modifies the VASP (Vienna Ab initio Simulation Package) code, specifically the `src/mlwf.F` and `src/wave_high.F` files, to enhance its `VASP2WANNIER90v2` interface. The primary improvements focus on enabling and extending capabilities for non-collinear Wannier function calculations, providing finer control over output files, and supporting advanced spinor projection techniques.

This patch is intended for VASP version 5.4.4.pl2.

## Key Changes

The patch introduces significant modifications to VASP's Wannier90 interface logic. Below is a summary of the key changes:

### 1. Enhanced Control over Output Files via INCAR Tags:

New logical and integer INCAR tags have been added to give users more precise control over what data is calculated and written:

*   **`LWRITE_UNK`** (Logical, Default: `.FALSE.`): Controls whether to write the `UNK` (wave function overlap with plane waves) files.
*   **`LUNK_FMTED`** (Logical, Default: `.FALSE.`): If `LWRITE_UNK` is true, this determines if the `UNK` files are written in human-readable formatted text (`.TRUE.`) or binary unformatted (`.FALSE.`).
*   **`LREDUCE_UNK`** (Logical, Default: `.FALSE.`): If `LWRITE_UNK` is true, this option allows writing `UNK` files with reduced size (e.g., by skipping every other grid point).
*   **`LWRITE_SPN`** (Logical, Default: `.FALSE.`): Controls writing of `.spn` files, which contain the matrix elements of the Pauli spin matrices $\langle \psi_{m,k} | \sigma | \psi_{n,k} angle$. This is typically for non-collinear calculations and currently works only for serial execution.
*   **`LSPN_FMTED`** (Logical, Default: `.FALSE.`): If `LWRITE_SPN` is true, determines if `.spn` files are formatted.
*   **`LCALC_MMN`** (Logical, Default: `.TRUE.`): Controls whether the overlap matrices $M_{mn}^{(k,b)} = \langle u_{mk} | u_{nk+b} angle$ (for `.mmn` file) are calculated.
*   **`LWRITE_MMN`** (Logical, Default: Value of `.NOT.LWANNIER90_RUN`): Controls whether the `.mmn` file is written. Automatically set to `.FALSE.` if `LCALC_MMN` is false.
*   **`LCALC_AMN`** (Logical, Default: `.TRUE.`): Controls whether the projection matrices $A_{mn}^{(k)} = \langle \psi_{mk} | g_n angle$ (for `.amn` file) are calculated.
*   **`LWRITE_AMN`** (Logical, Default: Value of `.NOT.LWANNIER90_RUN`): Controls whether the `.amn` file is written. Automatically set to `.FALSE.` if `LCALC_AMN` is false.
*   **`LWRITE_EIG`** (Logical, Default: Value of `.NOT.LWANNIER90_RUN`): Controls whether the `.eig` file (eigenvalues) is written.

### 2. Non-Collinear Spin and Spinor Projection Enhancements:

*   **`W90_SPIN`** (Integer, Default: `0`): For collinear calculations, selects the spin channel to process:
    *   `0`: Process all spin channels.
    *   `1`: Process spin-up channel only.
    *   `2`: Process spin-down channel only.
*   **Spinor Projections (`proj_s`, `proj_s_qaxis` in `wannier90.win`):** The patch extends the projection capabilities (`VASP2WANNIER90v2`) to correctly handle spinor projections with arbitrary quantization axes. This allows projecting onto specific spin states (e.g., $s_z+$, $s_x-$) defined by `proj_s` (spin channel, e.g., 1 or -1) and `proj_s_qaxis` (quantization axis vector) in the `wannier90.win` file.
*   **Non-Collinear `UNK` files:** Enables writing of `UNK.NC` files for non-collinear calculations, which include both components of the spinor.

### 3. Modified/New Subroutines:

The patch modifies existing subroutines and introduces new ones in `mlwf.F` and `wave_high.F` to implement the above features.

**In `mlwf.F`:**
*   Reads and processes the new INCAR tags.
*   Handles logic for selective spin channel processing (`W90_SPIN`).
*   Implements writing of formatted/unformatted and full/reduced `UNK` files.
*   Adds routines for calculating and writing `.spn` files (e.g., `CALC_SPN_EXP`).
*   Modifies the handling of `.amn` projections to support spinor quantization axes (e.g., `CALC_OVERLAP_GN_ALL`, `CONSTRUCT_FUNCTION_RYlm_ALL`).
*   Adjusts logic for writing `.mmn`, `.amn`, and `.eig` files based on new INCAR tags.

**In `wave_high.F`:**
*   **`W1_SPN_DOT`**: New function to calculate the matrix elements $\langle W1 | \Sigma | W2 angle$ where $\Sigma$ is a Pauli spin matrix. Used for generating `.spn` files.
*   **`ECCP_NL_SPN`**: New subroutine related to the non-local contribution calculation when spin matrices are involved.
*   Modifications to existing routines to correctly pass and handle spinor components and new control flags.

## Important Variables/Constants (INCAR Tags)

The primary way to control the features introduced by this patch is through INCAR tags. See the "Key Changes" section above for a detailed list of these tags and their meanings (e.g., `LWRITE_UNK`, `LUNK_FMTED`, `W90_SPIN`, `LWRITE_SPN`, etc.).

Additionally, for spinor projections, parameters like `proj_s` and `proj_s_qaxis` are specified in the `wannier90.win` input file, not INCAR.

## Usage Examples

Applying and using this patch involves several steps:

1.  **Download the Patch**: Obtain the `mlwf.patch` file.
2.  **Apply the Patch**:
    *   Place `mlwf.patch` in the root directory of your VASP v5.4.4.pl2 source code.
    *   Navigate to the VASP root directory in your terminal.
    *   Run the command: `patch -p0 < mlwf.patch`
3.  **Recompile VASP**:
    *   You must recompile VASP for the changes to take effect.
    *   Ensure your `makefile.include` has the `-DVASP2WANNIER90v2` precompiler flag.
    *   Link against a compiled `libwannier.a` (Wannier90 library). Example `makefile.include` additions:
        ```makefile
        CPP_OPTIONS += -DVASP2WANNIER90v2
        LLIBS       += /path/to/your/wannier90_distro/libwannier.a
        ```
4.  **VASP `INCAR` Configuration**:
    *   Set `LWANNIER90 = .TRUE.` in your `INCAR` file to enable the VASP2Wannier90 interface.
    *   Use the new INCAR tags (listed in "Key Changes") to control the desired behavior (e.g., `LWRITE_UNK = .TRUE.`, `LUNK_FMTED = .TRUE.`, `W90_SPIN = 1`).
    *   Example `INCAR` snippet:
        ```ini
        LWANNIER90   = .TRUE.   ! Enable VASP2Wannier90 interface
        LWRITE_UNK   = .TRUE.   ! Write UNK files
        LUNK_FMTED   = .TRUE.   ! Write them in formatted (text) style
        LWRITE_SPN   = .TRUE.   ! Write .spn files (for non-collinear serial runs)
        W90_SPIN     = 0        ! Process all spin channels (if collinear)
        ```
5.  **Wannier90 Input (`wannier90.win`)**:
    *   Prepare your `wannier90.win` file as usual. For advanced spinor projections, include `proj_s` and `proj_s_qaxis` specifications within your `projections` block if needed.

## Dependencies and Interactions

*   **VASP Version**: This patch is specifically designed for **VASP v5.4.4.pl2**. Compatibility with other VASP versions is not guaranteed.
*   **Wannier90**: Requires a compiled Wannier90 library (`libwannier.a`) for linking during VASP compilation and for any subsequent Wannier90 post-processing (`wannier90.x -pp`). The patch enhances the interface that VASP uses to prepare input files for Wannier90.
*   **Fortran Compiler**: A Fortran compiler is needed to recompile VASP after applying the patch.
*   **MPI**: While VASP is typically run in parallel using MPI, some new features introduced by the patch (specifically `LWRITE_SPN`) might be restricted to serial execution (i.e., run on a single MPI process).

The patch modifies core VASP files (`mlwf.F`, `wave_high.F`) that are central to the VASP-Wannier90 interface. It changes how VASP prepares data for Wannier90 and what data it can output.
```
