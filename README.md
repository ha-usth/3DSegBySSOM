# 3D Facet Segmentation using Spherical SOM

This project focuses on **unsupervised facet segmentation of 3D meshes** using **Spherical Self-Organizing Maps (S-SOM)**. The goal is to partition a mesh into connected surface regions with similar face-normal directions, recovering planar or approximately planar facets.

Face normals are unit vectors on the sphere $S^2$, making a spherical map a natural starting point for clustering surface orientations. The pipeline smooths normals while preserving sharp changes, learns normal prototypes, separates disconnected regions, and merges adjacent regions using a power score. Connectivity processing distinguishes spatially separate facets that share a similar orientation.

The input is a 3D mesh in `.obj` format. The repository also includes **part segmentation** with Shape Diameter Function (SDF) and/or curvature descriptors as an additional application; facet segmentation is the primary objective.

### Facet segmentation via face normals
<img width="3840" height="2088" alt="Seg_facet" src="https://github.com/user-attachments/assets/f43124b4-79de-4c4b-80d7-5c4eacc20f79" />

### Additional application: part segmentation via SDF and/or curvature
<img width="3835" height="2299" alt="Seg_part_interactive" src="https://github.com/user-attachments/assets/e737af08-94b7-4f98-82fb-45066f4fd29e" />
 
## Features

- Load an OBJ mesh and compute and smooth face normals for facet segmentation.
- Cluster normal directions with an S-SOM initialized on an enclosing sphere.
- Separate disconnected regions and merge adjacent regions using a power score.
- Absorb tiny regions with a single neighbor using a mesh-relative area threshold.
- Inspect results and adjust the merging threshold interactively with `pyvista`.
- Compare planar and spherical maps with paired seeds and exported diagnostics.
- Apply the additional part segmentation pipeline to SDF and/or curvature descriptors.

## Requirements
- Python 3.13.5
- Main libraries: numpy, scipy, pyvista, matplotlib, scikit-learn, and pymeshlab.

Install dependencies:

```bash
pip install -r requirements.txt
```

## Run

### Facet segmentation

```bash
python seg_facet.py --input=./datasets/3DPuzzle/brick_part01.obj
```

After the power merge, facet segmentation absorbs tiny regions with only one
neighbor into that neighbor. The default `--small-region-area-ratio 1e-5` means
0.001% of the total mesh area, independent of model scale.
Use `--small-region-area-ratio 0` to disable this cleanup, or start with
`5e-6` for a lower area threshold. Regions touching multiple regions are preserved.
The same option applies to `seg_facet_planar.py`.

For a paired planar/S-SOM comparison, use `seg_facet_planar.py` with the explicit
settings in [the ablation section](#ablation-spherical-map-for-facet-segmentation).

### Additional application: part segmentation

`--fea` accepts `sdf_cur`, `sdf`, or `cur`:

```bash
python seg_part.py --input=./datasets/Princeton/30.obj --fea="sdf_cur" 
```

## Illustration of interactive adjustment of the segment merging threshold

Adjust the slider to inspect how the merging threshold changes the segmentation.

![Interactive](https://github.com/user-attachments/assets/4c04e266-3e20-4031-beb0-73d5600f33d7)


## Ablation: Spherical Map for Facet Segmentation

The primary comparison asks whether the spherical map with enclosing-sphere
initialization produces more coherent facet segments than an open planar SOM.
Both maps receive the same smoothed face normals and use the same number of
neurons, sampled-face sequence for each seed, update budget, learning-rate
schedule, and downstream facet processing. The planar map uses an open
triangular grid by default (`--grid hex`), with six neighbors at interior nodes.
Its neuron count is matched to `regular_sphere.obj` automatically.

### Initialization and training controls

| Setting | Planar SOM | S-SOM |
| --- | --- | --- |
| Lattice | Open grid with a boundary | Closed spherical mesh |
| Initial feature weights | PCA sheet or isotropic Gaussian (`--init pca` / `random`) | Enclosing sphere (`--sphere-init enclosing`) |
| Trainer in the matched comparison | `--kernel paper` | Actual `seg_facet.py` trainer |
| Neighborhood rule | Fixed n-ring connectivity, gated by current feature-weight distance | Same rule |
| Facet processing | Small-face correction, component separation, power merging, tiny-region cleanup | Same pipeline and thresholds |

This comparison changes **both lattice topology and initialization**. The
current facet comparison script always retains enclosing-sphere initialization
for S-SOM; it does not provide a shared-weight initialization control. Testing
both PCA and random planar initialization checks sensitivity to those two
choices, but does not isolate topology or rule out other initializations. Report
both baselines across the same seeds rather than selecting a favorable seed.

**The default `--n_rings 0` updates only the best-matching unit (BMU), so lattice
connectivity does not affect learning.** Such a run compares initialization and
prototype competition. Use a positive ring count to allow neighborhood updates,
and inspect `fraction_updates_with_non_bmu` and `mean_updated_non_bmu_neurons` in
the exported metrics: the feature-distance gate can still prevent neighbors
from updating. If these updates are absent or rare, evidence for a topology
effect is correspondingly limited.

Use `--kernel paper` for the matched training rule. `--kernel lattice` changes
only the planar branch to a Gaussian over fixed graph distances; S-SOM keeps its
original trainer. That option compares different training rules as well as maps.
The `--epochs` argument counts sampled updates, not full passes over all faces.

### Run the paired facet comparison

These examples use 1,000 updates, a 1-ring window, a learning rate decaying from
0.1 toward 0.01, and five paired seeds. They override the scripts' defaults of
2,000 updates, BMU-only learning, and an initial learning rate of 0.2.

```bash
python seg_facet_planar.py --input datasets/3DPuzzle/brick_part01.obj \
  --kernel paper --init pca --epochs 1000 --n-rings 1 \
  --lr 0.1 --lr-end 0.01 --radius 0.1 --power-thr 0.39 \
  --seeds 0 1 2 3 4 --no-render

python seg_facet_planar.py --input datasets/3DPuzzle/brick_part01.obj \
  --kernel paper --init random --epochs 1000 --n-rings 1 \
  --lr 0.1 --lr-end 0.01 --radius 0.1 --power-thr 0.39 \
  --seeds 0 1 2 3 4 --no-render
```

Remove `--no-render` for an interactive comparison, or replace it with
`--headless` to export figures offscreen. Each run writes `metrics.json`,
`metrics.csv`, stage labels (`*.seg`), meshes (`*.vtp`), and prototype/diagnostic
arrays (`*_som.npz`) under `outputs/topology_ablation/`. Multiple seeds also
produce a summary JSON with mean and standard deviation. Check
`merge_budget_reached` before comparing final region counts; if necessary,
increase `--max-merges` equally for both maps.

### Measure raw surface fragmentation

To measure fragmentation before any label correction or merging, add
`--fragmentation-only` to either command above. For example:

```bash
python seg_facet_planar.py --input datasets/3DPuzzle/brick_part01.obj \
  --kernel paper --init pca --epochs 1000 --n-rings 1 \
  --lr 0.1 --lr-end 0.01 --radius 0.1 --seeds 0 1 2 3 4 \
  --fragmentation-only
```

The surface fragmentation ratio is $\mathrm{SFR} = (C - A) / A$, where:

- $A$ is the number of active neurons in the untouched per-face BMU assignments.
- $C$ is the total number of edge-connected components of those assignments,
  including isolated single-face components.

SFR measures extra disconnected components per active cluster. Zero means each
active cluster is connected; it does not establish correct facet boundaries or
the correct number of facets. Separate facets with the same orientation may
legitimately share a BMU, so interpret SFR together with the mesh and final
segmentation. Raw-only runs skip rendering and all post-processing and export
`*_raw.npz` arrays alongside the metrics. For meshes containing zero-area faces
or undefined normals, use `--drop-degenerate-faces` to exclude them equally for
both maps in memory; the exports retain the original face IDs.

### Interpretation

The spherical map provides closed connectivity and initially distributes
prototypes around the normal sphere. A PCA initialization starts the planar
weights on a sheet, while random initialization gives them varying directions
and radii. These differences can affect how normals are assigned to prototypes
and how connected mesh regions emerge. However, feature weights in both maps
move freely in $\mathbb{R}^3$ during training: the planar weights can bend away
from their initial plane, and spherical weights are not constrained to remain
on a sphere. Neither lattice guarantees one neuron per facet or preservation of
all neighboring normal directions.

The illustration below provides a qualitative comparison. Claims about reduced
fragmentation or better facet recovery should be supported by paired results
across meshes and seeds, considering raw fragmentation, final boundaries, and
region counts together. Quantization error and topographic error are SOM
diagnostics, not direct measures of segmentation accuracy. The number of active
prototypes alone also does not establish final quality or runtime savings.

![Qualitative planar SOM and S-SOM facet segmentation comparison](https://github.com/anonymizedpaper/S-SOM/blob/d18a88302b7cc2ea487085158236ec46fe179ddc/Ablation_top.png)

### Additional control: part segmentation

Normalized SDF and/or curvature are scalar descriptors rather than unit normal
directions. The directional motivation for a spherical map therefore does not
transfer directly to this application. Use the separate part comparison script
to assess the effect empirically with identical data-sampled initial feature
weights for both maps:

```bash
python seg_part_planar.py --input datasets/Princeton/1.obj --fea sdf_cur \
  --init sample --seeds 0 1 2 3 4 --no-render
```

Both branches use the `SSOM2D.py` trainer. The default `--init reference`
reproduces the original uniform weights of 2; `--init sample` avoids that
identical-prototype starting condition. Results are saved under
`outputs/topology_ablation/part/`. Assess final segmentation quality and measured
processing times separately; the descriptor type alone does not establish an
accuracy or efficiency advantage for either map.
