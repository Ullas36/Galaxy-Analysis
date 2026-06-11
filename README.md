# Galaxies Spectrography and Analysis Pipeline

This repository contains a Python-based scientific pipeline for analyzing galaxy photometry and spectroscopy. Using astronomical data from the Sloan Digital Sky Survey (SDSS), the pipeline processes multi-band FITS image frames and 1D FITS coadded spectra to determine key physical parameters of galaxies, including redshift, dust extinction, chemical classification, Star Formation Rate (SFR), gas-phase metallicity, and source detection.

---

## 📁 Repository Structure

*   **`Galaxy_Analysis.ipynb`**: The main Jupyter Notebook executing the analysis pipeline. It contains cells for loading data, fitting emission lines, performing BPT diagnostics, calculating cosmological distances, stacking RGB frames, and detecting/cataloging nearby sources.
*   **`Untitled.ipynb`**: A lightweight scratch notebook containing initial astropy and matplotlib imports.
*   **`Galaxies Spectrography and Analysis 2/`**: The core project data directory.
    *   **`photometry_result_with_spectra.csv`**: A master metadata CSV containing cataloged results for all 15 galaxies, including sky coordinates (RA/DEC), spectroscopic redshift (Z), shape measurements (centroids, ellipticities, orientations, semimajor/semiminor axes), and multi-band fluxes.
    *   **`data/`**: Subdirectories for 15 individual galaxies (`galaxy_1` to `galaxy_15`). Each subdirectory contains:
        *   `u_frame.fits`, `g_frame.fits`, `r_frame.fits`, `i_frame.fits`, `z_frame.fits`: Calibrated FITS image frames corresponding to the SDSS filter bands ($u, g, r, i, z$).
        *   `spectrum.fits`: Calibrated 1D coadded spectrum from SDSS containing wavelength, flux, and error data.

---

## ⚙️ Analysis Pipeline Workflow

The pipeline inside [Galaxy_Analysis.ipynb](file:///f:/Galaxy_Analysis/Galaxy_Analysis.ipynb) is designed to run end-to-end for a single galaxy directory (configured by updating `GALAXY_DIR` in the config block). The execution steps are as follows:

### 1. Spectroscopic Reading & Emission Line Fitting
*   **Data Extraction**: Wavelengths are calculated from log-wavelength values (`10**tab['loglam']`) and fluxes are extracted from the coadded spectrum HDU.
*   **Line Modeling**: Fits 1D Gaussian models (`models.Gaussian1D`) using Levenberg-Marquardt least-squares optimization (`fitting.LevMarLSQFitter`) to measure four key emission lines:
    *   **H$\beta$** ($4861.0 \text{ Å}$)
    *   **[OIII]** ($5007.0 \text{ Å}$)
    *   **H$\alpha$** ($6563.0 \text{ Å}$)
    *   **[NII]** ($6583.0 \text{ Å}$)
*   **Redshift Refinement**: A first-pass fit uses the header redshift $z_{\text{hdr}}$ as a guess. The line centroids ($\mu$) are measured, and the median difference from rest wavelengths is used to calculate the final redshift ($z_{\text{final}}$).

### 2. Dust Extinction Correction (Balmer Decrement)
*   **Balmer Decrement**: The ratio of the measured H$\alpha$ and H$\beta$ fluxes is compared to the intrinsic Case B recombination ratio ($F_{\text{H}\alpha} / F_{\text{H}\beta} = 2.86$, assuming $T \sim 10^4 \text{ K}$).
*   **Color Excess $E(B-V)$**: Extinction is computed assuming the Cardelli extinction law ($R_V = 3.1$):
    $$E(B-V) = \frac{2.5 \log_{10}(F_{\text{H}\alpha}/F_{\text{H}\beta} \times \frac{1}{2.86})}{k(\text{H}\beta) - k(\text{H}\alpha)}$$
    where $k(\text{H}\beta) = 3.61$ and $k(\text{H}\alpha) = 2.53$.
*   **De-reddening**: Observed line fluxes are multiplied by correction factors $10^{0.4 A_\lambda}$ to recover intrinsic emission line strengths.

### 3. BPT Diagnostic Diagram & Classification
*   Calculates line ratios $\log([\text{NII}]/\text{H}\alpha)$ and $\log([\text{OIII}]/\text{H}\beta)$ using de-reddened fluxes.
*   Classifies the ionizing source of the galaxy using empirical and theoretical demarcation lines:
    *   **Kauffmann (2003)**: Star-forming vs. Composite/AGN demarcation.
    *   **Kewley (2001)**: Pure AGN vs. Composite demarcation.
*   Categorizes the galaxy as **Star-forming**, **Composite**, or **AGN/LINER**.

### 4. Cosmological Distances & Absolute Magnitudes
*   Uses `Planck18` cosmology from Astropy to calculate the Luminosity Distance ($D_L$ in Mpc) and Distance Modulus ($DM$ in magnitudes).
*   Calculates absolute magnitudes and color ($g - r$) if apparent magnitudes are available.

### 5. Star Formation Rate & Gas-Phase Metallicity
*   **Star Formation Rate (SFR)**: Calculated from the de-reddened H$\alpha$ luminosity:
    $$L(\text{H}\alpha) = 4\pi D_L^2 \times F_{\text{H}\alpha}$$
    $$\text{SFR } [M_\odot / \text{yr}] = 7.9 \times 10^{-42} \times L(\text{H}\alpha) \text{ [erg/s]}$$
*   **Gas-phase Metallicity**: Estimated using the empirical N2 ratio index:
    $$12 + \log(\text{O/H}) = 8.90 + 0.57 \log_{10}([\text{NII}] / \text{H}\alpha)$$

### 6. Image Stacking & Pseudo-RGB Visualization
*   Loads three bands ($r, g, i$).
*   Applies a `ZScaleInterval` normalization to enhance faint structures and stacks them as a pseudo-RGB composite image to visualize the target galaxy's spatial morphology.

### 7. Background Subtraction & Source Detection
*   Applies a 2D background model (`Background2D` with `MedianBackground`) to the detection band image (default is $r$-band).
*   Subtracts the background and smooths the image using a 2D Gaussian kernel.
*   Detects and segments sources using photutils `SourceFinder` and `deblend_sources`.
*   Applies a morphological classification based on ellipticity (elongation) and spatial size compared to the PSF guess to catalog objects as `star?` or `galaxy?`.
*   Saves the detected objects to `detected_sources.csv` in the galaxy's folder.

---


## 🚀 How to Run the Project

### Prerequisites
Install the required packages using pip:
```bash
pip install numpy matplotlib astropy photutils pandas
```

### Executing the Pipeline
1. Open the [Galaxy_Analysis.ipynb](file:///f:/Galaxy_Analysis/Galaxy_Analysis.ipynb) notebook in Jupyter Notebook or VS Code.
2. In the first cell, configure the target galaxy you wish to analyze by changing `GALAXY_DIR`:
   ```python
   # Edit to analyze galaxy_1 through galaxy_15
   GALAXY_DIR = os.path.join(BASE_DIR, "data", "galaxy_2") 
   ```
3. Run all cells in the notebook. Plots will render inline, and source detection results will be saved to `data/galaxy_X/detected_sources.csv`.
