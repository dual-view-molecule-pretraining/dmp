# Introduction
This repository contains the code for **Dual-view Molecule Pre-training**, which is introduced in the NeurIPS-21 submission. The code is originally forked from [Fairseq](https://github.com/pytorch/fairseq).
# Requirements and Installation
* PyTorch version == 1.8.0
* PyTorch Geometric version == 1.6.3
* RDKit version == 2020.09.5

To install the code from source
```
git clone https://github.com/dual-view-molecule-pretraining/dmp
cd dmp
pip install -e . 
```
# Getting Started
## Pre-training
### Data Preprocessing
```shell
DATADIR=/yourdatadir

# Canonicalize all SMILES
python molecule/canonicalize.py $DATADIR/train.smi --workers 30

# Tokenize all SMILES
python molecule/tokenize_re.py $DATADIR/train.smi.can --workers 30 \
  --output-fn $DATADIR/train.bpe 

# You also should canonicalize and tokenize the dev set.

# Binarize the data
fairseq-preprocess \
    --only-source \
    --trainpref $DATADIR/train.bpe \
    --validpref $DATADIR/valid.bpe \
    --destdir /data/pubchem \
    --workers 30 --srcdict molecule/dict.txt \
    --molecule

```
### Train
```shell
DATADIR=/data/pubchem

TOTAL_UPDATES=125000 # Total number of training steps
WARMUP_UPDATES=10000 # Warmup the learning rate over this many updates
PEAK_LR=0.0005       # Peak learning rate, adjust as needed
UPDATE_FREQ=16       # Increase the batch size 16x
MAXTOKENS=12288
DATATYPE=tg
arch=dmp

fairseq-train --fp16 $DATADIR \
    --task dmp --criterion dmp \
    --arch $arch --max-tokens $MAXTOKENS --update-freq $UPDATE_FREQ \
    --optimizer adam --adam-betas '(0.9,0.98)' --adam-eps 1e-6 --clip-norm 0 \
    --lr-scheduler polynomial_decay --lr $PEAK_LR --warmup-updates $WARMUP_UPDATES --total-num-update $TOTAL_UPDATES \
    --dropout 0.1 --attention-dropout 0.1 --weight-decay 0.01 \
    --validate-interval-updates 1000 --save-interval-updates 1000 \
    --datatype $DATATYPE --required-batch-size-multiple 1 \
    --use-bottleneck-head --bottleneck-ratio 4  --use-mlm | tee -a ${SAVE_DIR}/training.log
```
## Finetuning
Finetune with the pre-trained model {checkpoint}.
```shell
bash finetune_alltasks.sh -d clintox --lr 0.0001 -u 1 \
  --wd 0.01 --seed 1 \
  --datatype tg --gradmultiply --use-bottleneck-head --bottleneck-ratio 4 \
   -m {checkpoint} --coeff1 0 --bd --dict /yourdict
```
### Inference
```shell
python molecule/inference.py molnet-bin/$DATASET/$taskname {checkpoint}
```