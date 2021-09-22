# Setting up `megalodon` on a `slurm` cluster #

`megalodon` is an ONT-authored tool that calls modified bases from ONT reads. In this example, I wanted to call 5mC bases from reads produced from a GridION ( = MinION) machine.

`megalodon` depends on `guppy` and also on `rerio`, all from ONT.

All three tools are under active development, so this pipeline (written Sept 2021) might not apply in a few months' time.

## Preface ##

I like to collate my downloaded tools in `~/tools`. It looks like this

```shell
ls -l ~/tools/
total 861M
drwxr-xr-x  7 lie128 hpc-users 4.0K Sep 22 13:19 megalodon/
drwxr-xr-x 10 lie128 hpc-users 8.0K Sep 22 13:07 minimap2/
drwxr-xr-x  5 lie128 hpc-users 4.0K Sep 22 11:39 ont-guppy/
-rw-r--r--  1 lie128 hpc-users 857M Sep 22 10:53 ont-guppy_5.0.14_linux64.tar.gz
drwxr-xr-x  5 lie128 hpc-users 4.0K Sep 22 11:39 rerio/
```

## Setting up `guppy` ##

Download URL can be obtained from https://community.nanoporetech.com/downloads, but requires login.

However,  once the URL is known, no login credentials to download; i.e. download can be done on the cluster directly.

```shell
cd ~
mkdir tools
cd tools/
wget https://mirror.oxfordnanoportal.com/software/analysis/ont-guppy_5.0.14_linux64.tar.gz
tar zxvf ont-guppy_5.0.14_linux64.tar.gz
```

## Setting up rerio ##

From https://github.com/nanoporetech/rerio, plus modifications.

```shell
git clone https://github.com/nanoporetech/rerio
# models are poretype-specific and modified-base-specific. check `rerio` github for a full list of models; alter to taste
rerio/download_model.py rerio/basecall_models/res_dna_r941_min_modbases_5mC_CpG_v001
cp ont-guppy/data/barcoding/* rerio/basecall_models/barcoding/  # enable barcoding support
```

## Setting up megalodon ##

From https://github.com/nanoporetech/megalodon, plus modifications.

```shell
module load miniconda3                # if the module isn't on slurm, you'll need to install miniconda3
conda create -n megalodon python=3.8  # atm, ont-pyguppy-client-api needs python < 3.9
conda init bash                       # run this if prompted by conda; remember to restart your shell
conda activate megalodon
pip install megalodon

megalodon --help                      # should produce help screen, not "command not found"
```

## Troubleshooting corner ##

I'm still not 100% sure WHY these errors occur, but google + elbow grease led to fixes that work (mysteriously).

Problem 1: `mappy` (Python bindings for `minimap2`, automatically installed by `pip install megalodon`) complains about intel sse2 something something.

```py
Traceback (most recent call last):
  File "/home/lie128/.conda/envs/megalodon/bin/megalodon", line 8, in <module>
    sys.exit(_main())
  File "/home/lie128/.conda/envs/megalodon/lib/python3.8/site-packages/megalodon/__main__.py", line 689, in _main
    from megalodon import megalodon
  File "/home/lie128/.conda/envs/megalodon/lib/python3.8/site-packages/megalodon/megalodon.py", line 11, in <module>
    import mappy
ImportError: /home/lie128/.conda/envs/megalodon/lib/python3.8/site-packages/mappy.cpython-38-x86_64-linux-gnu.so: undefined symbol: __intel_sse2_strcpy
```

Fix: recompile mappy with gcc

```shell
pip remove mappy
module load gcc/11.1.0
export CC=gcc
cd ~/tools
git clone https://github.com/lh3/minimap2
cd minimap2
pip install .
```

Problem 2: `megalodon` complains about... svml something wtfbbq.

```py
Traceback (most recent call last):
  File "/home/lie128/.conda/envs/megalodon/bin/megalodon", line 8, in <module>
    sys.exit(_main())
  File "/home/lie128/.conda/envs/megalodon/lib/python3.8/site-packages/megalodon/__main__.py", line 689, in _main
    from megalodon import megalodon
  File "/home/lie128/.conda/envs/megalodon/lib/python3.8/site-packages/megalodon/megalodon.py", line 15, in <module>
    from megalodon import (
  File "/home/lie128/.conda/envs/megalodon/lib/python3.8/site-packages/megalodon/aggregate.py", line 9, in <module>
    from megalodon import (
  File "/home/lie128/.conda/envs/megalodon/lib/python3.8/site-packages/megalodon/mods.py", line 15, in <module>
    from megalodon import (
ImportError: /home/lie128/.conda/envs/megalodon/lib/python3.8/site-packages/megalodon/decode.cpython-38-x86_64-linux-gnu.so: undefined symbol: __svml_log1pf4
```

Fix: also recompile megalodon with gcc

```shell
pip remove megalodon
module load gcc/11.1.0
export CC=gcc
cd ~/tools
git clone https://github.com/nanoporetech/megalodon
cd megalodon
pip install .
```

# SLURM script that worked for me #

`megalodon` DOES NOT have multiplexing capabilities yet--it won't automatically split results by barcode; this has to be done manually. \
https://github.com/nanoporetech/megalodon/issues/43. 

SLURM script for non-multiplexed runs / want to merge all multiplexed reads together.

```shell
#!/bin/bash
#SBATCH --job-name=test_meg_gpu
#SBATCH --time=1:59:00
#SBATCH --ntasks-per-node=14
#SBATCH --gres=gpu:1
#SBATCH --mem=16g

# application specific commands
echo assigned CUDA device: ${CUDA_VISIBLE_DEVICES}

megalodon ../00_raw_data/20210914_0348_X4_FAQ88026_aa298ba1/fast5_pass/ \
  --outputs basecalls mappings mod_mappings mods \
  --reference ../refs/homo_sapiens.KY962518.last_11kb_first.ont.mmi \
  --guppy-server-path ./ont-guppy/bin/guppy_basecall_server \
  --guppy-params "-d ./rerio/basecall_models/ --num_callers 10" \
  --guppy-config res_dna_r941_min_modbases_5mC_CpG_v001.cfg \
  --guppy-timeout 600 \
  --mod-motif m CG 0 \
  --suppress-progress-bars \
  --devices "${CUDA_VISIBLE_DEVICES}" --processes 14 --overwrite
```

SLURM script that splits reads by barcode, for input data that looks like

```shell
ls -l ../00_raw_data/20210914_0348_X4_FAQ88026_aa298ba1/fast5_pass/
total 52K
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode13/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode14/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode15/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode16/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode17/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode18/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode19/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode20/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode21/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode22/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode23/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 barcode24/
drwxr-xr-x 2 lie128 hpc-users 4.0K Sep 16 12:22 unclassified/
```

```shell
#!/bin/bash
#SBATCH --job-name=test_meg_gpu
#SBATCH --time=1:59:00
#SBATCH --ntasks-per-node=14
#SBATCH --gres=gpu:1
#SBATCH --mem=16g

# application specific commands
#module load miniconda3
#conda activate megalodon

echo assigned CUDA device: ${CUDA_VISIBLE_DEVICES}

for a in ../00_raw_data/20210914_0348_X4_FAQ88026_aa298ba1/fast5_pass/*
do
  b=`basename ${a}`
  megalodon ${a} \
    --outputs basecalls mappings mod_mappings mods \
    --output-directory ${b} \
    --reference ../refs/homo_sapiens.KY962518.last_11kb_first.ont.mmi \
    --guppy-server-path ./ont-guppy/bin/guppy_basecall_server \
    --guppy-params "-d ./rerio/basecall_models/ --num_callers 10" \
    --guppy-config res_dna_r941_min_modbases_5mC_CpG_v001.cfg \
    --guppy-timeout 600 \
    --mod-motif m CG 0 \
    --suppress-progress-bars \
    --devices "${CUDA_VISIBLE_DEVICES}" --processes 14 --overwrite
done
```

Note: `*.mmi` files are minimap2 index files, generated with commands like \
`minimap2 -x map-ont -d homo_sapiens.KY962518.last_11kb_first.ont.mmi homo_sapiens.KY962518.last_11kb_first.fa`

Note 2: GPU `megalodon` was ~100x faster than CPU `megalodon`. Highly recommend getting GPU to work, something that took 3 days to run on CPU took ~40 mins to complete.

# Results #

The files you should pay attention to are "modified_bases.5mC.bed" files in the results folders. These are in bedMethyl format. \
https://www.encodeproject.org/data-standards/wgbs/

TL;DR coverage is in penultimate column, methyl % is in final column.