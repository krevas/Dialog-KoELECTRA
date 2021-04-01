[![license](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](https://github.com/skplanetml/Dialog-KoELECTRA/blob/master/LICENSE)


# Dialog-KoELECTRA

<br>

## Introduction

**Dialog-KoELECTRA** is a language model specialized for dialogue. It was trained with 22GB colloquial and written style Korean text data. Dialog-ELECTRA model is made based on the [ELECTRA](https://openreview.net/pdf?id=r1xMH1BtvB) model. ELECTRA is a method for self-supervised language representation learning. It can be used to pre-train transformer networks using relatively little compute. ELECTRA models are trained to distinguish "real" input tokens vs "fake" input tokens generated by another neural network, similar to the discriminator of a [GAN](https://arxiv.org/pdf/1406.2661.pdf). At small scale, ELECTRA achieves strong results even when trained on a single GPU. 

Dialog-KoELECTRA can speed up learning and use less memory by using the mixed precision option during pre-training. When finetuning, parameter optimization is possible by using [NNI](https://github.com/microsoft/nni) option.

<br>

## Released Models

We are initially releasing small version pre-trained model.
The model was trained on Korean text. We hope to release other models, such as base/large models, in the future.

| Model | Layers | Hidden Size | Params | Max<br/>Seq Len | Learning<br/>Rate | Batch Size | Train Steps  |
| :---: | :---: | :---: | :---: | :---:  | :---: | :---:  | :---:  |
| Dialog-KoELECTRA-Small | 12 | 256 | 14M | 128 | 1e-4 | 512 | 700K |

### Download

| Model | Pytorch-Generator | Pytorch-Discriminator | Tensorflow-v1 | ONNX |
| :---: | :---: | :---: | :---: | :---: |
| Dialog-KoELECTRA-Small | [link](https://drive.google.com/file/d/1uaAcnfLKqftmvMG7oB7jt66J1sQ0hyuW/view?usp=sharing) | [link](https://drive.google.com/file/d/1TdINJDUB5Imfbo9JWHlO15HAFsLu7DAo/view?usp=sharing) | [link](https://drive.google.com/file/d/13smn2Bt5Zjxkr7DNIPyGK0nFLErDRhAR/view?usp=sharing) |[link](https://drive.google.com/file/d/1EM0Krkylnhm5j83jPP9uiRa88skc7yTi/view?usp=sharing)|

<br>

## Pre-training

Use `preprocess.py` to preprocess from a raw text. Data preprocessing only removed repetitive characters and Chinese characters. It has the following arguments:

* `--corpus_dir`: A directory containing raw text files.
* `--output_file`: File created after preprocessing.

Then run (for example)
```python
python3 preprocess.py \
    --corpus_dir raw_data_dir \
    --output_file preprocessed_data.txt \
```

---

Use `build_vocab.py` to create a vocabulary file from a raw text or preprocessed data. It has the following arguments:

* `--corpus`: A raw text file or preprocessed file to turn into a vocabulary file.
* `--tokenizer`: a name for the tokenizer such as a wordpiece or mecab_wordpiece (wordpiece by default).
* `--vocab_size`: The number of word in vocabulary (40000 by default).
* `--min_frequency`: The minimum frequency a pair must have to produce a merge operation (3 by default).
* `--limit_alphabet`: The number of initial tokens that can be kept before computing merges (6000 by default).
* `--unused_size`: The number of unused token (500 by default).

Then run (for example)
```python
python3 build_vocab.py \
    --corpus preprocessed_data.txt \
    --tokenizer mecab_wordpiece \
    --vocab_size 40000 \
    --min_frequency 3 \
    --limit_alphabet 6000 \
    --unused_size 500
```

---

Use `build_pretraining_dataset.py` to create a pre-training dataset from a dump of raw text. It has the following arguments:

* `--corpus_dir`: A directory containing raw text files to turn into Dialog-KoELECTRA examples. A text file can contain multiple documents with empty lines separating them.
* `--vocab_file`: File defining the wordpiece vocabulary.
* `--output_dir`: Where to write out Dialog-KoELECTRA examples.
* `--max_seq_length`: The number of tokens per example (128 by default).
* `--num_processes`: If >1 parallelize across multiple processes (1 by default).
* `--blanks-separate-docs`: Whether blank lines indicate document boundaries (True by default).
* `--do-lower-case/--no-lower-case`: Whether to lower case the input text (True by default).
* `--tokenizer_type`: a name for the tokenizer such as a wordpiece or mecab_wordpiece (wordpiece by default).

Then run (for example)
```python
python3 build_pretraining_dataset.py \
    --corpus_dir data/train_data/raw/split_normalize \
    --vocab_file data/vocab/vocab.txt \
    --tokenizer_type wordpiece \
    --output_dir data/train_data/tfrecord/pretrain_tfrecords_len_128_wordpiece_train \
    --max_seq_length 128 \
    --num_processes 8
```

---

Use `run_pretraining.py` to pre-train an Dialog-KoELECTRA model. It has the following arguments:

* `--data_dir`: a directory where pre-training data, model weights, etc. are stored.
* `--model_name`: a name for the model being trained. Model weights will be saved in `<data-dir>/models/<model-name>` by default.
* `--hparams` (optional): a JSON dict or path to a JSON file containing model hyperparameters, data paths, etc. See `configure_pretraining.py` for the supported hyperparameters.
* `--use_tpu` (optional): Option to use tpu when training the model.
* `--mixed_precision` (optional): Option for whether to use mixed precision when training the model.

Then run (for example)
```python
python3 run_pretraining.py \
    --data_dir data/train_data/tfrecord/pretrain_tfrecords_len_128_wordpiece_train \
    --model_name data/ckpt/pretrain_ckpt_len_128_small_wordpiece_train \
    --hparams data/config/small_config_kor_wordpiece_train.json \
    --mixed_precision
```

---

Use `pytorch_convert.py` to convert the tf model to pytorch model. It has the following arguments:

* `--tf_ckpt_path`: a directory where tensorflow checkpoint are stored.
* `--pt_discriminator_path`: Where to write out pytorch discriminator model.
* `--pt_generator_path` (optional): Where to write out pytorch generator model.

Then run (for example)
```python
python3 pytorch_convert.py \
    --tf_ckpt_path model/ckpt/pretrain_ckpt_len_128_small \
    --pt_discriminator_path model/pytorch/dialog-koelectra-small-discriminator \
    --pt_generator_path model/pytorch/dialog-koelectra-small-generator \
```

<br>

## Fine-tuning

Use `run_finetuning.py` to fine-tune and evaluate an Dialog-KoELECTRA model on a downstream NLP task. It expects three arguments:

* `--config_file`: a YAML file containing model hyperparameters, data paths, etc..
* `--nni`: Option for whether to use nni when finetuning the model.

Then run (for example)
```python
python3 run_finetune.py --config_file conf/hate-speech/electra-small.yaml
```

<br>

## Model Performance

Dialog-KoELECTRA shows strong performance in conversational downstream tasks.

|               | **NSMC**<br/>(acc) | **Question Pair**<br/>(acc) | **Korean-Hate-Speech**<br/>(F1) | **Naver NER**<br/>(F1) | **KorNLI**<br/>(acc) | **KorSTS**<br/>(spearman) | 
| :--------------------- | :----------------: | :--------------------: | :----------------: | :------------------: | :-----------------------: | :-------------------------: | 
| DistilKoBERT           |       88.60        |            92.48          |                 60.72                 |      84.65          |        72.00         |           72.59           |   
| **Dialog-KoELECTRA-Small** |     **90.01**      |           **94.99**             |               **68.26**               |  **85.51**         |      **78.54**       |         **78.96**         |    

<br>

## Train Data


<table class="tg">
<thead>
  <tr>
    <th class="tg-c3ow"></th>
    <th class="tg-c3ow">corpus name</th>
    <th class="tg-c3ow">size</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-c3ow" rowspan="4">dialog</td>
    <td class="tg-0pky"><a href="https://aihub.or.kr/aidata/85" target="_blank" rel="noopener noreferrer">Aihub Korean dialog corpus</a></td>
    <td class="tg-c3ow" rowspan="4">7GB</td>
  </tr>
  <tr>
    <td class="tg-0pky"><a href="https://corpus.korean.go.kr/" target="_blank" rel="noopener noreferrer">NIKL Spoken corpus</a></td>
  </tr>
  <tr>
    <td class="tg-0pky"><a href="https://github.com/songys/Chatbot_data" target="_blank" rel="noopener noreferrer">Korean chatbot data</a></td>
  </tr>
  <tr>
    <td class="tg-0pky"><a href="https://github.com/Beomi/KcBERT" target="_blank" rel="noopener noreferrer">KcBERT</a></td>
  </tr>
  <tr>
    <td class="tg-c3ow" rowspan="2">written</td>
    <td class="tg-0pky"><a href="https://corpus.korean.go.kr/" target="_blank" rel="noopener noreferrer">NIKL Newspaper corpus</a></td>
    <td class="tg-c3ow" rowspan="2">15GB</td>
  </tr>
  <tr>
    <td class="tg-0pky"><a href="https://github.com/lovit/namuwikitext" target="_blank" rel="noopener noreferrer">namuwikitext</a></td>
  </tr>
</tbody>
</table>

<br>

## Vocabulary

We applied morpheme analysis using [huggingface_konlpy](https://github.com/lovit/huggingface_konlpy) when creating a vocabulary dictionary.
As a result of the experiment, it showed better performance than a vocabulary dictionary created without applying morpheme analysis.
<table>
<thead>
  <tr>
    <th>vocabulary size</th>
    <th>unused token size</th>
    <th>limit alphabet</th>
    <th>min frequency</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>40,000</td>
    <td>500</td>
    <td>6,000</td>
    <td>3</td>
  </tr>
</tbody>
</table>

<br>

## References
- [ELECTRA](https://github.com/google-research/electra): Pre-training Text Encoders as Discriminators Rather Than Generators.
- [KoELECTRA](https://github.com/monologg/KoELECTRA): Pretrained ELECTRA Model for Korean

<br>

## Contact Info

For help or issues using Dialog-KoELECTRA, please submit a [GitHub issue](https://github.com/skplanetml/Dialog-KoELECTRA/issues).

For personal communication related to Dialog-KoELECTRA, please contact [Wonchul Kim](https://github.com/skplanetml) (`wonchul.kim@sk.com`).

<br>

## Citation

If you apply this library to any project and research, please cite our code:

```
@misc{DialogKoELECTRA,
  author       = {Okkyun Jeong and Junseok Kim and Wonchul Kim},
  title        = {Dialog-KoELECTRA: Korean conversational language model based on ELECTRA model},
  howpublished = {\url{https://github.com/skplanetml/Dialog-KoELECTRA}},
  year         = {2021},
}
```

<br>

## License

Dialog-KoELECTRA project is licensed under the [Apache License 2.0](https://github.com/skplanetml/Dialog-KoELECTRA/blob/master/LICENSE).

```
 Copyright 2020 ~ present SK Planet Co. RB Dialog solution

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

 http://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.      
```
