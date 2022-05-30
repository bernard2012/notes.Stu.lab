## Normalization 

### Step 1:
Go to the bam directory. Create a list of bam files to be merged:

```
cd /n/scratch3/users/q/qz64/c2c12_dtag_k27me3_rad21_yy1_smad3/220524_TL10096_NS500233_fastq/aligned.aug10/dup.marked
#note the dup.marked at the end of the path
```

```
#vim list1
#i for insert
QZ_rad21dtag_aligned_reads.bam
QZ_rad21dmso_aligned_reads.bam
#save and exit
```

Start merging:
```
cp ~/merge.bam.2.sh .
sbatch ./merge.bam.2.sh list1 merged.QZ_rad21.bam
```

### Step 2:
Peak calling on merged bam file

```
ln -s CTCF/merged.bam merged_CTCF_aligned_reads.bam
ln -s CTCF/merged.bam.bai merged_CTCF_aligned_reads.bam.bai
cd /n/scratch3/users/q/qz64/scarlett_cutrun_dec29_21/211220_TL9605_NB551325_fastq/aligned.aug10/dup.marked
#modify the script macs2.merged.sh
sbatch ./macs2.merged.sh merged_CTCF_aligned_reads.bam
```

### Step 3:

Get the flank regions around merged peak file. Then subtract peak regions from flank regions.
```
cp ~/chipseq-scripts/get_flank.py .
python3 get_flank.py merged_CTCF_aligned_reads_narrow_peaks.narrowPeak > flank_merged_CTCF.narrowPeak
bedops -d flank_merged_CTCF.narrowPeak merged_CTCF_aligned_reads_narrow_peaks.narrowPeak > flank_good_merged_CTCF.narrowPeak
```


### Step 4:

Use the file as input to do_bamliquidator.sh
```
cd ..
cp ~/norm_cutrun_scripts/do_bamliquidiator_ctcf.sh .
vim do_bamliquidator_ctcf.sh #edit the script file
#pay attention to the merged_flank_peak file
#then add the bam list in there
sbatch ./do_bamliquidator_ctcf.sh
```

This script will create the summary_flank folder, and put the resulting feature count table in that folder.

### Step 5:

Go to the `summary_flank` folder. Then check the results. Modify the do_script.py. Then do deseq.R to generate the scaling factors.
```
cd summary_flank
cp ~/norm_cutrun_scripts/summary_flank/deseq.R .
cp ~/norm_cutrun_scripts/summary_flank/calculate_ratio.py .
cp ~/norm_cutrun_scripts/summary_flank/list .
cp ~/norm_cutrun_scripts/summary_flank/scale .
cp ~/norm_cutrun_scripts/summary_flank/create_subsample.py .
cp ~/norm_cutrun_scripts/summary_flank/do_script.py .
```

Add the samples and phenotype annotations to the do_script.py.

Add the sample list to list file.
```
vim do_script.py #edit
ls -1 *CTCF*.sum > list
python3 do_script.py list list
#example output
 KO0_CTCF KO41_CTCF NS01_CTCF NS41_CTCF 
0.9081629 1.0271722 1.2782090 0.8513654 
```

Create a new file called `scale`, with the list of samples in the first column, and the factors in the second column:
```
KO0_CTCF	0.908
KO41_CTCF	1.027
NS01_CTCF	1.278
NS41_CTCF	0.8513
```

Locate the smallest number (0.8513). Divide this number by each sample's factor. For example, for KO0_CTCF, it is 0.8513 / 0.908 = 0.937458. Put this in the third column. Note for sample with smallest value where 0.8513 / 0.8513, put in 0.99999 instead of 1.0.

```
KO0_CTCF	0.908	0.93745
KO41_CTCF	1.027	0.8288
NS01_CTCF	1.278	0.666
NS41_CTCF	0.8513	0.9999
```
### Step 6:

Create the subsample script:

```
python2 create_subsample.py scale
#!/bin/bash
#SBATCH -n 1                               # Request one core
#SBATCH -N 1                               # Request one node (if you request more than one core with -n, also using
                                           # -N 1 means all cores will be on the same node)
#SBATCH -t 0-5:00                         # Runtime in D-HH:MM format
#SBATCH -p short                           # Partition to run in
#SBATCH --mem=8000                        # Memory total in MB (for all cores)
#SBATCH -o hostname_%j.out                 # File to which STDOUT will be written, including job ID
#SBATCH -e hostname_%j.err                 # File to which STDERR will be written, including job ID
#SBATCH --mail-type=ALL                    # Type of email notification- BEGIN,END,FAIL,ALL
#SBATCH --mail-user=bernardzhu@gmail.com   # Email to which notifications will be sent
module load samtools
samtools view -s 0.9374589 -b dup.marked/KO0_CTCF_aligned_reads.bam > subsample/KO0_CTCF.bam
samtools view -s 0.82884388 -b dup.marked/KO41_CTCF_aligned_reads.bam > subsample/KO41_CTCF.bam
samtools view -s 0.666061184 -b dup.marked/NS01_CTCF_aligned_reads.bam > subsample/NS01_CTCF.bam
samtools view -s 0.99999 -b dup.marked/NS41_CTCF_aligned_reads.bam > subsample/NS41_CTCF.bam
```

Once satisfied,
```
python3 create_subsample.py scale > subsample.sh
chmod a+x subsample.sh
cp subsample.sh ../.
cd ..
```

## Step 7:

Use the scaling factors to subsample bam files.

```
cd /n/scratch3/users/q/qz64/scarlett_cutrun_dec29_21/211220_TL9605_NB551325_fastq/aligned.aug10
mkdir subsample
sbatch ./subsample.sh
```

## Step 8:

```
cd subsample
cp ~/norm_cutrun_scripts/subsample/integrated.step2.sh .
cp ~/norm_cutrun_scripts/subsample/integrated.step2.large.sh .
```

Open integrated.step2.sh, change the organism lines:
```
#mm10
chromsizedir=`dirname /home/qz64/chrom.mm10.unmasked/mm10.fa`
$macs2bin/macs2 callpeak -t $dir/"$base_file".bam -g mm

#hg19

```
