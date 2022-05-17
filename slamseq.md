
## Slamseq data processing

This tutorial introduces the steps to running Slamdunk.

### Pre-requisites:

- Slamdunk (install using pip, https://github.com/jkobject/slamdunk)
- ngm tools (NextGenMap) (https://github.com/Cibiv/NextGenMap, https://github.com/Cibiv/NextGenMap/wiki)
- Installation of the ngm-tools using the conda route is highly recommended.

### Step 1:

Download the primary assembly FASTA file from Ensembl website. We have downloaded the file and it is called `Mus_musculus.GRCm38.dna.primary_assembly.fa`.

### Step 2:

Download the 3'UTR annotations for the species of interest from UCSC hgTable tool. An example of the file is provided as `mm10.3utr.ensembl.bed`. Select "Gene and Predictions", followed by "knownGenes", then "all of the genome", and in the output option click save to TSV, then select "3'UTR region" on next screen. **Note:** if you have previously performed this step, you may jump right to section "list of output files" at the end of this step.

```
#example
1	3073252	3074322	ENSMUST00000193812.1_utr3_0_0_1_3073253_f	0	+
1	3102015	3102125	ENSMUST00000082908.1_utr3_0_0_1_3102016_f	0	+
1	3252756	3253236	ENSMUST00000192857.1_utr3_0_0_1_3252757_f	0	+
1	3466586	3466687	ENSMUST00000161581.1_utr3_0_0_1_3466587_f	0	+
1	3513404	3513553	ENSMUST00000161581.1_utr3_1_0_1_3513405_f	0	+
1	3531794	3532720	ENSMUST00000192183.1_utr3_0_0_1_3531795_f	0	+
1	3214481	3216021	ENSMUST00000070533.4_utr3_0_0_1_3214482_r	0	-
1	3680154	3681788	ENSMUST00000193244.1_utr3_0_0_1_3680155_f	0	+
1	3752009	3754360	ENSMUST00000194454.1_utr3_0_0_1_3752010_f	0	+
1	4344145	4344599	ENSMUST00000027032.5_utr3_0_0_1_4344146_r	0	-
1	4491712	4491715	ENSMUST00000116652.7_utr3_0_0_1_4491713_r	0	-
1	4490930	4491715	ENSMUST00000027035.9_utr3_0_0_1_4490931_r	0	-
1	4491249	4491715	ENSMUST00000195555.1_utr3_0_0_1_4491250_r	0	-
1	4491389	4491715	ENSMUST00000192650.5_utr3_0_0_1_4491390_r	0	-
1	4496550	4499378	ENSMUST00000193450.1_utr3_0_0_1_4496551_f	0	+
1	4497473	4497654	ENSMUST00000194935.1_utr3_0_0_1_4497474_f	0	+
1	4498006	4498211	ENSMUST00000194935.1_utr3_1_0_1_4498007_f	0	+
```

- I would recommend replacing all "chr1" with "1", i.e. remove all "chr" words from the first column and the 4th column. This is done to be compatible with the chromosome naming in the primary assembly file.
- Sort the bed:
```
sort-bed mm10.3utr.ensembl.bed > mm10.3utr.ensembl.sort.bed
```
- Get the Ensembl Transcript ID to Gene ID using Biomart tool. Save the result to `mm10.ensembl.trans.to.genes.2.tsv`. If you already have this file, skip.
```
#content
Gene stable ID  Gene stable ID version  Transcript stable ID    Transcript stable ID version    Gene name
ENSMUSG00000064372  ENSMUSG00000064372.1    ENSMUST00000082423  ENSMUST00000082423.1    mt-Tp
ENSMUSG00000064371  ENSMUSG00000064371.1    ENSMUST00000082422  ENSMUST00000082422.1    mt-Tt
ENSMUSG00000064370  ENSMUSG00000064370.1    ENSMUST00000082421  ENSMUST00000082421.1    mt-Cytb
```
- The 3'UTR has a lot of overlaps in coordinates. Find 3'UTR clusters:
```
bedmap --echo --echo-map-id mm10.3utr.ensembl.sort.bed > mm10.3utr.ensembl.sort.bedmap
python3 get_cluster.py mm10.3utr.ensembl.sort.bedmap edgelist > mm10.3utr.ensembl.sort.edgelist
# run MCL clustering
~/usr/bin/mcl mm10.3utr.ensembl.sort.edgelist -l 2.0 --abc -o mm10.3utr.ensembl.sort.clusters.2.0
~/usr/bin/mcl mm10.3utr.ensembl.sort.edgelist -l 1.2 --abc -o mm10.3utr.ensembl.sort.clusters.1.2
 ~/usr/bin/mcl mm10.3utr.ensembl.sort.edgelist -l 5.0 --abc -o mm10.3utr.ensembl.sort.clusters.5.0
python3 get_cluster.py mm10.3utr.ensembl.sort.bedmap singleton > singletons.txt
```
- **Test run only** (Optional)
```
python3 read_cluster.py mm10.3utr.ensembl.sort.clusters.1.2
python3 read_cluster.py singletons.txt
```
You should see a bunch of 3'UTR converted to set of genes:
```
{'Gm22489', 'Snord80', 'Gas5', 'Gm50452', 'Gm26224', 'Snord47', 'Gm22357', 'Snord78', 'Gm23212'}
{'Gm24895', 'Rian', 'DQ267100', 'DQ267101'}
{'Gm36199'}
{'Gm24452', 'Gm25855', 'Gm25894', 'Gm24453', 'Snhg1', 'Gm22744', 'Gm22680'}
{'Gm26922', 'Gm23508', 'Rian'}
{'Rian', 'Gm23600'}
{'Snord22', 'Gm23246', 'Snhg1'}
{'Lilr4b'}
...
```

- **Real run**: convert the 3'UTR to 3'UTR clusters, and output the bed file to `joined.utr.txt`.
```
#this will generate utr_cluster_mapping.txt file.
#it will also convert the 3'UTR bed file into a 3'UTR cluster bed file, named joined.utr.txt.
python3 read_cluster_2.py > joined.utr.txt
sort-bed joined.utr.txt > joined.utr.sort.bed
```

- **A list of files that should be generated so far**:
```
Mus_musculus.GRCm38.dna.primary_assembly.fa
mm10.ensembl.trans.to.genes.2.tsv
mm10.3utr.ensembl.sort.bed
mm10.3utr.ensembl.sort.bedmap
mm10.3utr.ensembl.sort.edgelist
mm10.3utr.ensembl.sort.clusters.2.0
singletons.txt
utr_cluster_mapping.txt
joined.utr.txt
joined.utr.sort.bed
```

### Step 3:

- Reads trimming step.
This is the typical trimming submission script: `do_trim.sh`. We use Trimmomatic.
```
#!/bin/bash
#SBATCH -n 1                               # Request one core
#SBATCH -N 1                               # Request one node (if you request more than one core with -n, also using
                                           # -N 1 means all cores will be on the same node)
#SBATCH -t 0-12:00                         # Runtime in D-HH:MM format
#SBATCH -p short                           # Partition to run in
#SBATCH --mem=32000                        # Memory total in MB (for all cores)
#SBATCH -o hostname_%j.out                 # File to which STDOUT will be written, including job ID
#SBATCH -e hostname_%j.err                 # File to which STDERR will be written, including job ID
#SBATCH --mail-type=ALL                    # Type of email notification- BEGIN,END,FAIL,ALL
#SBATCH --mail-user=bernardzhu@gmail.com   # Email to which notifications will be sent

trimmomaticbin=/n/app/trimmomatic/0.36/bin
trimmomaticjarfile=trimmomatic-0.36.jar
adapterpath=/home/qz64/cutrun_pipeline/adapters
bowtie2bin=/n/app/bowtie2/2.2.9/bin
samtoolsbin=/n/app/samtools/1.3.1/bin
javabin=/n/app/java/jdk-11.0.11/bin

infile=$1
#expand the path of infile
relinfile=`realpath -s $infile`
dirname=`dirname $relinfile`
base=`basename $infile _R1_001.fastq.gz`

cd $dirname
workdir=`pwd`

trimdir=$workdir/trimmed
trimdir2=$workdir/trimmed3

>&2 echo "Trimming file $base ..."
>&2 date
$javabin/java -jar $trimmomaticbin/$trimmomaticjarfile PE -threads 1 -phred33 $dirname/"$base"_R1_001.fastq.gz $dirname/"$base"_R2_001.fastq.gz $trimdir/"$base"_1.paired.fastq.gz $trimdir/"$base"_1.unpaired.fastq.gz $trimdir/"$base"_2.paired.fastq.gz $trimdir/"$base"_2.unpaired.fastq.gz ILLUMINACLIP:$adapterpath/Truseq3.PE.fa:2:15:4:4:true LEADING:20 TRAILING:20 SLIDINGWINDOW:4:15 MINLEN:25
```
- Run trimming:
```
sbatch ./do_trim.sh C2C12_30min_slamseq_R1_001.fastq.gz
```

- Prepare the interleaving fastq files. In other words, the _R1_ and _R2_ reads must be presented in the same order.
```
source ~/.init.conda
conda activate myenv
cd trimmed
ngm-utils interleave -1 C2C12_60min_slamseq_1.paired.fastq.gz -2 C2C12_60min_slamseq_2.paired.fastq.gz -o C2C12_60min_slamseq.interleave.fastq
perl separate_readpairs.pl C2C12_60min_slamseq.interleave.fastq
mv forward.fastq C2C12_30min_slamseq.interleave_1.paired.fastq
mv backward.fastq C2C12_30min_slamseq.interleave_2.paired.fastq
```
Note, the files generated from this step are big, as they are FASTQ, not gzipped.

### Step 4:

- Run Slamdunk. Create submission script that looks like the following:

- In my case, the file is called `do_one_2.sh`.
```
#!/bin/bash
#SBATCH -n 4                               # Request one core
#SBATCH -N 1                               # Request one node (if you request more than one core with -n, also using
                                           # -N 1 means all cores will be on the same node)
#SBATCH -t 0-12:00                         # Runtime in D-HH:MM format
#SBATCH -p short                           # Partition to run in
#SBATCH --mem=32000                        # Memory total in MB (for all cores)
#SBATCH -o hostname_%j.out                 # File to which STDOUT will be written, including job ID
#SBATCH -e hostname_%j.err                 # File to which STDERR will be written, including job ID
#SBATCH --mail-type=ALL                    # Type of email notification- BEGIN,END,FAIL,ALL
#SBATCH --mail-user=bernardzhu@gmail.com   # Email to which notifications will be sent

source ~/.init.conda
conda activate myenv

module load samtools
~/.local/bin/slamdunk all -r Mus_musculus.GRCm38.dna.primary_assembly.fa -rl 152 -5 12 -b mm10.3utr.ensembl.bed -t 4 -c 2 -o here -N C2C12_60min ../scarlett_slamseq/220502_TL9991_fastq/trimmed/C2C12_60min_slamseq.interleave_1.paired.fastq ../scarlett_slamseq/220502_TL9991_fastq/trimmed/C2C12_60min_slamseq.interleave_2.paired.fastq
```

- Note: `slamdunk all` consists of `map`, `filter`, and `count` steps.
- Run slamdunk submission script:
```
sbatch ./do_one_2.sh
```

- If you want to just run the slamdunk count step:
```
#!/bin/bash
#SBATCH -n 1                               # Request one core
#SBATCH -N 1                               # Request one node (if you request more than one core with -n, also using
                                           # -N 1 means all cores will be on the same node)
#SBATCH -t 0-12:00                         # Runtime in D-HH:MM format
#SBATCH -p short                           # Partition to run in
#SBATCH --mem=32000                        # Memory total in MB (for all cores)
#SBATCH -o hostname_%j.out                 # File to which STDOUT will be written, including job ID
#SBATCH -e hostname_%j.err                 # File to which STDERR will be written, including job ID
#SBATCH --mail-type=ALL                    # Type of email notification- BEGIN,END,FAIL,ALL
#SBATCH --mail-user=bernardzhu@gmail.com   # Email to which notifications will be sent
~/.local/bin/slamdunk count -r Mus_musculus.GRCm38.dna.primary_assembly.fa -b joined.utr.sort.bed -c 1 -l 153 here/filter/C2C12_30min_filtered.bam -o here/count.joined
```
```
sbatch ./do_slamdunk_count_2.sh
```

### Step 5:

View results. Convert 3'UTR into gene-centric results:
```
#python3 read_utr_count.py
#view results filtering the genes:
python3 read_utr_count.py |sort -t" " -g -k4 -r|grep -v "^Gm"|grep -v "Rik"|grep -v "Mir[0-9]"|grep -v "\-ps" |sort -t" " -g -k3 -r|grep -v "Snord"|less
```
