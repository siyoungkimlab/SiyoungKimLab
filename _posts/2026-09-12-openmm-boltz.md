---
layout: post
title: OpenMM and Boltz Benchmark 
date: 2026-09-12
published: true
---

This blog benchmarks the performance of a molecular dynamics simulation engine (**openMM**) and a co-folding model (**Boltz**) on four GPU types: *A10*, *A30*, and *A100* (all on Gilbreth) and *H100* (on Gautschi).

## Systems

Four soluble proteins of various sizes were selected for the performance benchmark. IR is an insulin receptor whose kinase domain is studied in this blog.

| | Ubiquitin | HIF2A | IR | SHP2 |
| --- | --- | --- | --- | --- |
| APO PDB | 1UBQ | 1P97 | 1IRK | 2SHP |
| # of residues | 76 | 114 | 306 | 524 |
| # of protein atoms | 1231 | 1810 | 4842 | 8379 |

## Boltz

Boltz-1 and Boltz-2 were tested on all four GPU types. Each prediction used a single GPU. Although the apo protein structures do not contain any ligands, we added a relevant ligand from another PDB entry to test the protein–ligand co-folding capability in our co-folding benchmark.

| | Ubiquitin | HIF2A | IR | SHP2 |
| --- | --- | --- | --- | --- |
| APO PDB | 1UBQ | 1P97 | 1IRK | 2SHP |
| Ligand PDB | N/A | 5TBM | 5HHW | 4RDD |
| Ligand SMILES | N/A | `S(=O)(=O)(C)c1ccc(Oc2cc(F)cc(C#N)c2)c2CC(F)(F)C(O)c12` | `O(CC1OCCCC1)c1cc(-c2cn(C3CC(C[NH+]4CCC4)C3)c3ncnc(N)c23)ccc1` | `S(=O)(=O)([O-])C(C(=O)NC(C(=O)[O-])C1SCC(C)=C(C(=O)[O-])N1)c1ccccc1` |

Each bar comprises the two most time-consuming stages: model loading and prediction. The model-loading time is roughly constant, whereas the prediction time scales with protein size or, more precisely, with the number of tokens. Because the MSA is obtained from a server, the time to retrieve it depends heavily on server availability and is not reproducible. It is usually well under 10 seconds per MSA, so it is not a significant bottleneck for prediction.

![Boltz1](/assets/blog/openmm-boltz/Boltz1.png)
![Boltz2](/assets/blog/openmm-boltz/Boltz2.png)

## openMM
Three force-field combinations were tested on all four GPU types. The Amber simulations used a non-bonded interaction cutoff of 0.9 nm, and the CHARMM36 simulations used 1.2 nm. Each simulation used a single GPU. Each system consists of a protein chain, water, and ions, and does not include a ligand.

![amber19 + TIP3P](/assets/blog/openmm-boltz/amber19_tip3p.png)
![amber19 + OPC](/assets/blog/openmm-boltz/amber19_opc.png)
![charmm36](/assets/blog/openmm-boltz/charmm36.png)

## SLURM scripts
<details markdown="1">
  <summary>Boltz on A10</summary>

```
#!/bin/bash
#SBATCH -J A10_boltz
#SBATCH -A siyoungk
#SBATCH -p a10
#SBATCH -N 1
#SBATCH -n 1      # one task: boltz starts one process per GPU itself
#SBATCH -c 10     # all of the node's cores for that task
#SBATCH --mem=165G
#SBATCH --gres=gpu:1
#SBATCH -t 4:00:00

# sbatch *.sub

# Each A10 (24GB) node has
# 3 × A10 GPUs
# 32 cores

module load monitor
module load conda/2026.03
module load cuda/12.6.0
conda activate boltz

hostname
echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"
nvidia-smi -L        # should list exactly one H100
nproc                # CPUs this job can use; 14 if SLURM pins cores

# track per-code CPU load
monitor cpu percent --all-cores >cpu-percent.log &
CPU_PID=$!

# track memory usage
monitor cpu memory >cpu-memory.log &
MEM_PID=$!

# track gpu usage
monitor gpu percent >gpu-percent.log &
GPU_PID=$!


# actual code
DATA=/depot/siyoungk/data/performance_benchmark/Boltz/dataset/

mkdir -p 00

# Boltz starts its own, so hide the task count from Lightning.
unset SLURM_NTASKS
for pdb in 1IRK 1P97 1UBQ 2SHP; do
    mkdir -p 00/$pdb

    for model in boltz1 boltz2; do

        boltz predict $DATA/$pdb.yaml \
            --model $model \
            --out_dir 00/$pdb/$model \
            --diffusion_samples 5 \
            --output_format mae \
            --devices 1 \
            --preprocessing-threads 1 \
            --use_potentials --use_msa_server

    done
done

# shut down the resource monitors
kill -s INT $CPU_PID $MEM_PID

```
</details>

<details markdown="1">
  <summary>Boltz on A30</summary>
  
#!/bin/bash
#SBATCH -J A30_boltz
#SBATCH -A siyoungk
#SBATCH -p a30
#SBATCH -N 1
#SBATCH -n 1      # one task: boltz starts one process per GPU itself
#SBATCH -c 8      # all of the node's cores for that task
#SBATCH --mem=60G
#SBATCH --constraint=B
#SBATCH --gres=gpu:1
#SBATCH -t 4:00:00

# sbatch *.sub

# Each A30-B (24GB) node has
# 3 × A30 GPUs
# 24 cores

module load monitor
module load conda/2026.03
module load cuda/12.6.0
conda activate boltz

hostname
echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"
nvidia-smi -L        # should list exactly one H100
nproc                # CPUs this job can use; 14 if SLURM pins cores

# track per-code CPU load
monitor cpu percent --all-cores >cpu-percent.log &
CPU_PID=$!

# track memory usage
monitor cpu memory >cpu-memory.log &
MEM_PID=$!

# track gpu usage
monitor gpu percent >gpu-percent.log &
GPU_PID=$!


# actual code
DATA=/depot/siyoungk/data/performance_benchmark/Boltz/dataset/

mkdir -p 00

# Boltz starts its own, so hide the task count from Lightning.
unset SLURM_NTASKS
for pdb in 1IRK 1P97 1UBQ 2SHP; do
    mkdir -p 00/$pdb

    for model in boltz1 boltz2; do

        boltz predict $DATA/$pdb.yaml \
            --model $model \
            --out_dir 00/$pdb/$model \
            --diffusion_samples 5 \
            --output_format mae \
            --devices 1 \
            --preprocessing-threads 1 \
            --use_potentials --use_msa_server

    done
done

# shut down the resource monitors
kill -s INT $CPU_PID $MEM_PID
</details>

<details markdown="1">
  <summary>Boltz on A100-N</summary>

#!/bin/bash
#SBATCH -J A100_boltz
#SBATCH -A siyoungk
#SBATCH -p a100-40gb
#SBATCH --constraint=N
#SBATCH -N 1
#SBATCH -n 1      # one task: boltz starts one process per GPU itself
#SBATCH -c 12     # all of the node's cores for that task
#SBATCH --mem=240G
#SBATCH --gres=gpu:1
#SBATCH -t 4:00:00

# sbatch *.sub

# Each A100-N (40GB) node has
# 2 × A100 GPUs
# 128 cores

module load monitor
module load conda/2026.03
module load cuda/12.6.0
conda activate boltz

hostname
echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"
nvidia-smi -L        # should list exactly one H100
nproc                # CPUs this job can use; 14 if SLURM pins cores

# track per-code CPU load
monitor cpu percent --all-cores >cpu-percent.log &
CPU_PID=$!

# track memory usage
monitor cpu memory >cpu-memory.log &
MEM_PID=$!

# track gpu usage
monitor gpu percent >gpu-percent.log &
GPU_PID=$!


# actual code
DATA=/depot/siyoungk/data/performance_benchmark/Boltz/dataset/

mkdir -p 00

# Boltz starts its own, so hide the task count from Lightning.
unset SLURM_NTASKS
for pdb in 1IRK 1P97 1UBQ 2SHP; do
    mkdir -p 00/$pdb

    for model in boltz1 boltz2; do

        boltz predict $DATA/$pdb.yaml \
            --model $model \
            --out_dir 00/$pdb/$model \
            --diffusion_samples 5 \
            --output_format mae \
            --devices 1 \
            --preprocessing-threads 1 \
            --use_potentials --use_msa_server

    done
done

# shut down the resource monitors
kill -s INT $CPU_PID $MEM_PID
</details>

<details markdown="1">
  <summary>Boltz on A100-G</summary>

#!/bin/bash
#SBATCH -J A100G_boltz
#SBATCH -A siyoungk
#SBATCH -p a100-40gb
#SBATCH --constraint=G
#SBATCH -N 1
#SBATCH -n 1      # one task: boltz starts one process per GPU itself
#SBATCH -c 64     # all of the node's cores for that task
#SBATCH --mem=240G
#SBATCH --gres=gpu:1
#SBATCH -t 4:00:00

# sbatch *.sub

# Each A100-G (40GB) node has
# 2 × A100 GPUs
# 128 cores

module load monitor
module load conda/2026.03
module load cuda/12.6.0
conda activate boltz

hostname
echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"
nvidia-smi -L        # should list exactly one H100
nproc                # CPUs this job can use; 14 if SLURM pins cores

# track per-code CPU load
monitor cpu percent --all-cores >cpu-percent.log &
CPU_PID=$!

# track memory usage
monitor cpu memory >cpu-memory.log &
MEM_PID=$!

# track gpu usage
monitor gpu percent >gpu-percent.log &
GPU_PID=$!


# actual code
DATA=/depot/siyoungk/data/performance_benchmark/Boltz/dataset/

mkdir -p 00

# Boltz starts its own, so hide the task count from Lightning.
unset SLURM_NTASKS
for pdb in 1IRK 1P97 1UBQ 2SHP; do
    mkdir -p 00/$pdb

    for model in boltz1 boltz2; do

        boltz predict $DATA/$pdb.yaml \
            --model $model \
            --out_dir 00/$pdb/$model \
            --diffusion_samples 5 \
            --output_format mae \
            --devices 1 \
            --preprocessing-threads 1 \
            --use_potentials --use_msa_server

    done
done

# shut down the resource monitors
kill -s INT $CPU_PID $MEM_PID
</details>

<details markdown="1">
  <summary>Boltz on H100</summary>
  
```
#!/bin/bash
#SBATCH -J H100_boltz
#SBATCH -A siyoungk
#SBATCH -p ai
#SBATCH -N 1
#SBATCH -n 1      # one task: boltz starts one process per GPU itself
#SBATCH -c 14     # all of the node's cores for that task
#SBATCH --gres=gpu:1
#SBATCH -t 4:00:00

# sbatch *.sub

# Each Gautschi-H node has
# 8 × H100 GPUs
# 2 × Intel Xeon Platinum 8480+
# 112 CPU cores total (56 cores × 2)

module load monitor
module load conda/2026.03
module load cuda/12.6.0
conda activate boltz

hostname
echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"
nvidia-smi -L        # should list exactly one H100
nproc                # CPUs this job can use; 14 if SLURM pins cores

# track per-code CPU load
monitor cpu percent --all-cores >cpu-percent.log &
CPU_PID=$!

# track memory usage
monitor cpu memory >cpu-memory.log &
MEM_PID=$!

# track gpu usage
monitor gpu percent >gpu-percent.log &
GPU_PID=$!


# actual code
DATA=/depot/siyoungk/data/performance_benchmark/Boltz/dataset/

mkdir -p 00

# Boltz starts its own, so hide the task count from Lightning.
unset SLURM_NTASKS
for pdb in 1IRK 1P97 1UBQ 2SHP; do
    mkdir -p 00/$pdb

    for model in boltz1 boltz2; do

        boltz predict $DATA/$pdb.yaml \
            --model $model \
            --out_dir 00/$pdb/$model \
            --diffusion_samples 5 \
            --output_format mae \
            --devices 1 \
            --preprocessing-threads 1 \
            --use_potentials --use_msa_server

    done
done

# shut down the resource monitors
kill -s INT $CPU_PID $MEM_PID

```
</details>

<details markdown="1">
  <summary>openMM on A10</summary>
  
```
#!/bin/bash
#SBATCH -J A10
#SBATCH -A siyoungk
#SBATCH -p a10
#SBATCH -N 1
#SBATCH -n 1      # one task
#SBATCH -c 10     # all of the node's cores for that task
#SBATCH --mem=165G
#SBATCH --gres=gpu:1
#SBATCH -t 24:00:00

# sbatch *.sub

# Each A10 (24GB) node has
# 3 × A10 GPUs
# 32 cores

module load monitor
module load conda/2026.03
module load cuda/12.6.0
conda activate ommflow

hostname
echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"
nvidia-smi -L        # should list exactly one H100
nproc                # CPUs this job can use; 14 if SLURM pins cores

# track per-code CPU load
monitor cpu percent --all-cores >cpu-percent.log &
CPU_PID=$!

# track memory usage
monitor cpu memory >cpu-memory.log &
MEM_PID=$!

# track gpu usage
monitor gpu percent >gpu-percent.log &
GPU_PID=$!


# actual code
DATA=/depot/siyoungk/data/performance_benchmark/MD/dataset

mkdir -p 00

for pdb in 1IRK 1P97 1UBQ 2SHP; do
    mkdir -p 00/$pdb

    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/charmm36 \
        --proteinff charmm36_2024 \
        --waterff tip3p \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
    
    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/amber19_tip3p \
        --proteinff amber19sb \
        --waterff tip3p \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
 
    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/amber19_opc \
        --proteinff amber19sb \
        --waterff opc \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
done

 
# shut down the resource monitors
kill -s INT $CPU_PID $MEM_PID $GPU_PID
```
</details>

<details markdown="1">
  <summary>openMM on A30</summary>
  
```
#!/bin/bash
#SBATCH -J A30
#SBATCH -A siyoungk
#SBATCH -p a30
#SBATCH -N 1
#SBATCH -n 1      # one task
#SBATCH -c 8      # all of the node's cores for that task
#SBATCH --mem=60G
#SBATCH --constraint=B
#SBATCH --gres=gpu:1
#SBATCH -t 24:00:00

# sbatch *.sub

# Each A30-B (24GB) node has
# 3 × A30 GPUs
# 24 cores

module load monitor
module load conda/2026.03
module load cuda/12.6.0
conda activate ommflow

hostname
echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"
nvidia-smi -L        # should list exactly one H100
nproc                # CPUs this job can use; 14 if SLURM pins cores

# track per-code CPU load
monitor cpu percent --all-cores >cpu-percent.log &
CPU_PID=$!

# track memory usage
monitor cpu memory >cpu-memory.log &
MEM_PID=$!

# track gpu usage
monitor gpu percent >gpu-percent.log &
GPU_PID=$!


# actual code
DATA=/depot/siyoungk/data/performance_benchmark/MD/dataset

mkdir -p 00

for pdb in 1IRK 1P97 1UBQ 2SHP; do
    mkdir -p 00/$pdb

    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/charmm36 \
        --proteinff charmm36_2024 \
        --waterff tip3p \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
    
    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/amber19_tip3p \
        --proteinff amber19sb \
        --waterff tip3p \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
 
    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/amber19_opc \
        --proteinff amber19sb \
        --waterff opc \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
done

 
# shut down the resource monitors
kill -s INT $CPU_PID $MEM_PID $GPU_PID
```
</details>

<details markdown="1">
  <summary>openMM on A100-N</summary>
  
```
#!/bin/bash
#SBATCH -J A100
#SBATCH -A siyoungk
#SBATCH -p a100-40gb
#SBATCH --constraint=N
#SBATCH -N 1
#SBATCH -n 1      # one task
#SBATCH -c 12     # all of the node's cores for that task
#SBATCH --mem=240G
#SBATCH --gres=gpu:1
#SBATCH -t 24:00:00

# sbatch *.sub

# Each A100-N (40GB) node has
# 2 × A100 GPUs
# 128 cores

module load monitor
module load conda/2026.03
module load cuda/12.6.0
conda activate ommflow

hostname
echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"
nvidia-smi -L        # should list exactly one H100
nproc                # CPUs this job can use; 14 if SLURM pins cores

# track per-code CPU load
monitor cpu percent --all-cores >cpu-percent.log &
CPU_PID=$!

# track memory usage
monitor cpu memory >cpu-memory.log &
MEM_PID=$!

# track gpu usage
monitor gpu percent >gpu-percent.log &
GPU_PID=$!


# actual code
DATA=/depot/siyoungk/data/performance_benchmark/MD/dataset

mkdir -p 00

for pdb in 1IRK 1P97 1UBQ 2SHP; do
    mkdir -p 00/$pdb

    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/charmm36 \
        --proteinff charmm36_2024 \
        --waterff tip3p \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
    
    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/amber19_tip3p \
        --proteinff amber19sb \
        --waterff tip3p \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
 
    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/amber19_opc \
        --proteinff amber19sb \
        --waterff opc \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
done

 
# shut down the resource monitors
kill -s INT $CPU_PID $MEM_PID $GPU_PID
```
</details>

<details markdown="1">
  <summary>openMM on A100-G</summary>

#!/bin/bash
#SBATCH -J A100
#SBATCH -A siyoungk
#SBATCH -p a100-40gb
#SBATCH --constraint=G
#SBATCH -N 1
#SBATCH -n 1      # one task
#SBATCH -c 64     # all of the node's cores for that task
#SBATCH --mem=240G
#SBATCH --gres=gpu:1
#SBATCH -t 24:00:00

# sbatch *.sub

# Each A100-G (40GB) node has
# 2 × A100 GPUs
# 128 cores

module load monitor
module load conda/2026.03
module load cuda/12.6.0
conda activate ommflow

hostname
echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"
nvidia-smi -L        # should list exactly one H100
nproc                # CPUs this job can use; 14 if SLURM pins cores

# track per-code CPU load
monitor cpu percent --all-cores >cpu-percent.log &
CPU_PID=$!

# track memory usage
monitor cpu memory >cpu-memory.log &
MEM_PID=$!

# track gpu usage
monitor gpu percent >gpu-percent.log &
GPU_PID=$!


# actual code
DATA=/depot/siyoungk/data/performance_benchmark/MD/dataset

mkdir -p 00

for pdb in 1IRK 1P97 1UBQ 2SHP; do
    mkdir -p 00/$pdb

    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/charmm36 \
        --proteinff charmm36_2024 \
        --waterff tip3p \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
    
    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/amber19_tip3p \
        --proteinff amber19sb \
        --waterff tip3p \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
 
    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/amber19_opc \
        --proteinff amber19sb \
        --waterff opc \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
done

 
# shut down the resource monitors
kill -s INT $CPU_PID $MEM_PID $GPU_PID
</details>

<details markdown="1">
  <summary>openMM on H100</summary>
  
```
#!/bin/bash
#SBATCH -J H100
#SBATCH -A siyoungk
#SBATCH -p ai
#SBATCH -N 1
#SBATCH -n 1      # one task
#SBATCH -c 14     # all of the node's cores for that task
#SBATCH --gres=gpu:1
#SBATCH -t 24:00:00

# sbatch *.sub

# Each Gautschi-H node has
# 8 × H100 GPUs
# 2 × Intel Xeon Platinum 8480+
# 112 CPU cores total (56 cores × 2)

module load monitor
module load conda/2026.03
module load cuda/12.6.0
conda activate ommflow

hostname
echo "CUDA_VISIBLE_DEVICES=$CUDA_VISIBLE_DEVICES"
nvidia-smi -L        # should list exactly one H100
nproc                # CPUs this job can use; 14 if SLURM pins cores

# track per-code CPU load
monitor cpu percent --all-cores >cpu-percent.log &
CPU_PID=$!

# track memory usage
monitor cpu memory >cpu-memory.log &
MEM_PID=$!

# track gpu usage
monitor gpu percent >gpu-percent.log &
GPU_PID=$!


# actual code
DATA=/depot/siyoungk/data/performance_benchmark/MD/dataset

mkdir -p 00

for pdb in 1IRK 1P97 1UBQ 2SHP; do
    mkdir -p 00/$pdb

    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/charmm36 \
        --proteinff charmm36_2024 \
        --waterff tip3p \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
    
    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/amber19_tip3p \
        --proteinff amber19sb \
        --waterff tip3p \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
 
    ommflow $DATA/$pdb.pdb \
        --workdir 00/$pdb/amber19_opc \
        --proteinff amber19sb \
        --waterff opc \
        --platform CUDA \
        --precision mixed \
        --production-ns 10
done

 
# shut down the resource monitors
kill -s INT $CPU_PID $MEM_PID $GPU_PID
```
</details>
