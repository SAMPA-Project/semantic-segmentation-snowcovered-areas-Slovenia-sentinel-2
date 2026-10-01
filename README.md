# Semantic Segmentation of Snowcovered Areas in Slovenia Based on Sentinel-2 Imagery

---

This repository includes:
- The abstract of the paper presented at the 'GIS v Sloveniji 18' conference
- A description of the proposed snow segmentation method, the datasets and the experiments
- Results of the k-fold cross-validation and of testing on an independent test set
- The complete codebase for dataset creation, model training and evaluation
- Trained models and a demo notebook for running inference on Sentinel-2 imagery

---

### Information regarding the paper

- **Title (Slovenian):** Semantična segmentacija zasneženih površin v Sloveniji na osnovi posnetkov Sentinel-2
- **Title (English):** Semantic Segmentation of Snowcovered Areas in Slovenia Based on Sentinel-2 Imagery
- **Authors:** Domen Kavran¹, Mihaela Triglav Čekada², Niko Lukač¹
  - ¹ University of Maribor, Faculty of Electrical Engineering and Computer Science (UM FERI), Maribor, Slovenia
  - ² Geodetic Institute of Slovenia, Ljubljana, Slovenia
- **Published in:** GIS v Sloveniji 18, Pametni prostor, 2026
- **Language:** Slovenian
- **Dataset:** [Global Sentinel-2 Snow Semantic Segmentation Dataset with Dynamic World Labels](https://doi.org/10.5281/zenodo.19482842) (Zenodo)

### Abstract

The paper focuses on semantic segmentation of snow-covered surfaces in Slovenia based on multispectral Sentinel-2 satellite imagery. The proposed semantic segmentation method relies on deep learning models to generate a binary snow map, with the U-Net architecture and the GASSL foundation model evaluated on a custom dataset covering 8,300 areas. The proposed method was tested on satellite imagery of the Martuljek and Prisojnik mountains, where permanent snowfields thrive through whole summer. The results show that using all 13 spectral bands improves semantic segmentation performance, with the GASSL foundation model achieving the highest mean Intersection Over Union (mIoU) of 0.4027.

**Keywords:** semantic segmentation, snow, snowfield, Sentinel-2, deep learning, foundation model

Predictions for all test images and thresholds are available in `trained_model/all/<model>/CrossEntropyLoss/predictions/` (`probabilities/` and `labels/`).

### Source code

Requirements:
- Python and Anaconda
- PyTorch, PyTorch Lightning and segmentation-models-pytorch
- Google Earth Engine API and geemap (only for dataset creation)
- rasterio, fiona, OpenCV, NumPy, pandas, scikit-learn and matplotlib

Code structure:

- `1_download_data_and_create_dataset/`: creation of the training dataset:
    - **`1_build_snow_dataset.ipynb`**
      Samples tiles from `bounding_boxes.json`, downloads cloud-masked Sentinel-2 images and Dynamic World labels/probabilities via Google Earth Engine, derives binary snow masks and writes `dataset/manifest.csv`.
    - **`2_filter_validate_and_delete_invalid_tiles.ipynb`**
      Validates all tiles (missing files, raster shapes, contrast, label quality), deletes invalid ones and removes them from the manifest.
    - **`3_preview_dataset.ipynb`**
      Visualizes random tiles: Sentinel-2 imagery, Dynamic World labels, label probabilities and snow masks.
    - `bounding_boxes.json`: seed bounding boxes and collection periods of the sampled locations.

- `2_download_test_data_and_create_test_dataset/`: creation of the test dataset:
    - **`1_build_test_dataset.ipynb`**
      Downloads Sentinel-2 images matching the footprints and dates of the reference orthophotos of the Martuljek mountains and Prisojnik. It also exports the corresponding Dynamic World snow masks.
    - **`2_filter_validate_and_delete_invalid_test_tiles.ipynb`**
      Validates and quarantines invalid test images and builds `test_dataset/manifest.csv`.
    - **`3_preview_test_dataset.ipynb`**
      Visualizes the test images together with their snow masks.

- `3_train_state_of_the_art/`: model training and evaluation:
    - **`1_k-fold_validation.ipynb`**
      5-fold cross-validation of the selected model (`U-Net` or `GASSL-basic`) and band configuration (`rgb` or all 13 bands).
    - **`2_train_model.ipynb`**
      Trains the final model on the whole training dataset and saves the checkpoint (`snow-best.ckpt`) and per-band normalization statistics (`channel_stats.json`). It then runs inference on the test dataset and computes IoU/mIoU and F1 for each threshold *τ*.
    - `my_code/`: library wrapping the models into a unified segmentation pipeline:
        - `backbones.py`: pretrained encoders (`MoCoResNet50Backbone` for GASSL, as well as SeCo, SatMAE and TOV). Each loads its pretrained checkpoint, removes the classification head and expands the first layer to the required number of input channels.
        - `model.py`: `create_state_of_the_art_model()` builds the encoder–decoder model (U-Net, or encoder + UPerNet/TransUNet decoder). `MyModel` is a PyTorch Lightning module handling input resizing, normalization, augmentation, loss, optimization and mIoU logging.
        - `utils.py`, `dataset.py`: metric helpers (mIoU) and generic data-loading utilities.
    - `methods/`: original repositories of the remote sensing foundation models (GASSL, SeCo, SatMAE, TOV). Pretrained checkpoints must be downloaded into each method's `weights/` folder before training.
    - `decoders/TransUNet/`: original TransUNet repository; its `DecoderCup` is used as the decoder for ViT-based encoders.
    - `dataloader_wrapper.py`: thin wrapper around PyTorch's `DataLoader`.

- `trained_model/all/<model>/CrossEntropyLoss/`: trained models using all 13 spectral bands (`GASSL-basic`, `Unet`), each with:
    - `snow-best.ckpt`: model weights
    - `channel_stats.json`: per-band means and standard deviations used for input standardization
    - `predictions/`: predicted snow probabilities and masks for the test dataset

  > **Note:** The model weights `trained_model/all/GASSL-basic/CrossEntropyLoss/snow-best.ckpt` and `trained_model/all/Unet/CrossEntropyLoss/snow-best.ckpt` are not included in this repository, because the files are too large. If you would like to obtain them, please contact us (domen.kavran1@um.si).

- **`demo_snow_segmentation.ipynb`**: demo notebook. It loads a trained model (`ARCH = "GASSL-basic"` or `"Unet"`), runs it on a single 13-band Sentinel-2 GeoTIFF (`INPUT_TIF`) and plots the following:
    - the true-color composite
    - the snow probability map
    - the binary snow mask (`THRESHOLD`, default *τ* = 0.25)
    - the reference mask, if available

  Run it from the repository root.

### Acknowledgements

The research work was co-funded by the Slovenian Research and Innovation Agency (ARIS) within the basic research project No. J7-50095 and the research programme No. P2-0041. We thank the EU Copernicus programme for the Sentinel-2 imagery.
