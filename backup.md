## Backup scripts

```
rsync --dry-run -av qz64@transfer.rc.hms.harvard.edu:/n/scratch2/qz64/Brandon_mar9 .
rsync -avun --delete qz64@transfer.rc.hms.harvard.edu:/n/scratch2/qz64/Brandon_mar9 .
rsync -avun --delete . qz64@transfer.rc.hms.harvard.edu:/n/scratch2/qz64/Brandon_mar9
rsync -avun --delete Brandon_mar9 qz64@transfer.rc.hms.harvard.edu:/n/scratch2/qz64/Brandon_mar9
rsync -avun --size-only --delete . qz64@transfer.rc.hms.harvard.edu:/n/scratch2/qz64/Brandon_mar9
rsync -avun --size-only --delete qz64@transfer.rc.hms.harvard.edu:/n/scratch2/qz64/Brandon_mar9 .
rsync -avn --size-only --delete qz64@transfer.rc.hms.harvard.edu:/n/scratch2/qz64/Brandon_mar9 .
rsync -av --size-only qz64@transfer.rc.hms.harvard.edu:/n/scratch2/qz64/Brandon_mar9 .
rsync -avn --size-only --delete qz64@transfer.rc.hms.harvard.edu:/n/scratch2/qz64/bing_ren_cardio .
```

## Incremental backup scripts:

### Step 1:
```
rsync -avn --size-only --delete qz64@transfer.rc.hms.harvard.edu:/n/scratch3/users/q/qz64/stuti.atacseq . > list.stuti
```

### Step 2:
Remove directories (anything ends in "/") in list.stuti
Can be done in VIM via:
```
:
%s,.*\/\n,,g
```

### Step 3:
```
#put this in a bash script, such as do.stuti.sh
rsync -av --files-from=list.stuti qz64@transfer.rc.hms.harvard.edu:/n/scratch3/users/q/qz64 . > list.stuti.nohup 2>&1
chmod a+x do.stuti.sh
nohup ./do.stuti.sh &
```

### Step 4:
Be careful about symlink directories as they may be removed in Step 2. For these directories, it is better to create a separate file list containing symlink directories, and sync separately. 
To check for these, on the host machine, type:
```
find . -type l -ls
```

### Step 5: What to do with symlinks:

Get symlink lines in VIM:
```
:
%s,rsync: link_stat ,,g
:
%s, failed: No such file or directory (2),,g
:
%s,^",,g
:
%s,"$,,g
```
Save the file and move it to the home directory: /labs/sorkin/homes/Qian
Say I named it "rename1"
Then:
```
python3 do_rename.py rename1
```

