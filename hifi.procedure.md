## HIFI procedure

All the files below apply for mm10
These include `chr.list`, `intervals.bed` files

### Step 1: Bowtie_results directory
1. go to `bowtie_results/bwt2/K0_combined`
2. create directory `by_chr`, `sorted`
3. copy over from `~/HIFI.bin/scripts/bwt2/` the following files: `do_sort.sh`, `chr.list`, `get.reads.py`, `do_break_down_by_chr.sh`
4. `sbatch ./do_break_down_by_chr.sh K0_combined_mm10.bwt2pairs.bam`
Note that this script has a 2:00 hour time limit. Change it if needed (i.e. dataset too big).
5. `sbatch ./do_sort.sh K0_combined_mm10.bwt2pairs.bam` (maybe optional)

### Step 2: Hic_results directory. Run HIFI.

1. Change to `hic_results` directory. Copy files:
```bash
cp ~/HIFI.bin/scripts/get_hifi_input_2.py .
cp ~/HIFI.bin/scripts/get_hifi_input_cust.py .
cp ~/HIFI.bin/scripts/chr.list .
cp ~/HIFI.bin/scripts/do_one_chr.sh .
cp ~/HIFI.bin/scripts/do_all_chr.sh .
cp ~/HIFI.bin/scripts/sort.py .
cp ~/HIFI.bin/scripts/get.reads.py .
cp ~/HIFI.bin/scripts/intervals.bed .
```
2. Generate intrachromosomal.validPairs
`sbatch ./do_all_chr.sh K0_combined_mm10.bwt2pairs.validPairs`
3. Copy some files
```bash
~/split.py intervals.bed 20
cp ~/HIFI.bin/scripts/do_all_test.sh .
cp ~/HIFI.bin/scripts/do_test.sh .
cp ~/HIFI.bin/mm10_dpnii_8frag.bed .
```
4. Open the file `do_test.sh` to inspect that the enzyme file is correct
Then perform the `do_all_test.sh`:
```bash
for i in `seq 0 19`; do
sbatch ./do_all_test.sh intervals.bed.$i
done
```
5. Plot the matrix, generate the png files. First, copy file:
```bash
cp ~/HIFI.bin/scripts/plot.sh .
cp ~/HIFI.bin/scripts/do_plot.sh .
```
```bash
for i in `seq 0 19`; do
sbatch ./do_plot.sh intervals.bed.$i
done
```
6. **Optional** step.
If for some reasons Step 4 or Step 5 did not finish, run the `inspect.program.py` script to check what are missing, and create the new intervals.bed:
```bash
cp ~/HIFI.bin/scripts/inspect.py inspect.program.py
python3 inspect.program.py intervals.bed
# This will generate a file new.step1, and a file new.step2.
```
```bash
~/split.py new.step1 3
~/split.py new.step2 3
for i in `seq 0 2`; do
sbatch ./do_all_test.sh new.step1.$i
done
for i in `seq 0 2`; do
sbatch ./do_plot.sh new.step2.$i
done
```
Note: in the above `inspect.py`, the chromosomes M, Y are skipped.

### Step 3: Hic_results/hifi.out directory. Run loop calling by Juicer.

1. Loop calling by **Hiccup**. First go to the `hifi.out` directory.
```bash
cd hifi.out
cp ~/HIFI.bin/scripts/hifi.out/do_loops.sh .
cp ~/HIFI.bin/scripts/hifi.out/do_all_loops.sh .
cp ~/HIFI.bin/scripts/hifi.out/callPeaks.8frag .
cp ~/HIFI.bin/scripts/hifi.out/callPeaks.4frag .
cp ~/HIFI.bin/scripts/hifi.out/callPeaks .
```
Check the enzyme name in `do_loops.sh`.
```bash
for i in `seq 0 19`; do
sbatch ./do_all_loops.sh ../intervals.bed.$i
done
```
This should be very quick. Approximately 5 minutes per 10MB file.
It uses the `callPeaks.8frag` program by default, which uses efficient small data structure. Alternatively, one could use the original callPeaks which will take more much more memory and will usually expect a full chromosome interaction matrix as background.
This step should take about **1.5-2 hours**.

2. **Optional**. **FitHiC** loop calling pipeline.
Note this procedure does loop calling on HIFI-imputed matrix.
```bash
cp ~/HIFI.bin/scripts/hifi.out/do.process.fithic.sh .
cp ~/HIFI.bin/scripts/hifi.out/process.fithic.sh .
```
Check the enzyme name in `process.fithic.sh`.
```bash
for i in `seq 0 19`; do
sbatch ./do.process.fithic.sh ../intervals.bed.$i
done
```

3. **Optional**. **FitHiC** loop calling procedure for **non-HIFI matrix** (i.e. raw count).
```bash
cp ~/HIFI.bin/scripts/hifi.out/do.process.fithic.nohifi.sh .
cp ~/HIFI.bin/scripts/hifi.out/process.fithic.nohifi.sh .
```
Check the enzyme name in `process.fithic.nohifi.sh`.
```bash
for i in `seq 0 19`; do
sbatch ./do.process.fithic.nohifi.sh ../intervals.bed.$i
done
```


4. Loop processing steps


First call do_loop_compare.sh
```
mkdir loops.overlap
```

```
ls -1 hifi.out/out2 |grep "loops_.*25kb".txt|sed "s/.25kb.txt//g" > loop.list
python2 ~/split.py loop.list 20
```

```
for i in `seq 0 19`; do sbatch ./do_loop_compare.sh loop.list.$i; done
```
This will call the `check.overlap.2.py` to find overlapping regions across samples. Be sure to modify `check.overlap.2.py` to modify the contrast group conditions.
A directory called `loops.overlap` will be created.

```
for i in `seq 0 19`; do sbatch ./do_loop_compare_2.sh $i; done
for i in `seq 0 19`; do sbatch ./do_loop_compare_3.sh $i; done
```
This script will call the `analyze.overlap.py` script.


Copy several files:
```
cp ~/HIFI.bin/scripts/hifi.out/get.significant.py .
cp ~/HIFI.bin/scripts/hifi.out/remove.duplicate.lines.py .
```
```
for i in 5kb 10kb 25kb 50kb; do cat loops.3/loops_chr*."$i".txt > all.loops.3."$i".txt; done
```
```
for i in 5kb 10kb 25kb 50kb; do python3 get.significant.py all.loops.2."$i".txt short > all.loops.2.significant."$i".txt; done
```
```
for i in 5kb 10kb 25kb 50kb; do python3 remove.duplicate.lines.py all.loops.2.significant."$i".txt > all.loops.2.significant.rmdup."$i".txt; done
```


5. Generation of HIC file

```
cd /labs/sorkin/homes/Wenqing/WC_HiChip_mar28/output/hic_results/data/sample_E135
scp merge.sh qz64@o2.hms.harvard.edu:/n/scratch3/users/q/qz64/XX/output/hic_results/data/KO0/.
```

Open `merge.sh` and modify the enzyme digestion bed file.

```
sbatch ./merge.sh KO0_mm10.bwt2pairs.validPairs
```



