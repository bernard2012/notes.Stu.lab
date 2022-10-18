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
#add the _aligned_reads.bam suffix to each file
ln -s merged.QZ_rad21.bam merged_QZ_rad21_aligned_reads.bam
ln -s merged.QZ_rad21.bam.bai merged_QZ_rad21_aligned_reads.bam.bai
cp ~/cutrun_scripts/macs2.merged.sh .
sbatch ./macs2.merged.sh merged_QZ_rad21_aligned_reads.bam
```

### Step 3:

Get the flank regions around merged peak file. Then subtract peak regions from flank regions.
```
cp ~/chipseq-scripts/get_flank.py .
python3 get_flank.py merged_QZ_rad21_aligned_reads_narrow_peaks.narrowPeak > flank_merged_QZ_rad21.narrowPeak
bedops -d flank_merged_QZ_rad21.narrowPeak merged_QZ_rad21_aligned_reads_narrow_peaks.narrowPeak > flank_good_merged_QZ_rad21.narrowPeak
```


### Step 4:

Use the file as input to do_bamliquidator.sh
```
cd ..
pwd
#should be now in XXX/aligned.aug10 directory
cp ~/norm_cutrun_scripts/do_bamliquidator_list1.sh do_bamliquidator_list1.sh
#vim do_bamliquidator_list1.sh #view the script file to make sure everything looks right (optional)
sbatch ./do_bamliquidator_list1.sh dup.marked/flank_good_merged_QZ_rad21.narrowPeak dup.marked/list1
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

Add the samples and phenotype annotations to the do_script.py. For example, modify do_script.py in the following function. 

```
def read_phenotype():
    pheno = {}
    pheno["DMSO1_H3K27Ac"] = ["dmso", "h3k27ac"]
    pheno["DMSO2_H3K27Ac"] = ["dmso", "h3k27ac"]
    pheno["dTag1_H3K27Ac"] = ["dtag", "h3k27ac"]
    pheno["dTag2_H3K27Ac"] = ["dtag", "h3k27ac"]
    return pheno
```
In the above, for `pheno["XXX"]`, fill in the sample name (which can be gotten using `ls -1 *.sum` command). For `["dmso", "h3k27ac"]`, you can fill in the phenotype annotations, for example for `pheno["DMSO1_H3K27Ac"]`, it will be "dmso" for first column, and "h3k27ac" for the second column. The `1` in `DMSO1` refers to the replicate ID.

Add the sample list to list file.
```
vim do_script.py #edit
ls -1 *CTCF*.sum > list

module load R/3.3.3
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
```

The generated `subsample.sh` does not contain the submission script header above. Add them in to the beginning of `subsample.sh` using VIM text editor:
```
vim subsample.sh
```

Insert the following:
```
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
```
Save and exit VIM.

Copy `subsample.sh` out:
```
chmod a+x subsample.sh
cp subsample.sh ../.
cd ..
```

### Step 7:

Use the scaling factors to subsample bam files.

```
cd /n/scratch3/users/q/qz64/scarlett_cutrun_dec29_21/211220_TL9605_NB551325_fastq/aligned.aug10
mkdir subsample
sbatch ./subsample.sh
```

### Step 8:

```
cd subsample
cp ~/norm_cutrun_scripts/subsample/integrated.step2.sh .
cp ~/norm_cutrun_scripts/subsample/integrated.step2.large.sh .
cp ~/cutrun_pipeline/filter*.awk .
```


### Choice 1: histone CUT&RUN

Create a sample list of bam's to merge:
```
vim list.histone
```
```
DMSO8h_CTCF11.bam
DMSO8h_CTCF12.bam
DMSO8h_CTCF13.bam
dTag8h_CTCF14.bam
dTag8h_CTCF15.bam
dTag8h_CTCF16.bam
```
Type `:wq` to save and exit. 



### Choice 2: TF CUT&RUN (where we need to divide into 120bp and 300bp groups)

Create a sample list of bam's to process:
```
vim list.CTCF
```
```
DMSO8h_CTCF11.bam
DMSO8h_CTCF12.bam
DMSO8h_CTCF13.bam
dTag8h_CTCF14.bam
dTag8h_CTCF15.bam
dTag8h_CTCF16.bam
```
Type `:wq` to save and exit. 


```
for i in `cat list.CTCF`; do sbatch ./integrated.step2.sh $i; done
```
The above script after finished should create a set of directories dup.marked.120bp, dup.marked.300bp, dup.marked.gt.300bp corresponding to BAM files subset by the fragment length (120bp and 300bp).

```
for i in `cat list.CTCF`; do sbatch ./integrated.step2.large.sh $i; done
```
Do this again for large fragments, i.e. 300bp and gt.300bp.

The following step will do the merging of CTCF BAM's.
```
cd dup.marked.120bp #should be in dup.marked/dup.marked.120bp
cp ../list.CTCF .
sbatch ~/merge.bam.2.sh list.CTCF merged_CTCF.bam
cp ~/norm_cutrun_scripts/subsample/macs2.merged.sh .
sbatch ./macs2.merged.sh merged_CTCF.bam
```

```
cd ../dup.marked.300bp
cp ../list.CTCF .
sbatch ~/merge.bam.2.sh list.CTCF merged_CTCF.bam
cp ~/norm_cutrun_scripts/subsample/macs2.merged.sh .
sbatch ./macs2.merged.sh merged_CTCF.bam
```

### Step 9

```
cd ../dup.marked.120bp
cp ~/norm_cutrun_scripts/subsample/do_bamliquidator.sh .
cp ~/norm_cutrun_scripts/subsample/do_bamliquidator_broad.sh .
```

Check the content of do_bamliquidator.sh:
```
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


type=$1 #120bp
mkdir summary.narrow.$type
type2=$2

#=====================================================
file2=dup.marked."$type"/merged_"$type2"_narrow_peaks.narrowPeak #bed file

for i in `cat list.$type2|tr "\n" " "|sed "s/ $//g"|sed "s/.bam//g"`; do
echo $i;
file1=dup.marked."$type"/"$i".bam
outfile=`basename $file1 .bam`.sum
~/bamliquidator/bamliquidator_batch $file1 $file2 . 1 0 > summary.narrow."$type"/$outfile
done
```

Run do_bamliquidator.sh:

```
sbatch ./do_bamliquidator.sh 120bp CTCF
sbatch ./do_bamliquidator.sh 300bp CTCF
```

The above scripts mean that run bamliquidator for 120bp CTCF and run once for the large fragments. Bamliquidator is a program that takes a BAM file and a BED file and count reads of each BED interval in each BAM file. The output files should be in summary.narrow.120bp and summary.narrow.300bp. 

```
cd summary.narrow.120bp
ls -ltr
-rw-rw-r-- 1 qz64 qz64 1166918 Sep  1 14:21 DMSO8h_CTCF11.sum
-rw-rw-r-- 1 qz64 qz64 1165796 Sep  1 14:21 DMSO8h_CTCF12.sum
-rw-rw-r-- 1 qz64 qz64 1164396 Sep  1 14:22 DMSO8h_CTCF13.sum
-rw-rw-r-- 1 qz64 qz64 1168293 Sep  1 14:22 dTag8h_CTCF14.sum
-rw-rw-r-- 1 qz64 qz64 1167441 Sep  1 14:22 dTag8h_CTCF15.sum
-rw-rw-r-- 1 qz64 qz64 1167795 Sep  1 14:23 dTag8h_CTCF16.sum
```

### Step 10: DESeq2 finding differential peaks

```
cd summary.narrow.120bp
cp ~/norm_cutrun_scripts/subsample/summary.narrow.120bp/*.sh .
cp ~/norm_cutrun_scripts/subsample/summary.narrow.120bp/*.py .
cp ~/norm_cutrun_scripts/subsample/summary.narrow.120bp/*.R .
```

```
ls -1 *.sum > list.CTCF
```

Modify do_script.py to update the genotype and time column information:
```
#For example:
def read_phenotype():
    pheno["DMSO8h_CTCF11"] = ["dmso", "8h"]
    pheno["DMSO8h_CTCF12"] = ["dmso", "8h"]
    pheno["DMSO8h_CTCF13"] = ["dmso", "8h"]
    pheno["dTag8h_CTCF14"] = ["dtag", "8h"]
    pheno["dTag8h_CTCF15"] = ["dtag", "8h"]
    pheno["dTag8h_CTCF16"] = ["dtag", "8h"]
```

Modify deseq.R:
```
dres<-results(dds, contrast=list("group4hdmso", "group4hdtag"))
dres.sign<-dres[which(dres$pvalue<0.05),]
dres.sign<-dres.sign[order(dres.sign$pvalue, decreasing=F),]
outfile<-paste("DESeq2.", prefix, ".4hdmso.4hdtag.txt", sep="")
write.table(dres.sign, file=outfile, sep="\t", row.names=T, col.names=T, quot=F)

```

Remember to change the group4hdmso and group4hdtag to correspond to the correct genotype and time information (in this case: group8hdtag and group8hdmso). Also remember to modify outfile name to .8hdmso and 8hdtag.

Save and exit deseq.R.

Now:
```
python3 do_script.py list.CTCF CTCF
```

The result should be a DESeq2... file.
