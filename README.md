# MicroED continuous-rotation acquisition with SerialEM

[日本語版](README_ja.md)

This repository contains SerialEM macros and a concise workflow for continuous-rotation MicroED on JEOL CRYO ARM systems.

> [!IMPORTANT]
> This is not a complete microscope operating manual. Alignments, calibrations, safety checks, camera and Low Dose configuration, and suitable dose conditions must be established and validated at each facility. Values originating from the CRYOARM200/Rio source workflow must not be copied blindly to another system.

## Files

```text
.
├── README.md
├── README_ja.md
└── JEOL_microED_Osaka-workflow.txt
```

`JEOL_microED_Osaka-workflow.txt` contains five macros:

- `MicroEDContinuousRotation`: continuous-rotation acquisition with `TiltDuringRecord`
- `BeforeTilt`: pre-acquisition flash/refill checks and Navigator-item realignment
- `AfterTilt`: saves a View image and restores selected conditions
- `ScreeningShot`: acquires a Preview diffraction image and saves a JPEG
- `ScreeningShotDamage`: performs a 180-second Preview exposure and saves damage-test data and a JPEG

## 1. Parameters to check or edit

### MicroEDContinuousRotation

| Variable | Meaning | Supplied value |
|---|---|---:|
| `angleBegin` / `angleFinish` | Start/end angles (degrees) | `-30` / `30` |
| `cameraBinning` | Binning for a non-DE camera | `2` |
| `secondsPerFrame` | Frame time for a non-DE camera (s) | `0.5` |
| `degreesPerSecond` | Rotation rate (degrees/s), used to calculate duration | `1.0` |
| `stageRateIndex` | JEOL stage-speed index passed to `TiltDuringRecord` | `102` |
| `preAcquisitionWait` | Delay after raising the screen (s) | `2` |
| `stageSettlingTime` | Optional settling delay after tilt (s) | `0` |
| `cameraID` | SerialEM camera ID | `1` |
| `recordPreset` | Camera Parameter Set used for acquisition | `R` |
| `processingSetting` | Value passed to `SetProcessing` | `2` |
| `useDECamera` | `1` for DE; `0` for non-DE | `0` |
| `deFramesPerSecond` | DE frame rate (fps) | `20` |

Acquisition duration is calculated as `ABS(angleFinish - angleBegin) / degreesPerSecond`. For a non-DE camera, `secondsPerFrame` is applied with `SetFrameTime`. In DE mode, binning is set to 2 and `deFramesPerSecond` is applied. Validate the camera ID, allowed binning, processing, frame saving, and the relationship between `degreesPerSecond` and `stageRateIndex` locally.

### BeforeTilt

| Variable | Meaning | Supplied value |
|---|---|---:|
| `FlashInterval` | C-FEG flash interval (s) | `8 * 3600` |
| `flash_when_refill` | Flash during a detected refill when `1` | `1` |
| `delay_after_refill` | Additional post-refill delay (min) | `5` |
| `darkRef_when_refill` | Defined but not used later in this file | `0` |
| `update_dark` | Defined but not used later in this file | `0` |

`BeforeTilt` calls the external macros `EMProperties` and `Funcs::CustomMoveStage`, which are not included. Validate these dependencies and the microscope-specific commands `LongOperation FF`, `LongOperation RS/RT`, and `AreDewarsFilling` before enabling automated flash/refill handling.

### ScreeningShotDamage

| Variable | Meaning | Supplied value |
|---|---|---:|
| `binning` | Preview binning | `2` |
| `totalExposureTime` | Total exposure (s) | `180` |
| `expTimeperFrame` | Frame time (s) | `1` |
| `currentCamera` | Camera ID; source comment: `1` Rio, `2` K3 | `1` |
| `darkgain` | Processing; source comment: `0` raw, `1` dark/gain, `2` gain-normalized | `2` |
| `recordMode` | Camera Parameter Set | `P` |

Confirm camera IDs, processing values, and safe exposure conditions at your facility.

## 2. Register the macros in SerialEM

1. Copy `JEOL_microED_Osaka-workflow.txt` to the SerialEM PC.
2. In SerialEM, open an existing destination script with **Edit**.
3. Open `JEOL_microED_Osaka-workflow.txt` in a text editor.
4. Copy from `MacroName MicroEDContinuousRotation` through its matching `EndMacro`, then paste it into the SerialEM script editor.
5. Repeat for `BeforeTilt`, `AfterTilt`, `ScreeningShot`, and `ScreeningShotDamage`, using a separate SerialEM script entry for each macro.
6. Save all five scripts and confirm that each is available by its macro name.
7. Confirm that `BeforeTilt` can call `EMProperties` and `Funcs::CustomMoveStage`.

Treat each `MacroName`-to-`EndMacro` block as one script. Check the destination before pasting so that an existing local script is not overwritten. The exact Edit and save procedure may vary with the SerialEM version and local configuration.

## 3. Measurement workflow

### A. Preparation

1. Load the grid onto the stage.
2. Start DigitalMicrograph and select the intended camera.
3. Start SerialEM. If the camera selection changes, select the intended camera again and stop any automatically started View acquisition.
4. Apply the required emission, flashing, and Auto Emission settings in TEM Center.
5. Check the View, Focus, Trial, Record, Preview, and Mont-map parameter sets. Record and Preview require diffraction mode and frame saving. Configure all values locally.

### B. Atlas and Atlas/View alignment

1. Open Navigator and select the facility's grid-map Imaging State.
2. Move to the grid-view position and check the Column Valve and standard focus.
3. Run the facility Atlas macro (`TakeAtlas` in the Osaka workflow) or an Atlas montage, saving it in a date/time folder.
4. Add a recognizable feature, such as contamination, as a Navigator point and move to it.
5. Center the same feature in Trial/View, place the marker at the corresponding position in the View image, and apply **Navigator > Shift To Marker** to all items.

`TakeAtlas`, `CallLDMRecord`, Imaging States, magnifications, apertures, and alignment files are not included and require facility-specific setup.

### C. View montages

1. Use **Add Points** on the Atlas to select squares likely to contain crystals.
2. Select **Stop Adding** and mark the points for acquisition (`A`).
3. Enable **New file at group** and choose montage acquisition. The source workflow used 3 × 3 montage and MRC integer output.
4. In **Navigator > Acquire at Items**, choose Mapping and configure montage saving, Navigator-map creation, Low Dose View, and Rough Eucentricity at every item.
5. Review the View montages and exclude items outside the facility's permitted stage-Z range.

### D. Beam and aperture adjustment

1. Select an empty carbon area in a View montage and move to the marker.
2. Insert the upper camera and enter the Trial Low Dose area.
3. Apply the facility's data-acquisition CLA setting and the required standard-focus and SerialEM-defocus reset.
4. Center the condensed beam; expand it and center the CLA aperture. Repeat as needed.
5. Set parallel illumination.
6. In the Record diffraction condition, minimize and center the diffraction center.
7. Return to View and correct any large beam-position shift.
8. Repeat Trial → Record → View until the centers converge.
9. Blank the beam and stop the upper camera.

CLA and DAC values are facility-specific and are therefore not prescribed here.

### E. Screening

1. In **Camera & Script > Setup**, confirm frame saving for Preview and Record.
2. Set a base name, include the Navigator label and sequential number, and select a `screening` directory below the Atlas directory.
3. Open a View montage, select and move to a crystal, add a Navigator point, and acquire a View image.
4. Run `ScreeningShot` and inspect the diffraction pattern. Screen crystals of different shapes and sizes.
5. When required, run `ScreeningShotDamage`. Its supplied settings are a 180-second Preview exposure with 1 second per frame; verify that these conditions are suitable for the sample.

### F. Continuous acquisition

1. Confirm Record frame saving and select a `data` directory below the Atlas directory.
2. Select crystals with **Add Points** in each View montage and mark all selected points for acquisition (`A`).
3. Repeat the beam/aperture adjustment if necessary.
4. Open **Navigator > Acquire at Items** and select Final acquisition.
5. Set **Run script** to `MicroEDContinuousRotation`.
6. Set **Run script before action** to `BeforeTilt`.
7. Set **Run script after action** to `AfterTilt`.
8. Review the remaining Navigator options. The source workflow used Rough Eucentricity and skipped selected Z moves; validate these choices locally.
9. Select **GO** and monitor the first acquisitions. Check the diffraction appearance and angular range, agreement between TEM Center and SerialEM defocus, and relevant microscope readouts.

To end a Navigator run, the source workflow specifies **End Navigator**, not **Stop**. Before resuming, inspect the last attempted item and its acquisition flag.

## Notes

- `MicroEDContinuousRotation` inserts the energy-filter slit with `SetSlitIn 1`.
- `BeforeTilt` is facility-specific and cannot run unchanged without its external dependencies.
- `AfterTilt` saves a 512 × 512 View JPEG, enters Trial, and resets eucentric focus/defocus, stage tilt, and image shift.
- Atlas, alignment, calibration, `EMProperties`, and `Funcs` macros are not included.
- In the source package, the damage macro was named `ScreeningfShotDamage`. The public file corrects this to `ScreeningShotDamage`; the macro body is otherwise retained.

## Acknowledgements

The workflow was refered from *230901-CRYOARM200_microED_マニュアル* by Naruhiko Adachi. That manual acknowledges contributions from Fumiaki Makino, Haruaki Yanagisawa, Takanori Nakane, Akihiro Kawamoto, and Yusuke Yamada.

## References and code provenance

This workflow was developed with reference to the following:
- CRmov ver. 1.1, 2019-03-02, M. Jason de la Cruz, MSKCC.
- Takaba, K. et al., Journal of Structural Biology (2020).

When using this workflow, please cite the following publication:
Takaba, K., Maki-Yonekura, S., & Yonekura, K. (2020). Collecting large datasets of rotational electron diffraction with ParallEM and SerialEM. Journal of Structural Biology, 211(2), 107549. https://doi.org/10.1016/j.jsb.2020.107549

