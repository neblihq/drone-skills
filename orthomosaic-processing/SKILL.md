---
name: orthomosaic-processing
description: Process a folder of drone photos into an orthomosaic (a georeferenced aerial map), plus a surface model and 3D reconstruction, using OpenDroneMap. Use this whenever a drone pilot wants to stitch flight photos into a map, build an orthomosaic or orthophoto, generate a DSM/elevation model or point cloud from drone imagery, "make a map from my drone photos", or process a photogrammetry flight — even if they don't name a specific tool.
---

# Orthomosaic Processing with OpenDroneMap

Turn a folder of overlapping drone photos into a georeferenced orthomosaic —
plus an elevation model and a 3D point cloud from the same run — using
OpenDroneMap (ODM), which is free, open source, and runs locally through Docker.

The whole workflow is: check the photos are processable, run one Docker
command, wait, then open the results. Most failed runs die for reasons that
were visible in the photos before processing started, so do the checks first —
they take a minute and can save an hour.

## What you need

- **The photos from one flight, in one folder.** JPGs straight off the drone.
  Don't mix flights from different days or altitudes in one run unless they
  were flown as a single mission.
- **Proper overlap.** Photogrammetry needs each ground point visible in
  several photos: roughly 75% front overlap and 65% side overlap is the
  standard target for mapping missions. Photos from a casual flight (a few
  snapshots, video stills, orbiting one object) usually cannot make an
  orthomosaic — that's the single most common cause of failed runs.
- **Geotags help a lot.** Nearly all drones (any DJI, Autel, Skydio, Parrot)
  write GPS positions into the photo EXIF automatically. ODM can run without
  them, but the result won't be placed on the map.
- **Docker installed** ([docker.com](https://www.docker.com/products/docker-desktop/)),
  with at least 8 GB of RAM available to it — 16 GB is comfortable for a few
  hundred photos. On Docker Desktop (Mac/Windows), RAM allocation is in
  Settings → Resources.
- **Disk space:** expect the output project to grow to roughly 10–20× the
  size of the input photos. Check free space before starting, not after it
  runs out mid-processing.

## Pre-flight checks

Run these before starting a processing run. Each catches a real failure mode.

**1. Count the photos and check they're from one camera:**

```bash
exiftool -Model -q -q -T *.JPG | sort | uniq -c
```

One camera model should come back. Mixed models means mixed flights — sort
that out first. (No exiftool? `brew install exiftool` on Mac, or
`apt install libimage-exiftool-perl` on Linux.)

For a mapping mission, expect dozens to hundreds of photos. Fewer than ~20
photos rarely produces a useful orthomosaic of anything bigger than a single
structure.

**2. Confirm geotags are present:**

```bash
exiftool -GPSLatitude -q -q -T *.JPG | grep -c '^-$'
```

This counts photos *missing* GPS. It should print `0`. If every photo lacks
GPS, the run will still produce an orthomosaic but it won't be georeferenced —
tell the pilot that up front, don't let them discover it in the output.

**3. Check nothing is corrupt** (a truncated file crashes a run hours in):

```bash
for f in *.JPG; do sips -g pixelWidth "$f" >/dev/null 2>&1 || echo "BAD: $f"; done
```

(Mac; on Linux use `identify` from ImageMagick the same way.) No output means
all files open.

**4. Check altitude consistency** — a mapping flight flies one altitude:

```bash
exiftool -RelativeAltitude -q -q -T *.JPG | sort -n | sed -n '1p;$p'
```

First and last values should be within a few meters of each other. A big
spread means mixed flight phases (takeoff/landing shots in the folder) —
those are fine to leave in, ODM handles them, but a *bimodal* set (half at
30 m, half at 100 m) is two different missions and should be split.

## Set up the project and run

**First, ask the pilot where this should run.** Processing is the expensive
part, and the machine in front of them is often not their best machine. Two
things decide it: an NVIDIA GPU speeds up dense reconstruction (one of the
longest stages — with default settings the GPU image still does feature
extraction on CPU; adding `--feature-type sift` moves that to the GPU too),
and the run doesn't care where it happens — the project folder is
self-contained. Ask: *"Do you have
an NVIDIA GPU — on this machine or another one (a workstation, a render
box)?"*

- **NVIDIA GPU on this machine (Linux or Windows):** use the
  `opendronemap/odm:gpu` image and add `--gpus all` to the `docker run`
  command shown below. Linux needs the NVIDIA Container Toolkit; Docker
  Desktop on Windows needs the WSL2 backend (its default). Verify passthrough
  before the real run: `docker run --rm --gpus all --entrypoint nvidia-smi
  opendronemap/odm:gpu` should print the GPU's name. (The `--entrypoint`
  matters: the ODM image runs its own processing script by default, so
  without it, `nvidia-smi` gets swallowed as arguments to ODM and errors.)
- **GPU on a different machine:** the workflow moves, unchanged. Copy the
  project folder to that machine (`rsync -a my-site/ user@host:my-site/`),
  run the same command there over SSH with the GPU variant above, and copy
  the results back. Run the pre-flight checks before copying — don't ship
  a broken photo set over the network to find out remotely.
- **Mac (any), or no NVIDIA GPU anywhere:** use the standard CPU image below,
  locally. Docker cannot pass Apple or AMD GPUs through, so there is no GPU
  path on a Mac — don't send the pilot hunting for one. CPU-only is slower,
  not worse: the outputs are identical.

If the pilot processes regularly on a shared machine or a cluster, ODM has
purpose-built tools for that — NodeODM (a processing node with an API) and
ClusterODM (distributes across nodes). Those are beyond this skill; point the
pilot at docs.opendronemap.org rather than improvising a setup.

ODM wants a project folder containing an `images/` subfolder:

```
my-site/
└── images/
    ├── DJI_0001.JPG
    ├── DJI_0002.JPG
    └── ...
```

Copy (don't move) the photos in, so the originals stay untouched. Then run:

```bash
docker run -ti --rm \
  -v /absolute/path/to/my-site:/datasets/code \
  opendronemap/odm \
  --project-path /datasets --dsm
```

That's the whole thing. Notes on what it does:

- The first run pulls the ODM image (~1.5 GB) before processing starts.
- `--dsm` adds the digital surface model (elevation raster) to the outputs.
  It's not produced by default, and it's one of the most-used products, so
  include it unless the pilot only wants the flat map.
- Defaults are good. ODM has hundreds of flags; resist tuning them on a first
  run. The two worth knowing exist: `--orthophoto-resolution 2` sets output
  resolution in cm/pixel (it won't exceed what the flight altitude actually
  captured), and `--fast-orthophoto` skips the 3D reconstruction for a much
  faster preview-quality map when the pilot is in a hurry.

**What normal looks like:** stages named `dataset`, `opensfm` (feature
matching — the long one), `openmvs` (dense point cloud), `odm_filterpoints`,
`odm_meshing`, `odm_texturing`, `odm_georeferencing`, `odm_dem`,
`odm_orthophoto` scroll by with timestamps. A few hundred 12 MP photos takes
roughly 1–3 hours on a modern workstation. Silence for 20–30 minutes during
`opensfm` and `openmvs` is normal — check CPU usage before assuming a hang.

## The outputs, and how to open them

Everything lands inside the project folder:

| File | What it is | Open with |
|---|---|---|
| `odm_orthophoto/odm_orthophoto.tif` | **The orthomosaic** — georeferenced GeoTIFF map | QGIS (free), or any GIS |
| `odm_dem/dsm.tif` | Surface elevation model (terrain + buildings/trees) | QGIS |
| `odm_georeferencing/odm_georeferenced_model.laz` | 3D point cloud, georeferenced | CloudCompare (free) |
| `odm_texturing/odm_textured_model_geo.obj` | Textured 3D mesh | CloudCompare, Blender |
| `odm_report/report.pdf` | Processing report: camera positions, quality stats | any PDF viewer |

**The outputs belong to root.** Docker runs the container as root, so every file
ODM writes is root-owned — copying them is fine, but deleting or moving them
needs `sudo` (`sudo rm -rf my-site`), and a plain `rm -rf` fails partway through
with a pile of "Permission denied" lines. Worth knowing before the project's
10–20× disk footprint becomes a problem.

Open the report first — it shows the reconstructed camera positions over the
survey area, which is the fastest way to confirm the whole flight was used.
Then the orthomosaic. In QGIS, drag the `.tif` in; it lands at its real-world
location if the photos were geotagged.

The GeoTIFF is big. To share a quick look without GIS software:

```bash
docker run --rm -v /absolute/path/to/my-site:/d \
  --entrypoint /code/SuperBuild/install/bin/gdal_translate \
  opendronemap/odm -of JPEG -outsize 20% 0 \
  /d/odm_orthophoto/odm_orthophoto.tif /d/preview.jpg
```

(Reuses GDAL from the ODM image already on disk — nothing new to install. The
full path is required: the image doesn't put its GDAL tools on `$PATH`.)

## When it goes wrong

- **"Not enough features" / most photos discarded / sparse patchy result:**
  insufficient overlap or textureless ground (water, uniform sand, snow).
  This is a data problem, not a settings problem — the fix is reflying with
  more overlap, not flag tuning. Check the report to see which photos failed
  to match.
- **Killed / exit code 137 / stops during `openmvs`:** out of memory. Give
  Docker more RAM (Settings → Resources on Docker Desktop), or add
  `--pc-quality medium` to halve the dense-reconstruction memory footprint,
  or both.
- **"No space left on device":** the 10–20× disk estimate above was real.
  Free space, then rerun — ODM can resume with `--rerun-from odm_filterpoints`
  if the early stages completed.
- **Orthomosaic has warped or melted-looking buildings at the edges:** normal
  at survey boundaries where overlap thins out. Plan flights with one extra
  pass beyond the area of interest; crop the edges in the meantime.
- **Output opens in the wrong place / no coordinates:** photos weren't
  geotagged (see pre-flight check 2).

## What this skill doesn't cover

Survey-grade absolute accuracy. Geotag-based georeferencing is typically
accurate to a few meters — fine for visualization, inspection, and progress
tracking. If the pilot needs centimeter-level accuracy (boundary surveys,
volumetrics for pay, engineering deliverables), that requires ground control
points (GCPs): surveyed targets laid out before the flight and measured into
the processing. ODM supports GCP files, but that workflow — surveying the
targets, building the GCP file, tagging them in images — is its own
discipline and isn't covered here. Say so plainly rather than implying the
geotag result is survey-grade.
