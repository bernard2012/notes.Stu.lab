## Recipe for converting validPairs to hic file

```bash
module load java

srcfile=$1
srcbase=`basename $srcfile .validPairs`
#201811256_E135_Gata1_WC6307_S2_001_mm10.bwt2pairs.validPairs
echo "Step 1"
LC_ALL=C awk '{$4=$4!="+"; $7=$7!="+"; n1=split($9, frag1, "_"); n2=split($10, frag2, "_"); } $2<=$5{print $1, $4, $2, $3, frag1[n1], $7, $5, $6, frag2[n2], $11, $12 }$5<$2{ print $1, $7, $5, $6, frag2[n2], $4, $2, $3, frag1[n1], $12, $11}' $srcfile > step1.valid
ulimit -Sn hard

echo "Step 2"
LC_ALL=C sort -k3,3d  -k7,7d -T . --batch-size=1000 --buffer-size=10g step1.valid > "$srcbase".validPairs.pre_juicebox_sorted
awk 'BEGIN{OFS="\t"; prev_chr=""}$1!=prev{print ""; prev=$1; printf "%s\t", $1} $1==prev {printf "%s\t",$3+1} END{print ""}' /n/scratch3/users/q/qz64/scarlett.c2c12.HiC.rep2/220305_TL9818_fastq/mm10_dpnii.bed | sed "s/\t\n/\n/" | sed "/^$/d" > mm10_dpnii.juicebox

echo "Step 3"
java -jar /home/qz64/juicer_tools.1.7.6_jcuda.0.8.jar pre -f mm10_dpnii.juicebox "$srcbase".validPairs.pre_juicebox_sorted "$srcbase".hic /home/qz64/mm10.chrom.sizes
```
