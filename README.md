# Welcome to metagenomics tutorial !!!
These tutorials walk you through the process of analyzing shotgun metagenomics data. We will cover: 
1) [Assembly Based Metagenomics](https://github.com/ravinpoudel/metagenomics/wiki/Assembly-Based-Metagenomics)

2) [Read Based Metagenomics](https://github.com/ravinpoudel/metagenomics/wiki/Read-based-metagenomics-(HUMAnN2))

![alt text](https://github.com/ravinpoudel/metagenomics/blob/master/Metagenomics.png)


## Assembly Based Metagenomics
Assembling reads into contigs and mapping reads to the assembled contigs are the primary steps in assembly-based metagenomics. This tutorial walks you through the various steps involved in the process of genome assembly and mapping. 

[[https://github.com/ravinpoudel/metagenomics/blob/master/Assembly-Based-Metagenomics.png|alt=octocat]]


# Step 1: Quality Check
Prior to genome assembly, reads are cleaned using a custom [ARS-RQC pipeline](https://github.com/USDA-ARS-GBRU/ARS-RQC), which utilizes a suite of functions from [BBtools](https://jgi.doe.gov/data-and-tools/bbtools/) to remove host genome contamination and trim the standard Illumina barcodes. JSON files obtained from ARS-RQC can be exported as csv file and read in R to create a summary file. 

_1-1. Create a conda environment_

```
conda create -n Ametagenomics python=3.6
source activate Ametagenomics
conda install pandas
conda install numpy
conda install -c conda-forge mkl_random 
conda install -c anaconda cython y
conda install -c bioconda bbmap
```

_1-2. Get ars-rqc from the repository and install the editable version. This allows to auto-update the changes made in the repository – so that the used program will be the most recent one._

```	
git clone https://github.com/USDA-ARS-GBRU/ARS-RQC.git
cd ARS-RQC
pip install --editable .
```

_1-3. Soft link the input files from collaborator’s raw-data directory._

```
mkdir data
cd data
ln -vs /project/**/**/*.fastq.gz .
```
			
_1-4. Run `01_arsqc_interleave.sh` to create interleaved fastq._

```
#!/bin/bash
#SBATCH --job-name=rqc_interleave_Ametagenomics
#SBATCH --output=rqc_Ametagenomics_%A_%a.out
#SBATCH --error=rqc_Ametagenomics_%A_%a.err
#SBATCH --time=01:00:00
#SBATCH --array=1-156
#SBATCH -p short
#SBATCH -N 1
#SBATCH -n 10

module load miniconda
source activate Ametagenomics
module load pigz
module load bbtools/gcc/64/36.86

RUN=${SLURM_ARRAY_TASK_ID}
echo "My Slurm RUN_ID: '${RUN}'"
echo "My TMPDIR IS: " $TMPDIR

infile1=$(ls data/*_R1.fastq.gz | sed -n ${RUN}p)
echo "$infile1"

infile2=$(echo $infile1 | sed 's/R1/R2/')
echo "$infile2"

outbase=$(basename $infile1 _R1.fastq.gz)
outfile=data/${outbase}_interleave.fastq.gz
echo "$outfile"

reformat.sh in=$infile1 in2=$infile2 out=$outfile
```

_1-5. Run  `02_arsqc_filer.sh` to run qc on fastq._

```
#!/bin/bash
#SBATCH --job-name=rqc_filter_Ametagenomics
#SBATCH --output=rqc_Ametagenomics_%A_%a.out
#SBATCH --error=rqc_Ametagenomics_%A_%a.err
#SBATCH --time=01:00:00
#SBATCH --array=1-156
#SBATCH -p short
#SBATCH -N 1
#SBATCH -n 10

module load miniconda
source activate Ametagenomics
module load pigz


RUN=${SLURM_ARRAY_TASK_ID}
echo "My Slurm RUN_ID: '${RUN}'"
echo "My TMPDIR IS: " $TMPDIR

rootdir="/project/"
echo "$rootdir"


itl_file=$(ls data/*_interleave.fastq.gz | sed -n ${RUN}p)
echo "$itl_file"
time rqcfilter.py --fastq $itl_file --output "$rootdir"rqc_data -p
```

_1-6. Run `03_parse_json.py` – create a summary data-frame using json files obtained from qc. This file is then analyzed in R to create qc-report using R markdown._

```
import json
import pandas as pd
import os

wdir = os.getcwd()

list = []
for lfile in os.listdir("rqc_data"):
    file = os.path.join(wdir, "rqc_data", lfile)
    if file.endswith('.json'):
        bfile = file
        with open(file) as js:
            js_dict = json.load(js)
            tr = js_dict['scaffoldStats1.txt']['desc']['TotalReads']
            tb = js_dict['scaffoldStats1.txt']['desc']['TotalBases']
            contam = js_dict['scaffoldStats1.txt']['desc']['ReadsMatched']
            pctcontam = js_dict['scaffoldStats1.txt']['desc']['PctReadsMatched']
            list.append((os.path.basename(bfile), tr, tb, contam, pctcontam))

cols = ['SampleID', 'TotalReads', 'TotalBases', 'Contaminants', "Percent_Contaminants"]
result = pd.DataFrame(list, columns=cols)
result.to_csv("rqc_data/parse_json.csv")

```

# Step 2: De Novo Assembly
At this point, we have clean reads that are ready for assembling to contigs. Given we have a large number of fastq files (here 156), lets first create a list of file names and separate each file name with a comma, then pass to megahit. 

_2-1. Create a list of files, open in text editor, then find carry-over and replace with comma, then delete comma at the EOF. This information will be pass as input files for genome assembly in megahit._


```
find /project/rqc_data/reads/** -type f > file_list_withpath.txt

```

_2-2. Run genome assembly `04_megahit.sh`_

```
#!/bin/bash
#SBATCH --job-name=megahit_assembly
#SBATCH --output=megahit_assembly_%A_%a.out
#SBATCH --error=megahit_assembly_%A_%a.err
#SBATCH --time=96:00:00
#SBATCH -p mem
#SBATCH -N 1
#SBATCH -n 120

export PATH=$PATH:/home/ravin.poudel/bbmap
export PATH=$PATH:/home/ravin.poudel/megahit/1.1.1
module load miniconda
#source activate Ametagenomics
module load pigz
module load megahit

echo "My TMPDIR IS: " $TMPDIR

time megahit --k-min 27 --k-max 127 --k-step 10 -m 0.98 -t 120 --out-dir megahit_output --kmin-1pass --min-contig-len 300 --tmp-dir $TMPDIR --12 /project/rqc_data/reads/A1_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/A1_run4_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/A2_run4_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/A3_run4_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/C1W_run4_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/C2W_run4_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/C3W_run4_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/M1_run4_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/M2_run4_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/M3_run4_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/O1S_run4_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/O2S_run4_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run1_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run2_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run2_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run2_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run2_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run3_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run3_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run3_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run3_lane4_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run4_lane1_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run4_lane2_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run4_lane3_interleave.rqc.fq.gz,/project/rqc_data/reads/O3S_run4_lane4_interleave.rqc.fq.gz

```

# Step 3: Mapping
Output obtained after the assembly is also a fasta file, assembled contig. This assembled contigs will be used for mapping the reads. Prior we mapped the reads, we can concatenate/merge reads from technical replicates. In our case for each experimental unit, we have 13 fastq files. 

_3-1. Run `05_merge_interleaved_fastq.sh` to merge fastq by technical replicates._


```
#!/bin/bas
#SBATCH --job-name=merge
#SBATCH --output=merge_%A_%a.out
#SBATCH --error=merge_%A_%a.err
#SBATCH --time=01:00:00
#SBATCH -p short
#SBATCH -N 1
#SBATCH -n 40

cat reads/A1*.fq.gz > merged_fastqByRelicates/A1_interleaved_merge.fq.gz
cat reads/A2*.fq.gz > merged_fastqByRelicates/A2_interleaved_merge.fq.gz
cat reads/A3*.fq.gz > merged_fastqByRelicates/A3_interleaved_merge.fq.gz
cat reads/M1*.fq.gz > merged_fastqByRelicates/M1_interleaved_merge.fq.gz
cat reads/M2*.fq.gz > merged_fastqByRelicates/M2_interleaved_merge.fq.gz
cat reads/M3*.fq.gz > merged_fastqByRelicates/M3_interleaved_merge.fq.gz
cat reads/O1S*.fq.gz > merged_fastqByRelicates/O1S_interleaved_merge.fq.gz
cat reads/O2S*.fq.gz > merged_fastqByRelicates/O2S_interleaved_merge.fq.gz
cat reads/O3S*.fq.gz > merged_fastqByRelicates/O3S_interleaved_merge.fq.gz
cat reads/C1W*.fq.gz > merged_fastqByRelicates/C1W_interleaved_merge.fq.gz
cat reads/C2W*.fq.gz > merged_fastqByRelicates/C2W_interleaved_merge.fq.gz
```

_3-2. Also, prior to mapping reads, the genome needs to be indexed. This can be done using `bowtie2-build` function form bowtie2. Run `06_index_genome.sh`._

```
#!/bin/bash

#SBATCH --job-name=index_contig
#SBATCH --output=index_contig_%A_%a.out
#SBATCH --error=index_contig_%A_%a.err
#SBATCH --time=04:00:00
#SBATCH -p short
#SBATCH -N 1
#SBATCH -n 40

module load bowtie2/2.3.4
module load samtools


bowtie2-build megahit_output/final.contigs.fa megahit_output/contigs
```


_3-3. Run `07_mapping.sh` to map reads to assembled genome using bowtie2._

```
#!/bin/bash

#SBATCH --job-name=bowtie_map
#SBATCH --output=bowtie_map_%A_%a.out
#SBATCH --error=bowtie_map_%A_%a.err
#SBATCH --time=12:00:00
#SBATCH --array=1-12
#SBATCH -p short
#SBATCH -N 1
#SBATCH -n 40

module load bowtie2/2.3.4
module load samtools

rootdir="/project/"
echo "$rootdir"

RUN=${SLURM_ARRAY_TASK_ID}
echo "My Slurm RUN_ID: '${RUN}'"

iarray=("$rootdir"rqc_data/merged_fastqByRelicates/*_interleaved_merge.fq.gz)
echo "$iarray"

infile="${iarray[$SLURM_ARRAY_TASK_ID - 1]}"
echo "$infile"

bname=`basename $infile _interleaved_merge.fq.gz`
echo "$bname"

time bowtie2 -p 40 -x megahit_output/contigs --interleaved $infile -S "$rootdir"mapping/"$bname".sam
time samtools sort -o "$rootdir"mapping/"$bname".bam "$rootdir"mapping/"$bname".sam

```

# Step 4: Calling for bins or metagenomes.
At this point, we have obtained a large genomic fragment (genomic mess) representing microbial communities. We need to separate them into individual genomes/bins, for which we are using automated software called `MetaBAT`. MetaBAT accurately bins genomes using probabilistic distances of genome abundance and tetranucleotide frequency. More on `MetaBAT` is available at their [repo](https://bitbucket.org/berkeleylab/metabat) and accompanied [paper](https://peerj.com/articles/1165/).

_4-1. Run `08_metabat.sh`_

```
#!/bin/bash
#SBATCH --job-name=metabat
#SBATCH --output=anvi-metabat_%A_%a.out
#SBATCH --error=anvi-metabat_%A_%a.err
#SBATCH --time=04:00:00
#SBATCH -p short
#SBATCH -N 1
#SBATCH -n 40

module load miniconda
source activate metabat


runMetaBat.sh -m 5000 /project/megahit_output/final.contigs.fa /project/mapping/*.bam
```


# Step 5: Quality check of Metagenome-assembled genomes (MAGs). 

_5-1. Run `09_checkM.sh`_

```
#!/bin/bash
#SBATCH --job-name=CHECKM
#SBATCH --output=anvi-metabat_%A_%a.out
#SBATCH --error=anvi-metabat_%A_%a.err
#SBATCH --time=04:00:00
#SBATCH -p short
#SBATCH -N 1
#SBATCH -n 40

module load miniconda
module load checkm


checkm lineage_wf -t 40 -x fa final.contigs.fa.metabat-bins5000/ /projectq/checkM
```
Filtering bins is not an easy task, as criteria to define/filter bins is not hard and fast. We follow the guidelines for [MIMAG](https://www.nature.com/articles/nbt.3893) such that the bins with completeness > 95 % and contamination < 5% were selected for downstream analyses. [[https://github.com/ravinpoudel/metagenomics/blob/master/MIMAG.png|alt=octocat]]

# Step 6: Functional annotation of MAGs. 
To retrieve the relevant genomic features from the MAGs, we use [prokka](https://github.com/tseemann/prokka) 

_6-1. Run `10_porka_annotation.sh`, which usages prokka_

```
#!/bin/bash
#SBATCH --job-name=PORKA
#SBATCH --output=porka_bybins_%A_%a.out
#SBATCH --error=porka_bybins_%A_%a.err
#SBATCH --time=04:00:00
#SBATCH --array=1-19
#SBATCH -p short
#SBATCH -N 1
#SBATCH -n 40


module load miniconda
source activate megan


RUN=${SLURM_ARRAY_TASK_ID}
echo "My Slurm RUN_ID: '${RUN}'"
echo "My TMPDIR IS: " $TMPDIR

infile=$(ls filter_bins_metabat_checkM/*.fa | sed -n ${RUN}p)
echo "$infile"

outbase=$(basename $infile .fa)
outdirect=porka_bybins/${outbase}
echo "$outdirect"

prokka $infile --outdir $outdirect --prefix $outbase --addgenes --addmrna --centre X --compliant --genus --species --strain --kingdom --cpus 40
```

For each MAGs, prokka provides the following files as output. These outputs can be explored for other downstream analyses such as metabolic pathways. 

```
PROKKA_11302018.daa ## useful to explore using megan
PROKKA_11302018.err
PROKKA_11302018.faa
PROKKA_11302018.ffn
PROKKA_11302018.fna ## fasta file
PROKKA_11302018.fsa
PROKKA_11302018.gbk  ## genebank file
PROKKA_11302018.gff
PROKKA_11302018.log
PROKKA_11302018.sqn
PROKKA_11302018.tbl
PROKKA_11302018.tsv
PROKKA_11302018.txt
```


