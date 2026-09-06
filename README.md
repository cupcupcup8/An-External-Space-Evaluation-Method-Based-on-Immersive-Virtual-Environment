# An External Space Evaluation Method Based on Immersive Virtual Environment

Code for processing eye-tracking data in an external-space evaluation study based on an immersive virtual environment (IVE), conducted in a porcelain-culture commercial district.

## Project Background

This study introduces eye tracking into external-space evaluation within an immersive virtual environment: participants navigate and observe the external space of a commercial district in a VR scene while an eye tracker records sequences of gaze samples. Longer dwell times generally indicate that the corresponding spatial elements attract more visual attention, so effective fixations and their locations can be extracted from the dwell-time distribution and used as quantitative evidence of visual attraction.

The two files in this repository cover different stages of the data-processing pipeline:

- `eye-tracking.gh` (Grasshopper workflow) filters the raw eye-tracking coordinates exported by the VR experiment and keeps the fixations that fall on the study spaces / design elements;
- `data-processing.py` aggregates the time series of eye-tracking records into fixation groups and produces quantitative measures of gaze behavior for the subsequent spatial analysis.

## eye-tracking.gh (Grasshopper Definition)

`eye-tracking.gh` is a Grasshopper (Rhino) workflow that imports raw 3D eye-tracking coordinates from an immersive VR experiment, checks the positions of the gaze points, and outputs a dataset of **valid fixation points** for later spatial analysis. The workflow is split into two modules: data reading & parsing, and distance-based filtering.

### Module 1: Reading raw coordinates and generating 3D points

1. **Load raw data**: convert the exported eye-tracking data into a `.csv`/`.txt` file with the Time column hidden (X, Y, Z only); each line stores the raw coordinates of one fixation point.
2. **Parse line by line**: split the file content into lines and process them with a custom Python component, extracting the X, Y, Z values of each fixation into three coordinate lists.
3. **Generate 3D points**: map the three coordinate lists one-to-one and batch-generate 3D points in the Rhino scene that represent the gaze locations.

### Module 2: Filtering hit points by mesh distance

> Purpose: remove gaze points that do not land on the study space / design elements — a point too far from the element meshes is outside the study scope and should be discarded.

1. **Mesh input**: import the meshes of the design elements in the study area.
2. **Sample points**: extract sample points on the mesh surfaces for the distance comparison.
3. **Distance calculation**: compute the shortest distance from every gaze point to the area meshes.
4. **Threshold judgment**: compare the distances with a threshold set by a number slider. Within the threshold → valid point (landed on / near a design element); beyond the threshold → invalid point (drift).
5. **Dispatch & output**: split all gaze points into valid and invalid sets, count the valid set, and output the filtered fixation points for the subsequent element–space association analysis.

> Notes: the coordinate unit of the VR export must match the unit of the Rhino model; the input file path must not contain Chinese characters or special characters, otherwise no data will be read; import the whole model in Rhino and keep every design element selectable as a separate component so that Module 2 can decide which element a gaze point belongs to.

### Relationship with data-processing.py

The output of this workflow is the coordinates of all fixations that occurred within individual design elements / spaces. These valid points are then matched with their sampling times (Time) in a spreadsheet to build a file containing only valid fixations, which is handed over to `data-processing.py` for the time-series aggregation.

## data-processing.py

### Input

One `.xlsx` workbook in which **every worksheet** follows the same layout: column 1 (index 0) stores the sampling time, each visible (non-hidden) row represents one gaze sample, and the remaining columns are not used.

### Processing steps

1. **Time parsing**: three representations are accepted and converted to seconds —
   - `datetime.time` objects;
   - strings formatted as `minutes:seconds.milliseconds`, e.g. `12:34.567`;
   - floats interpreted as Excel day counts (multiplied by 86400).

   Unparseable values are counted as invalid and skipped without interrupting the run.

2. **Fixation grouping**: samples are visited in order; consecutive samples separated by **≤ 1 second** are merged into one group, and a gap **> 1 second** starts a new group. Each group records its start/end time (seconds), duration, number of samples, and whether it is an isolated sample (a group with a single sample).

3. **Significant fixation**: groups with a duration **> 0.2 seconds** are counted as significant fixation groups; the total significant duration and the first significant group are also reported.

4. **Output**: two worksheets are written to the result workbook —
   - `time_analysis_summary`: per-worksheet summary (visible sample count, invalid count, group count, significant group count, total significant duration, first significant group information);
   - `group_details`: per-group details (worksheet, group number, start/end seconds, duration, sample count, isolated flag).

### Usage

The script currently runs directly, and the input/output paths are hardcoded at the end of the file:

```python
# Replace these with your actual file paths
file_path = r"path\to\input\input_processed.xlsx"    # eye-tracking records (input)
output_path = r"path\to\output\analysis_result.xlsx"  # analysis result (output)
```

Then run:

```bash
python data-processing.py
```

## Constraints & Assumptions (read before use)

- **Only visible rows are analyzed**: hidden rows are skipped; a worksheet with no visible rows or with empty time values still appears in the summary (zero samples, no significant groups).
- **Column 1 must be the time column** (`TIME_COLUMN_INDEX = 0`), and the whole column may only mix the three formats listed above; string times only support `minutes:seconds[.milliseconds]` — neither hours nor `hh:mm:ss` are supported.
- **The thresholds are hardcoded empirical values, not configurable parameters**: the 1-second grouping gap and the 0.2-second significance duration are set inside the code; adjusting them requires editing the code and re-running.
- **An isolated sample never becomes a significant fixation**: a single-sample group has zero duration and can never exceed 0.2 s, so it contributes nothing.
- **The significant duration is a plain sum of per-group durations**; the gaps between groups (> 1 s) are not included.
- **The judgment is purely time-based**: coordinate columns are not read and sample events are not distinguished; "significant fixation" is a heuristic definition derived from the time series.
- **The input must be `.xlsx`** (openpyxl does not support legacy `.xls`); the example paths point to files on the author's local disk that are not part of this repository, so running the script as-is will fail — replace them first.
- The duplicate `TIME_COLUMN_INDEX = 0` assignment near the end of the script is harmless and can be safely removed.

## Environment & Dependencies

- Python 3.x;
- `pandas`, `numpy`, `openpyxl`;
- `eye-tracking.gh` requires a Rhino + Grasshopper environment.
