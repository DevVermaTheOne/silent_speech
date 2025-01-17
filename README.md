# Voicing Silent Speech

This repository contains code for synthesizing speech audio from silently mouthed words captured with electromyography (EMG).
It is the official repository for the papers [Digital Voicing of Silent Speech](https://aclanthology.org/2020.emnlp-main.445.pdf) at EMNLP 2020, [An Improved Model for Voicing Silent Speech](https://aclanthology.org/2021.acl-short.23.pdf) at ACL 2021, and the dissertation [Voicing Silent Speech](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2022/EECS-2022-68.pdf).
The current commit contains only the most recent model, but the versions from prior papers can be found in the commit history.
On an ASR-based open vocabulary evaluation, the latest model achieves a WER of approximately 36%.

## Data

The EMG and audio data can be downloaded from <https://doi.org/10.5281/zenodo.4064408>.  The scripts expect the data to be located in a `emg_data` subdirectory by default, but the location can be overridden with flags (see the top of `read_emg.py`).

Force-aligned phonemes from the Montreal Forced Aligner can be downloaded from <https://github.com/dgaddy/silent_speech_alignments/raw/main/text_alignments.tar.gz>.
By default, this data is expected to be in a subdirectory `text_alignments`.
Note that there will not be an exception if the directory is not found, but logged phoneme prediction accuracies reporting 100% is a sign that the directory has not been loaded correctly.

## Environment Setup

This code requires Python 3.6 or later.
We strongly recommend running in a new Anaconda environment.

First we will do some conda installs.  Your environment must use CUDA 10.1 exactly, since DeepSpeech was compiled with this version.
```
conda install pytorch==1.7.1 torchaudio==0.7.2 cudatoolkit=10.1 -c pytorch
conda install libsndfile=1.0.28 -c conda-forge
```

Pull HiFi-GAN into the `hifi_gan` folder (this replaces the WaveNet vocoder that was used in earlier versions).
```
git clone https://github.com/jik876/hifi-gan.git hifi_gan
```

The rest of the required packages can be installed with pip.
```
pip install absl-py librosa soundfile matplotlib scipy numba jiwer unidecode deepspeech==0.8.2 praat-textgrids
```

Download pre-trained DeepSpeech model files.  It is important that you use DeepSpeech version 0.7.0 model files to maintain consistency of evaluation.  Note that the DeepSpeech pip package we recommend is version 0.8.2 (which uses a more up-to-date CUDA), but this is compatible with version 0.7.x model files.
```
curl -LO https://github.com/mozilla/DeepSpeech/releases/download/v0.7.0/deepspeech-0.7.0-models.pbmm
curl -LO https://github.com/mozilla/DeepSpeech/releases/download/v0.7.0/deepspeech-0.7.0-models.scorer
```

(Optional) Training will be faster if you re-run the audio cleaning, which will save re-sampled audio so it doesn't have to be re-sampled every training run.
```
python data_collection/clean_audio.py emg_data/nonparallel_data/* emg_data/silent_parallel_data/* emg_data/voiced_parallel_data/*
```

## Pre-trained Models

Pre-trained models for the vocoder and transduction model are available at
<https://doi.org/10.5281/zenodo.6747411>.

## Running

To train an EMG to speech feature transduction model, use
```
python transduction_model.py --hifigan_checkpoint hifigan_finetuned/checkpoint --output_directory "./models/transduction_model/"
```
where `hifigan_finetuned/checkpoint` is a trained HiFi-GAN generator model (optional).
At the end of training, an ASR evaluation will be run on the validation set if a HiFi-GAN model is provided.

To evaluate a model on the test set, use
```
python evaluate.py --models ./models/transduction_model/model.pt --hifigan_checkpoint hifigan_finetuned/checkpoint --output_directory evaluation_output
```

By default, the scripts now use a larger validation set than was used in the original EMNLP 2020 paper, since the small size of the original set gave WER evaluations a high variance.  If you want to use the original validation set you can add the flag `--testset_file testset_origdev.json`.

## HiFi-GAN Training

The HiFi-GAN model is fine-tuned from a multi-speaker model to the voice of this dataset.  Spectrograms predicted from the transduction model are used as input for fine-tuning instead of gold spectrograms.  To generate the files needed for HiFi-GAN fine-tuning, run the following with a trained model checkpoint:
```
python make_vocoder_trainset.py --model ./models/transduction_model/model.pt --output_directory hifigan_training_files
```
The resulting files can be used for fine-tuning using the instructions in the hifi-gan repository.
The pre-trained model was fine-tuned for 75,000 steps, starting from the `UNIVERSAL_V1` model provided by the HiFi-GAN repository.
Although the HiFi-GAN is technically fine-tuned for the output of a specific transduction model, we found it to transfer quite well and shared a single HiFi-GAN for most experiments.
