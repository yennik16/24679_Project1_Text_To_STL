# 24679_Project1_Text_To_STL
CMU 24679 Project 1. Program to convert natural language into basic STL models

Describe a simple mechanical part in plain English and get a validated, watertight, 3D-printable STL.

> *"a box 6 by 4 by 3 inches with 1/8" walls. round the corners 1/4", add a divider down the middle and four M5 holes in the base 1" from the sides"*

Supported parts: **pipe gaskets, washers/spacers, L-brackets, rectangular containers and cylindrical containers**. Brackets and containers
can also have **holes** (single, rows or grids, including screw sizes such as "fits an M5 bolt"), **chamfers and fillets**, **dividers** and
**cutouts**. CMU Project 1.

# Interface
**GUI:** [`notebooks/Text_to_STL_App.ipynb`](notebooks/Text_to_STL_App.ipynb)
  *Open the notebook in Colab and choose **Runtime ▸ Run all**. No uploads, sign-ins or training: it downloads the trained models and dataset from
  Hugging Face and the app appears at the bottom after a few minutes (on the T4 GPU it requests automatically). Type a description of a part,
  including any features, and press **Read description**. Every extracted value appears in a colour-coded, editable table showing where it came
  from: found in your text, not found in your text, a catalog or standard size, a default, missing, or changed by you. Edit anything, add or
  remove features, then press **Build part** to get a rotatable 3D preview and a downloadable STL in millimetres. Values that would make an
  impossible part are refused with the reason, so no broken STL is ever offered. Things to try: a gasket for a 6 inch flange on a 2 3/8" pipe with
  4 bolt holes; an L bracket with 3 inch legs, 1.5 wide, 1/8 thick, with two holes for 1/4 inch bolts in leg A; a 4 inch cylindrical cup, 5 tall,
  with three holes in the floor and a rounded rim.*

The full pipeline (data processing, training, evaluation and publishing) is in
[`notebooks/Text_to_STL_Pipeline.ipynb`](notebooks/Text_to_STL_Pipeline.ipynb). It reuses the published
models when they match its data and settings; set `REUSE_SAVED_MODELS = False` to retrain (about 30-40 minutes on a T4).

# Models
- **Stage 1 - part-type classifier (trained from scratch):** [Model Card](https://huggingface.co/yennik16/text-to-stl-part-classifier)
  *Decides which of the five part types a description is about. A bag-of-words neural network built in PyTorch and trained from random
  weights: numbers are replaced by a placeholder and request-framing words ("I want", "make me") are removed, then learned embeddings of the
  words and word pairs are averaged and passed through a small two-layer network (48-dimensional embeddings, 64 hidden units, dropout 0.3).
  Trained on 1,323 descriptions (323 real, 1,000 synthetic) with AdamW (learning rate 5e-3) for 80 epochs, keeping the epoch with the best
  validation accuracy. Its confidence is shown in the app, which lists alternatives when it is below 85%. Intended only for these five part
  families.*
- **Stage 2 - parameter extractor (fine-tuned):** [Model Card](https://huggingface.co/yennik16/text-to-stl-parameter-extractor)
  *Reads the description and writes the part's dimensions as JSON in one standard schema (decimal inches). A LoRA adapter (rank 16,
  alpha 32) on Qwen2.5-1.5B-Instruct, trained on 823 descriptions (323 real, 500 augmented) for 5 epochs with AdamW (learning rate 2e-4,
  effective batch 8), with the loss on the JSON answer only; the epoch with the lowest validation loss is kept. Decoding is greedy, so the same
  input always gives the same output. Every value is checked by rule-based validation before any CAD is built.*
- **Stage 3 - feature editor (off the shelf):** [Model Card](https://huggingface.co/yennik16/text-to-stl-feature-editor)
  *Turns feature requests (holes, chamfers, fillets, dividers, cutouts) into JSON. Qwen2.5-1.5B-Instruct exactly as released, with no
  training: each request is sent with the feature schema, the part's dimensions and the 6 most similar solved requests, chosen by TF-IDF
  similarity (numbers masked) from a 589-request example bank. Supporting new phrasings means adding examples rather than retraining.*

Evaluation results on the held-out test sets are on each model card.

# Data
- **Dataset:** [Dataset Link](https://huggingface.co/datasets/yennik16/text-to-stl-parts)
  *501 manually created part descriptions (gaskets 134, washers 97, L-brackets 92, rectangular containers 89, cylindrical containers 89).
  Gasket, washer and bracket dimensions come from McMaster-Carr catalog listings and container dimensions from measured parts; every description
  was written by our team. An automated audit checks that each labelled number appears in its own description, and the data is split
  323 / 91 / 87 (train / validation / test), grouped by part size so test sizes are unseen. Stored separately and used for training only:
  500 augmented descriptions and 500 classifier phrase variations. Also included: 682 feature requests, the example bank and test set for
  Stage 3. All synthetic data was written or generated with an AI assistant. License: CC BY 4.0.*


# How it works

1. **Preprocess and split:** fractions, number words, feet and millimetres become decimal inches, and the request is split into the part
   description and the feature description.
2. **Classify, extract, edit:** Stage 1 picks the part type, Stage 2 extracts the dimensions, and Stage 3 reads the features.
3. **Validate and repair:** flags values not found in the text, snaps near-catalog sizes, fills defaults (centred holes, safe edge sizes,
   screw-size clearance holes) and checks the geometry: positive sizes, holes inside walls, edge sizes limited by wall thickness, clearances.
4. **Review and build:** the user edits the colour-coded table; CadQuery builds the part and only a single valid, watertight solid is offered.

<img width="508" height="276" alt="Screenshot 2026-10-06 221343" src="https://github.com/user-attachments/assets/beaf9620-5c9d-4349-9024-8e6f677263ad" />
<img width="488" height="461" alt="Screenshot 2026-10-06 221449" src="https://github.com/user-attachments/assets/e615ef31-f3eb-404a-8d25-8b0fd07547bb" />




# Repository layout
```
notebooks/Text_to_STL_App.ipynb       the app: loads the published models and runs the interface
notebooks/Text_to_STL_Pipeline.ipynb  data processing, training, evaluation, the app and publishing
data/                                 the manually created dataset and the synthetic feature requests
docs/figures/                         pipeline diagrams
```

# Limitations
- Five part families only; features for brackets and containers only.
- Geometry only: no material, pressure, temperature or load information. Parts for sealing or load-bearing use need engineering review.
- The app runs inside a Colab session and stops when the session ends. On a CPU runtime, model responses take tens of seconds to minutes.
- The feature requests are synthetic, so they may not reflect how real users phrase requests.

# License
Code and models: Apache 2.0. Data: CC BY 4.0.
