# Learning from Incomplete Keyword Metadata: Image Tagging for Historical Photo Archives

Data and results accompanying the paper. The repository contains the ground-truth keywords for three historical photo archives, references to the images (the images themselves are not redistributed), the cross-validation folds, the predictions of every model, and all result tables.

## Contents

```text
data/
  germancolonial/   images.csv, keywords.jsonl, vocabulary.txt, folds.json, preprocessing.jsonl
  sagarra/          images.csv, keywords.jsonl, vocabulary.txt, folds.json
  fairphotos/       images.csv, keywords.jsonl, vocabulary.txt, folds.json
results/
  tables/           all result tables as CSV
  evaluations/      per-fold and aggregate metrics for each archive and model (JSON)
  predictions/      held-out predictions for every image (gzipped JSONL)
  datascale/        data-scaling experiment (evaluations and predictions)
  ablations/        single-run evaluations over all GermanColonial images
```

## Datasets

| Archive | Images | Concepts | Keywords per image |
|---|---:|---:|---:|
| GermanColonial | 37,843 | 3,005 | 3.0 |
| Sagarra | 6,252 | 418 | 2.6 |
| FAIR Photos | 17,929 | 214 | 1.1 |

**GermanColonial.** Photographs from German colonial contexts, held by the University Library Frankfurt ([collection page](https://www.ub.uni-frankfurt.de/afrika/bildsammlung_en.html)). Images are identified by their URN, which resolves through `https://nbn-resolving.org/<URN>`. The keywords are English subject keywords from the collection's metadata. The photographs are mounted on cardboard, so we cropped each image to the photograph and corrected its rotation before use. `preprocessing.jsonl` gives, per image, the rotation angle and the detected regions used for cropping (coordinates in percent of the image size).

**Sagarra.** Photographs of Catalonia by Josep Maria Sagarra, retrieved from [Europeana](https://pro.europeana.eu/data/over-6-700-photographs-from-catalan-photographer-josep-maria-sagarra). `images.csv` links each image to its Europeana record. Keywords are the English labels of the Getty AAT concepts in each record's subject metadata.

**FAIR Photos.** A Dutch press-photo archive ([dataset](https://doi.org/10.17026/SS/7VD3ME)). `images.csv` gives each photograph's persistent handle URI. We use the pre-1960 photographs that have at least one Getty AAT concept, and the English AAT labels as keywords.

### File formats

- `images.csv`: one row per image, with the image identifier and, for Sagarra and FAIR Photos, a link to the source record.
- `keywords.jsonl`: one JSON object per image, `{"id": ..., "keywords": [...]}`. These keywords are the ground truth for all evaluations.
- `vocabulary.txt`: the archive's concept vocabulary, one concept per line. Matching ignores case, so keywords that differ only in case are listed once.
- `folds.json`: the five cross-validation folds (lists of image identifiers, random split with seed 42). Each image is in exactly one test fold.
- `predictions/<archive>/<model>.jsonl.gz`: one JSON object per image, `{"urn": ..., "predictions": [...]}`. For fine-tuned models, each image's prediction comes from the model trained on the other four folds.

## Models

| Name in files | Description |
|---|---|
| `zeroshot_ramplus` | RAM++ (Swin-L, 14M checkpoint), built-in tag list of 4,585 tags, default thresholds |
| `zeroshot_siglip2-so400m` | SigLIP2 So400m (`google/siglip2-so400m-patch14-384`), top 10 archive concepts by similarity to the prompt "This is a photo of {concept}." |
| `zeroshot_qwen3vl` | Qwen3-VL-8B-Instruct, greedy decoding, prompted for up to 10 English tags |
| `finetuned_siglip2-base_<loss>` | SigLIP2-base vision encoder (`google/siglip2-base-patch16-224`) with LoRA and a linear head over the archive vocabulary, top 10 concepts |
| `finetuned_ramplus-swinl_<loss>` | RAM++ Swin-L image encoder with LoRA and a linear head over the archive vocabulary, top 10 concepts |

Losses: `an` (assume negative, binary cross-entropy), `splc` (self-paced loss correction), `hill` (Hill loss), `dwn` (negative terms down-weighted by a constant factor of 0.5), `supportpu` (negative terms down-weighted per concept, more strongly for concepts with few positives in the training fold). All losses were run on GermanColonial. Sagarra and FAIR Photos were run with `an` and `splc`.

## Metrics

A ground-truth keyword counts as predicted correctly if at least one predicted tag matches it. An **exact** match requires identical strings (ignoring case and surrounding whitespace). A **semantic** match requires a cosine similarity of at least 0.80 between the two strings, embedded with `BAAI/bge-large-en-v1.5`.

- **Concept recall**: fraction of all keyword assignments predicted correctly (micro-averaged recall).
- **Image hit rate**: fraction of images with at least one keyword predicted correctly.
- **Vocabulary coverage**: fraction of distinct concepts predicted correctly for at least one image.

All values are percentages, given as mean ± standard deviation over the five folds.

## Results

### Concept recall

| Archive | Model | Exact | Semantic |
|---|---|---:|---:|
| GermanColonial | `zeroshot_ramplus` | 12.31 ± 0.16 | 24.89 ± 0.18 |
| GermanColonial | `zeroshot_siglip2-so400m` | 8.39 ± 0.27 | 15.15 ± 0.40 |
| GermanColonial | `zeroshot_qwen3vl` | 6.82 ± 0.18 | 22.38 ± 0.43 |
| GermanColonial | `finetuned_siglip2-base_an` | 46.29 ± 0.78 | 49.87 ± 0.75 |
| GermanColonial | `finetuned_siglip2-base_splc` | 51.79 ± 1.23 | 55.43 ± 1.31 |
| GermanColonial | `finetuned_siglip2-base_hill` | 50.56 ± 1.05 | 54.25 ± 0.98 |
| GermanColonial | `finetuned_siglip2-base_dwn` | 47.22 ± 1.24 | 50.98 ± 1.31 |
| GermanColonial | `finetuned_siglip2-base_supportpu` | 46.47 ± 1.02 | 50.15 ± 0.98 |
| GermanColonial | `finetuned_ramplus-swinl_an` | 63.69 ± 0.38 | 66.85 ± 0.25 |
| GermanColonial | `finetuned_ramplus-swinl_splc` | 65.66 ± 0.50 | 68.86 ± 0.50 |
| Sagarra | `zeroshot_ramplus` * | 2.28 ± 0.21 | 22.58 ± 0.43 |
| Sagarra | `zeroshot_siglip2-so400m` * | 36.77 ± 0.83 | 39.56 ± 0.76 |
| Sagarra | `zeroshot_qwen3vl` * | 3.47 ± 0.21 | 29.41 ± 0.84 |
| Sagarra | `finetuned_siglip2-base_an` * | 76.23 ± 0.44 | 77.36 ± 0.51 |
| Sagarra | `finetuned_siglip2-base_splc` * | 80.88 ± 0.44 | 81.78 ± 0.37 |
| Sagarra | `finetuned_ramplus-swinl_an` | 82.30 ± 0.28 | 83.21 ± 0.37 |
| Sagarra | `finetuned_ramplus-swinl_splc` | 88.10 ± 0.44 | 88.73 ± 0.35 |
| FAIR Photos | `zeroshot_ramplus` * | 2.85 ± 0.29 | 19.74 ± 0.87 |
| FAIR Photos | `zeroshot_siglip2-so400m` * | 65.21 ± 0.57 | 65.64 ± 0.62 |
| FAIR Photos | `zeroshot_qwen3vl` * | 5.62 ± 0.40 | 27.42 ± 0.50 |
| FAIR Photos | `finetuned_siglip2-base_an` * | 82.67 ± 0.78 | 82.96 ± 0.84 |
| FAIR Photos | `finetuned_siglip2-base_splc` * | 88.10 ± 1.24 | 88.20 ± 1.26 |
| FAIR Photos | `finetuned_ramplus-swinl_an` | 93.82 ± 0.50 | 93.96 ± 0.49 |
| FAIR Photos | `finetuned_ramplus-swinl_splc` | 95.16 ± 0.26 | 95.26 ± 0.26 |

### Image hit rate

| Archive | Model | Exact | Semantic |
|---|---|---:|---:|
| GermanColonial | `zeroshot_ramplus` | 31.56 ± 0.43 | 53.19 ± 0.61 |
| GermanColonial | `zeroshot_siglip2-so400m` | 22.93 ± 0.71 | 38.05 ± 0.83 |
| GermanColonial | `zeroshot_qwen3vl` | 18.48 ± 0.49 | 47.00 ± 0.76 |
| GermanColonial | `finetuned_siglip2-base_an` | 77.32 ± 0.74 | 80.25 ± 0.54 |
| GermanColonial | `finetuned_siglip2-base_splc` | 81.60 ± 0.84 | 84.23 ± 0.72 |
| GermanColonial | `finetuned_siglip2-base_hill` | 80.87 ± 1.00 | 83.69 ± 0.89 |
| GermanColonial | `finetuned_siglip2-base_dwn` | 78.24 ± 1.18 | 81.20 ± 1.02 |
| GermanColonial | `finetuned_siglip2-base_supportpu` | 77.33 ± 1.13 | 80.30 ± 0.92 |
| GermanColonial | `finetuned_ramplus-swinl_an` | 89.85 ± 0.24 | 91.45 ± 0.23 |
| GermanColonial | `finetuned_ramplus-swinl_splc` | 90.54 ± 0.25 | 92.04 ± 0.19 |
| Sagarra | `zeroshot_ramplus` * | 5.42 ± 0.67 | 47.95 ± 1.09 |
| Sagarra | `zeroshot_siglip2-so400m` * | 69.58 ± 1.65 | 72.07 ± 1.48 |
| Sagarra | `zeroshot_qwen3vl` * | 8.21 ± 0.39 | 58.32 ± 1.75 |
| Sagarra | `finetuned_siglip2-base_an` * | 93.44 ± 0.65 | 93.99 ± 0.52 |
| Sagarra | `finetuned_siglip2-base_splc` * | 94.98 ± 0.49 | 95.38 ± 0.50 |
| Sagarra | `finetuned_ramplus-swinl_an` | 96.77 ± 0.15 | 96.99 ± 0.24 |
| Sagarra | `finetuned_ramplus-swinl_splc` | 97.70 ± 0.35 | 97.90 ± 0.30 |
| FAIR Photos | `zeroshot_ramplus` * | 3.01 ± 0.31 | 20.63 ± 0.88 |
| FAIR Photos | `zeroshot_siglip2-so400m` * | 67.66 ± 0.81 | 68.07 ± 0.86 |
| FAIR Photos | `zeroshot_qwen3vl` * | 5.91 ± 0.38 | 28.72 ± 0.65 |
| FAIR Photos | `finetuned_siglip2-base_an` * | 84.29 ± 1.02 | 84.53 ± 1.09 |
| FAIR Photos | `finetuned_siglip2-base_splc` * | 89.30 ± 1.28 | 89.39 ± 1.30 |
| FAIR Photos | `finetuned_ramplus-swinl_an` | 94.57 ± 0.35 | 94.68 ± 0.36 |
| FAIR Photos | `finetuned_ramplus-swinl_splc` | 95.72 ± 0.16 | 95.83 ± 0.16 |

### Vocabulary coverage

| Archive | Model | Exact | Semantic |
|---|---|---:|---:|
| GermanColonial | `zeroshot_ramplus` | 11.16 ± 0.39 | 24.44 ± 0.58 |
| GermanColonial | `zeroshot_siglip2-so400m` | 23.27 ± 0.41 | 32.92 ± 0.46 |
| GermanColonial | `zeroshot_qwen3vl` | 12.17 ± 0.50 | 32.14 ± 0.12 |
| GermanColonial | `finetuned_siglip2-base_an` | 27.48 ± 0.92 | 32.13 ± 0.77 |
| GermanColonial | `finetuned_siglip2-base_splc` | 38.69 ± 5.00 | 42.90 ± 4.78 |
| GermanColonial | `finetuned_siglip2-base_hill` | 32.46 ± 2.37 | 37.24 ± 2.59 |
| GermanColonial | `finetuned_siglip2-base_dwn` | 27.70 ± 1.44 | 32.37 ± 1.40 |
| GermanColonial | `finetuned_siglip2-base_supportpu` | 35.78 ± 3.61 | 40.28 ± 3.93 |
| GermanColonial | `finetuned_ramplus-swinl_an` | 54.11 ± 1.08 | 58.46 ± 1.01 |
| GermanColonial | `finetuned_ramplus-swinl_splc` | 60.45 ± 1.13 | 64.37 ± 0.82 |
| Sagarra | `zeroshot_ramplus` * | 2.57 ± 0.85 | 33.99 ± 1.76 |
| Sagarra | `zeroshot_siglip2-so400m` * | 59.35 ± 3.68 | 62.47 ± 2.99 |
| Sagarra | `zeroshot_qwen3vl` * | 14.24 ± 0.60 | 51.20 ± 1.14 |
| Sagarra | `finetuned_siglip2-base_an` * | 46.53 ± 3.86 | 49.28 ± 4.15 |
| Sagarra | `finetuned_siglip2-base_splc` * | 59.49 ± 2.77 | 62.38 ± 2.59 |
| Sagarra | `finetuned_ramplus-swinl_an` | 46.20 ± 3.59 | 49.39 ± 3.80 |
| Sagarra | `finetuned_ramplus-swinl_splc` | 70.27 ± 1.97 | 72.54 ± 1.57 |
| FAIR Photos | `zeroshot_ramplus` * | 10.69 ± 2.36 | 33.86 ± 2.55 |
| FAIR Photos | `zeroshot_siglip2-so400m` * | 78.93 ± 2.86 | 80.43 ± 3.09 |
| FAIR Photos | `zeroshot_qwen3vl` * | 16.14 ± 2.25 | 44.11 ± 1.15 |
| FAIR Photos | `finetuned_siglip2-base_an` * | 55.37 ± 4.00 | 56.48 ± 4.22 |
| FAIR Photos | `finetuned_siglip2-base_splc` * | 80.40 ± 3.86 | 80.77 ± 3.73 |
| FAIR Photos | `finetuned_ramplus-swinl_an` | 90.11 ± 1.70 | 90.24 ± 1.87 |
| FAIR Photos | `finetuned_ramplus-swinl_splc` | 94.06 ± 1.24 | 94.18 ± 1.38 |

\* Aggregate values only. The per-fold evaluation file and predictions for this run are not included yet.

### Other tables

- `results/tables/kfold_results_per_fold.csv`: every metric for every fold.
- `results/tables/datascale_germancolonial_fold0.csv`: fine-tuning on 5%, 10%, 25% and 50% of the training images (GermanColonial, fold 0 only).
- `results/tables/ablations_germancolonial_all_images.csv`: zero-shot RAM++ on original vs. cropped images, and zero-shot SigLIP2 base vs. So400m with top 5 vs. top 10 concepts, each evaluated once over all GermanColonial images.

## Licence

The result files are released under the Creative Commons Attribution 4.0 International licence (see `LICENSE`). The keywords and image references are derived from the source collections listed above and remain subject to their terms.

## Citation

The citation will be added after the review period.
