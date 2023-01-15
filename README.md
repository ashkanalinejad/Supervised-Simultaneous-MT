# Supervised Simultaneous Machine Translation

## Requirements

- Python 3.7
- run `pip install --editable .` to install required dependencies

You also need to have a fully trained NMT model and its output by fairseq for your desired language pair, before starting to run the code. So you will have two parallel folders: 1. A clone of this repo 2. Original Fairseq folder with your NMT experiments.

The following commands are based on IWSLT14 dataset. Change it according to your own dataset. `/path/to/datafolder/data-bin/iwslt14.tokenized.de-en/` refers to the bin folder of modified data by fairseq, where the .bin and .idx files are located. You may need to add an additional dictionary for the actions. There is a sample action dictionary in this repo, called: `dict.act.txt`. Place this in the same data folder next next to the other data files.

## Generating oracle action sequences
In order to generate action sequences for Test set run the the following command:

```
python fairseq_cli/generate_action.py /path/to/datafolder/data-bin/iwslt14.tokenized.de-en/
-s de
-t en
--user-dir ../examples/Supervised_simul_MT
--task Supervised_simultaneous_translation
--gen-subset test
--path /path/to/model/folder/nmt_trans_iwslt14_ende_base/checkpoint_best.pt
--left-pad-source False
--max-tokens 8000
--beam 10 > /path/to/model/folder/nmt_trans_iwslt14_ende_base/action_sequence.txt
```
Then you can run

```
python scripts/sort-sentences.py /path/to/model/folder/nmt_trans_iwslt14_ende_base/action_sequence.txt 5 > /path/to/model/folder/nmt_trans_iwslt14_ende_base/action_sequence.lines.txt
```

to have a clean folder of action sequences for each sentence.

## Training the model

Before starting to train the model, for each sentence in our dataset we need to have a gold action sequence and the translation generated by the MT model following the gold actions. We can generate such input by running the following command on our train, valid, and test subsets:

```
python fairseq_cli/generate_input.py /path/to/datafolder/iwslt14.tokenized.de-en/bin-en_de/
-s en -t de
--user-dir ../examples/Supervised_simul_MT
--task Supervised_simultaneous_translation
--gen-subset test
--path /path/to/model/folder/nmt_trans_iwslt14_ende_base/checkpoint_best.pt
--left-pad-source False
--max-tokens 8000
--skip-invalid-size-inputs-valid-test
--beam 5
--has-target False > /path/to/model/folder/nmt_trans_iwslt14_ende_base/test.beam5_notarget.input.txt
```

Then we can pass these files to start training the model:

```
python train.py /path/to/datafolder/nmt_trans_iwslt14_ende_base/bin-data
-s src -t trg
--clip-norm 5
--save-dir ./agent5_trans_iwslt14_ende_big_nmt_bi_prepinput/
--user-dir ../examples/Supervised_simul_MT
--max-epoch 50
--lr 8e-4
--dropout 0.4
--lr-scheduler fixed
--optimizer adam
--arch agent_lstm_5_big
--task Supervised_simultaneous_translation
--update-freq 4
--skip-invalid-size-inputs-valid-test
--sentence-avg
--criterion label_smoothed_cross_entropy
--label-smoothing 0.1
--batch-size 100
--ddp-backend=no_c10d
--nmt-path /path/to/model/folder/nmt_trans_iwslt14_ende_base/checkpoint_best.pt
--report-accuracy
--lr-shrink 0.95
--force-anneal 4
--has-reference-action | tee ./agent5_trans_iwslt14_ende_big_nmt_bi_prepinput/agent5_trans_iwslt14_ende_big_nmt_bi_prepinput.log
```

## Testing the model

After finishing the training process, we can use the following command to generate the actions and translations simultaneously:

```
python fairseq_cli/generate_simultaneous.py /path/to/datafolder/iwslt14.tokenized.de-en/bin-en_de/
-s en -t de
--user-dir ../examples/Supervised_simul_MT
--task Supervised_simultaneous_translation
--gen-subset test
--path /path/to/model/folder/nmt_trans_iwslt14_ende_base/checkpoint_best.pt
--left-pad-source False
--max-tokens 8000
--skip-invalid-size-inputs-valid-test
--has-target False
--agent-path ./agent5_trans_iwslt14_ende_big_nmt_bi_prepinput/checkpoint_last.pt
--beam 5 > ./agent5_trans_iwslt14_ende_big_nmt_bi_prepinput/test_action_seq_beam5_last.txt
```

## Liscence

Main parts of the code is coming from https://github.com/facebookresearch/fairseq. All its liscenes also applies to this repo.
