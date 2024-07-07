# ASR For Egyptian Dialect

This repo is a submission for Speech Squad team for MTC-AIC2 Phase 1 challenge. The repo contains the code for our experiments conducted to train an ASR model for the Egyptian dialect. We reached a score of `17.406720` (the lower the better) on the test set ranking 7th on the leaderboard. Our approach provides **real-time speech recognition** reaching an average of 62ms without preprocessing and 140ms with preprocessing per audio sample. Our model perform competitively compared to those having better score but with much higher latency. The model is based on the FastConformer architecture and is trained using the CTC loss function. The model is pretrained on a synthetic dataset generated using GPT-4o and OpenAI TTS and fine-tuned on the real dataset provided by the competition.

## Table of Contents
- [ASR For Egyptian Dialect](#asr-for-egyptian-dialect)
  * [Installation Requirements](#installation-guide)
    + [Clone the repository](#clone-the-repository)
  * [Dataset](#dataset)
    + [Real](#real)
    + [Synthetic](#synthetic)
    + [Data Filteration](#data-filteration)
  * [Tokenizer](#tokenizer)
  * [Preprocessing](#preprocessing)
  * [Training](#training)
  * [Inference](#inference)
    + [Example Usage](#example-usage)
  * [Example Usage for Other Functionalities](#example-usage-for-other-functionalities)
    + [Generate Speaker Embeddings](#generate-speaker-embeddings)
    + [Generate Manifest File](#generate-manifest-file)
    + [Generate Labels](#generate-labels)
  * [Contributors](#contributors)
  * [Supervisor](#supervisor)
  * [References](#references)


## Installation Requirements
The provided scripts runs on Ubuntu and requires Python 3.10.

### Clone the repository

```bash
git clone https://github.com/AbdelrhmanElnenaey/ASR_for_egyptian_dialect.git
```
## Dataset

### Real
The competition organizers provided a relatively small dataset containing ~50,000 samples of speech-transcript paired data which will be used for training and provided another dataset for adaptation consisting of ~2200 samples. The dataset is in the form of a CSV file containing the following columns:
- `audio`: the name of the audio file
- `transcript`: the transcript of the audio file

### Synthetic

Since the dataset provided by the challenge organizers is very small, we decided to generate a synthetic dataset.

We utilized GPT-4o provided by OpenAI to generate a synthetic Egyptian text corpus using through API. These sentences generated by GPT-4o LLM later goes through OpenAI TTS model through another API to produce synthetic data that looks like the real samples. We sample the speed and speaker randomly to result in diverse dataset. The synthetic dataset contains roughly 30,000 sample of speech-transcript paired data. Since the synthetic data is not perfect, it is used in the pretraining phase to improve the model's performance before fine-tuning on the real data provided by the competition.

This data help the model capture plain Arabic phonemes in pretraining phase before finetuning it. We publicly released the synthetic dataset and can be found [Google Drive](https://drive.google.com/drive/folders/1jRb0X9_O6p6UOpIyZ2NoxF1_mjYbty4M?usp=sharing).

[Play First Audio Sample](assets/00d5ab4304518f11.wav)

[Play Second Audio Sample](assets/00b906eba080054a.wav)


### Data Filteration
To standarize the data, we normalized the transcripts. We follow the pipeline introduced by [ArTST: Arabic Text and Speech Transformer](https://arxiv.org/abs/2310.16621). All punctuation marks were removed with the exception of `@` and `%`. Additionally, all diacritics were removed, and Indo-Arabic numerals were replaced with Arabic numerals to ensure uniformity. The vocabulary is comprised of individual Arabic alphabets, numerals, and select English characters from the training dataset, in addition to some special characters like `@` and `%`. For speech data, we standardized the sampling rate to be 16 kHz across all collected datasets. An updated version of the `csv` files can be found in the `data` directory. Normalized transcripts were filtered to exclude foreign characters and only include the allowed characters in the `data/allowed_chars.txt` file.

- To normalize the transcripts, run the following command:

```bash
python utils/transcript_normalization.py --input_file=data/train.csv --output_file=data/train_artst_normalized.csv
python utils/transcript_normalization.py --input_file=data/adapt.csv --output_file=data/adapt_artst_normalized.csv
```

- To filter the normalized transcripts which execlude foreign characters or charachters not found in `data/allowed_chars.txt` file, run the following command:

```bash
python utils/execlude_foreign_characters.py --input_file=data/train_artst_normalized.csv --output_file=data/train_artst_normalized_filtered.csv --char_file=allowed_chars.txt  
python utils/execlude_foreign_characters.py --input_file=data/adapt_artst_normalized.csv --output_file=data/adapt_artst_normalized_filtered.csv --char_file=allowed_chars.txt  
```

## Tokenizer
We use a tokenizer to convert the transcripts into tokens. We use the `bpe` tokenizer obtained from `process_asr_text_tokenizer.py` script provided by NVIDIA NeMo. The tokenizer is trained on the training dataset provided by the competition and is saved in the `tokenizer` folder with a vocab size of `256`. 

We explored different tokenizer obtained through `SentencePiece` with a vocab size of 32,000 where we gathered Egyptian text corpus on the internet (Reddit, Twitter, EL-youm Elsabe', and Wikipedia) and trained the tokenizer on 63 million egyptian word (provided on demand due to large size). However, the tokenizer did not perform well on the dataset provided by the competition.

We believe that there is room for improvement in the tokenizer where we plan to explore different tokenizers and vocab sizes in the future.

## Preprocessing

A key component of our preprocessing pipeline is the optional use of [Cleanunet](https://github.com/NVIDIA/CleanUNet) provided by NVIDIA, a model designed for cleaning and enhancing the audio before running speech recognition. The model enhaces the quality of the audio by removing noise and enhancing the speech. The 

## Training
We provide a bash script to train the ASR model using the FastConformer architecture. The script is based on the NVIDIA NeMo toolkit and is provided in the `train.sh` file. The script installs the needed packages and trains the model using the CTC loss function and the Adam optimizer. The model is trained on the synthetic dataset and fine-tuned on the real dataset provided by the competition. You should provide the path to the training and adaptation datasets  and their respective csv files.

### Example Usage
N.B., Exclude comments when running the script.
```bash
cd ASR_for_egyptian_dialect
chmod +x train.sh
./train.sh "data/train.csv" \   ## csv file containing the training data filename and transcript
            "data/train" \      ## directory containing the training data
            "data/adapt.csv" \  ## csv file containing the adaptation data filename and transcript
            "data/adapt"        ## directory containing the adaptation data
```

## Inference
To replicate our inference results, `inference.py` is provided.

The script downlads the checkpoints from google drive, transcribes audio files found in `data_dir` using `enhancement` and `asr` models and outputs the results in `csv format`.

The checkpoints can be found [here](https://drive.google.com/drive/u/6/folders/11-oGdeyNT6pFJaf-_BqVE4PIUoqB2acU).
### Example Usage
N.B., Exclude comments when running the script.
```bash
cd ASR_for_egyptian_dialect
chmod +x infere.sh
./infere.sh asr_model_.ckpt \  ## asr model checkpoint path, if not found will be downloaded
            cleanunet.pt \     ## speech enhancement model checkpoint path, if not found will be downloaded
            /content/test \    ## test data directory (data_dir)
            results.csv        ## output file containing transcription in csv format
```
For more information, use `inference.py -h`.

## Example Usage for Other Functionalities

### Generate Speaker Embeddings
```bash
python utils/generate_speaker_embedding.py --input_dir=data/train --output_dir=data/speaker_embedding/train
python utils/generate_speaker_embedding.py --input_dir=data/adapt --output_dir=data/speaker_embedding/adapt
```

### Generate Manifest File
```bash
python utils/generate_manifest_file.py --audio_dir=data/train --embedding_dir=data/speaker_embedding/train --output_file=data/train.tsv --audio_csv=data/train_artst_normalized_filtered.csv
python utils/generate_manifest_file.py --audio_dir=data/adapt --embedding_dir=data/speaker_embedding/adapt --output_file=data/adapt.tsv --audio_csv=data/adapt_artst_normalized_filtered.csv
```

### Generate Labels
```bash
python utils/generate_label_file.py --tsv_file=data/train.tsv --csv_file=data/train_artst_normalized_filtered.csv --output_file=data/train_labels.txt
python utils/generate_label_file.py --tsv_file=data/adapt.tsv --csv_file=data/adapt_artst_normalized_filtered.csv --output_file=data/labels/adapt.txt
```

## Contributors
- [Yousef Kotp](https://github.com/yousefkotp)
- [Karim Alaa](https://github.com/Karim19Alaa)
- [Abdelrahman Elnenaey](https://github.com/AbdelrhmanElnenaey)
- [Rana Barakat](https://github.com/ranabarakat)
- [Louai Zahran](https://github.com/LouaiZahran)

## Supervisor
- [Ismail El-Yamany](https://github.com/IsmailElYamany)

## References
- [FastConformer: Efficient and Accurate Conformer ASR with Low Latency](https://arxiv.org/abs/2305.05084)
- [Conformer: Convolution-augmented Transformer for Speech Recognition](https://arxiv.org/abs/2005.08100)
- [wav2vec 2.0: A Framework for Self-Supervised Learning of Speech Representations](https://arxiv.org/abs/2006.11477)
- [Arabic - Egyptian comparable Wikipedia corpus](https://www.kaggle.com/datasets/mksaad/arb-egy-cmp-corpus)
- [ElevenLabs: Text to Speech & AI Voice Generator](https://elevenlabs.io/)
- [GPT-4o](https://openai.com/index/hello-gpt-4o/)
- [OpenAI TTS](https://platform.openai.com/docs/guides/text-to-speech)
