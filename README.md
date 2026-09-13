# Vehicle Detection and Counting (YOLOv8)

Detects and counts vehicles crossing two lines in a traffic video, using YOLOv8 for
detection and its built-in tracker to follow vehicles between frames. Counts are split
by direction (an "up" line and a "down" line) and by vehicle class.

A sample video is included, so the project runs immediately after install with no extra
downloads or dataset setup.

## Results on the included sample

601 frames, 1280x720, 30fps, using `yolov8n`:

| Direction | Car | Truck |
|---|---|---|
| UP | 6 | 2 |
| DOWN | 12 | 1 |

These numbers are reproducible across machines. They have not been validated against
hand-labelled ground truth, so treat them as the model's output rather than a measured
accuracy figure. See Limitations.

## Requirements

Python 3.9+. All Python dependencies are in `requirements.txt`.

Model weights are **not** stored in this repository. `yolov8n.pt` (~6 MB) downloads
automatically into `weights/` on first run.

## Install

```bash
git clone https://github.com/sakshiroy2026/Vehicle-Detection-and-Counting.git
cd Vehicle-Detection-and-Counting

python -m venv venv
source venv/bin/activate        # Windows: .\venv\Scripts\Activate.ps1

pip install -r requirements.txt
```

Note on `lap`: the YOLO tracker requires it for track association and fails on the first
`model.track()` call without it. It is pinned in `requirements.txt` for that reason.

## Run

```bash
python track.py
```

A window opens showing bounding boxes, the two counting lines, and live counts. Press
`q` to stop early. Final counts print to the terminal either way, and per-second counts
are written to a `.log` file named after the video.

### Using a different video

Edit `video_path` near the top of `track.py`. The counting lines are defined in pixel
coordinates on the **half-size** frame:

```python
up_line   = ((320, 230), (620, 250))
down_line = ((60, 200), (340, 380))
```

These are tuned to the sample footage. A different camera angle needs different
coordinates, or nothing will be counted.

### Using a larger model

`yolov8n` is the default because it downloads automatically and runs on modest hardware.
For better detection, change one line in `track.py`:

```python
WEIGHTS_PATH = PROJECT_ROOT / "weights" / "yolov8m.pt"
```

Roughly 4x slower on CPU. See Limitations for what changes in the output.

## How the counting works

1. YOLOv8 detects objects each frame, restricted to classes `[0, 1, 2, 3, 5, 7]`
   (person, bicycle, car, motorcycle, bus, truck).
2. The tracker assigns a persistent ID so the same vehicle is followed across frames.
3. For each line, the 2D cross product of the line vector and the vector to the vehicle
   centre gives which side the vehicle is on. A sign change between frames means it
   crossed.
4. A `visited` set prevents a vehicle being counted twice on the same line.

## Project structure

```
track.py            Main script. Detection, tracking, line-crossing counts.
requirements.txt    Dependencies.
weights/            Model weights land here (gitignored).
video/              Sample footage.
templates/          Digit templates for timestamp OCR (see legacy scripts).
helper.py           Bounding box conversions and colour maps.
```

### Legacy scripts

`counting.py`, `infer_vid.py`, `vehicle_detection.py`, `train.py`, `val.py`,
`template_matching.py`, `template_preprocessing.py`, `plot.py` and
`dataset_preprocess.ipynb` are development tooling from earlier work. **They do not run**
as-is: they depend on an annotations folder, an external dataset directory, and trained
weights that are not part of this repository. They are kept for reference only.

`track.py` is the working entry point.

## Limitations

Known and unresolved:

- **Rider identification does not work as intended.** The person-to-vehicle pairing in
  `track.py` is not gated on the person class, so it associates any two nearby tracked
  objects. On the sample clip it reports riders while the person and motorcycle counts
  are both zero, which is self-contradictory. The README previously claimed rider boxes
  are merged with motorcycle boxes; that behaviour is not implemented.
- **A vehicle alone in frame is never counted.** The line-crossing check sits inside a
  nested loop over pairs of detections, so it requires at least two objects visible at
  once. This undercounts on sparse footage and makes the per-frame cost O(n squared)
  for no reason, since the crossing check does not depend on the second object.
- **Class instability inflates counts on larger models.** Switching to `yolov8m` raises
  the UP truck count from 2 to 5 while the DOWN count is unchanged, which is more likely
  label flicker or a broken track ID than five additional trucks.
- **No ground-truth evaluation.** There are no hand-labelled counts for the sample clip,
  so no accuracy figure is claimed.
- **Line coordinates are hardcoded** to the sample camera angle.