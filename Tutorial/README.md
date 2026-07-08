# SpatialModal Tutorials

This directory contains example notebooks that demonstrate how to run SpatialModal on representative spatial transcriptomics datasets. The tutorials cover data loading, model training, spatial domain identification, and basic downstream visualization.

## Available Tutorials

| Notebook | Purpose |
|---|---|
| `DLPFC.ipynb` | Run SpatialModal on 10x Visium human dorsolateral prefrontal cortex data. |
| `Breast_Cancer.ipynb` | Analyze spatial domains in human breast cancer tissue. |
| `Mouse_Brain.ipynb` | Apply SpatialModal to mouse brain spatial transcriptomics data. |
| `Chicken_Heart.ipynb` | Analyze chicken heart developmental stages and downstream biological patterns. |
| `MTG.ipynb` | Run SpatialModal on human middle temporal gyrus data. |

## Data Organization

Download the data from Zenodo:

```text
https://doi.org/10.5281/zenodo.18220735
```

Place each dataset under the `Data/` directory. For a 10x Visium sample, the expected structure is:

```text
Data/
└── sample_name/
    ├── filtered_feature_bc_matrix.h5
    ├── spatial/
    │   ├── tissue_positions_list.csv
    │   ├── tissue_hires_image.png
    │   ├── tissue_lowres_image.png
    │   └── scalefactors_json.json
    ├── image_feat.csv
    └── sample_name_truth.txt
```

`image_feat.csv` is optional. If it is absent, SpatialModal can extract image features from the histology image by setting `if_img=True`. Ground-truth label files are optional and are only required for quantitative evaluation such as ARI or NMI.

For custom or non-Visium datasets, use an `AnnData` object with:

| Field | Requirement |
|---|---|
| `adata.X` | Gene expression matrix. |
| `adata.obsm["spatial"]` | Spatial coordinates with shape `n_spots x 2`. |
| `adata.obsm["feat"]` | Optional processed gene features. |
| `adata.obsm["image_representation"]` | Optional morphology/image features. |
| `adata.obs["ground_truth"]` | Optional annotation labels for evaluation. |

If `adata.obsm["feat"]` is not provided, SpatialModal will preprocess the expression matrix and use highly variable genes. If image features are not available, SpatialModal can be run without histological images by setting `image_use=False`.

## Minimal Workflow

```python
import torch
import pandas as pd
from Spatialmodal.spatialmodal import SpatialModal
from Spatialmodal.data_process import load_ST_file
from Spatialmodal.utils import clustering

sample = "151674"
fold = f"../Data/{sample}"
device = torch.device("cuda:0" if torch.cuda.is_available() else "cpu")

adata = load_ST_file(fold)

truth = pd.read_csv(f"{fold}/{sample}_truth.txt", sep="\t", header=None, index_col=0)
adata.obs["ground_truth"] = truth.reindex(adata.obs_names)[truth.columns[0]]
adata = adata[~pd.isnull(adata.obs["ground_truth"])].copy()
n_clusters = adata.obs["ground_truth"].nunique()

model = SpatialModal(
    adata,
    device=device,
    epochs=1000,
    fold=fold,
    image_use=True,
    if_img=True,
)
adata = model.train()
adata = clustering(adata, n_clusters=n_clusters, key="emb", add_key="SpatialModal")
```

## Key Parameters

| Parameter | Description |
|---|---|
| `device` | CPU or GPU device used for training. |
| `epochs` | Number of training epochs. The default setting used in the tutorials is 1000. |
| `learning_rate` | Learning rate for model optimization. |
| `weight_decay` | Weight decay used by AdamW. |
| `random_seed` | Random seed for reproducibility. |
| `fold` | Path to the sample directory. |
| `image_use` | Whether to use histological image features. |
| `if_img` | Whether to extract image features from raw tissue images. If `False`, the model reads `image_feat.csv` when `image_use=True`. |
| `img_size` | Size of image patches cropped around each spot. |
| `target_size` | Resized image patch size for the image encoder. |
| `modals` | Use `"2D"` for the built-in spatial graph construction. |
| `graph` | Optional precomputed graph for customized graph construction. |

## Image Feature Usage

There are three common settings:

| Setting | Usage |
|---|---|
| `image_use=True, if_img=True` | Extract image features from histological images and save them as `image_feat.csv`. |
| `image_use=True, if_img=False` | Read precomputed image features from `image_feat.csv`. |
| `image_use=False` | Run SpatialModal without histological image features; gene features are used for compatibility. |

## Outputs

After `model.train()`, the returned `AnnData` object contains:

| Field | Description |
|---|---|
| `adata.obsm["emb"]` | Learned SpatialModal embedding. |
| `adata.obsm["rec_gene"]` | Reconstructed gene expression representation. |
| `adata.obs["SpatialModal"]` | Spatial domain labels after running the clustering utility. |

The number of clusters should be specified according to the biological question. When ground-truth labels are available for benchmarking, the tutorials set `n_clusters` to the number of ground-truth categories.

## Recommended Order

New users are encouraged to start with `DLPFC.ipynb`, which demonstrates the basic 10x Visium workflow. The other notebooks follow the same model interface and highlight applications to additional tissues and biological settings.
