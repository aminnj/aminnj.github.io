# Bulk file transfer

## Overview
Rather than try to spam `gfal-copy` commands to move files between storage elements, we can
use a bulk data mover provided by CERN (FTS3). Full documentation for FTS is available 
[here](http://fts3-docs.web.cern.ch/fts3-docs/docs/cli/cli.html). Basically, the user provides
a two column text file with input and output file pairs as a filelist to FTS. FTS then manages
the moving (in parallel) and resubmits failures.

## Preparation
In a specific case, I had a filelist containing pairs of nfs paths and hadoop paths. I wanted
to move the UCSD hadoop files to files in Nebraska with a filename matching the nfs path.
```text
/nfs-7/userdata/dataTuple/kludge/Run2016H_MET_MINIAOD_03Feb2017_ver3-v1/merged/V08-00-18/merged_ntuple_1.root /hadoop/cms/store/user/namin/dataTuple/Run2016H_MET_MINIAOD_03Feb2017_ver3-v1/V08-00-18/MET_MINIAOD_03Feb2017_ver3-v1_80000_0EC2239F-8AEA-E611-9FB3-008CFA111218.root
/nfs-7/userdata/dataTuple/kludge/Run2016H_MET_MINIAOD_03Feb2017_ver3-v1/merged/V08-00-18/merged_ntuple_2.root /hadoop/cms/store/user/namin/dataTuple/Run2016H_MET_MINIAOD_03Feb2017_ver3-v1/V08-00-18/MET_MINIAOD_03Feb2017_ver3-v1_80000_2A9DE5C7-ADEA-E611-9F9C-008CFA111290.root
/nfs-7/userdata/dataTuple/kludge/Run2016H_MET_MINIAOD_03Feb2017_ver3-v1/merged/V08-00-18/merged_ntuple_3.root /hadoop/cms/store/user/namin/dataTuple/Run2016H_MET_MINIAOD_03Feb2017_ver3-v1/V08-00-18/MET_MINIAOD_03Feb2017_ver3-v1_80000_6AD62BA4-8AEA-E611-95C4-008CFA1979A0.root
```

And I can convert this into a suitable format for FTS with a script like
```bash
#!/bin/bash

UCSDPFX="gsiftp://gftp.t2.ucsd.edu"
UNLPFX="gsiftp://red-gridftp.unl.edu//mnt/hadoop/user/uscms01/pnfs/unl.edu/data4/cms"

while read -r nfspath hadooppath
do
    aftergroup=$(echo $nfspath | sed 's|/nfs-7/userdata/dataTuple/kludge/||')
    grouppath="/store/group/snt/${aftergroup}"
    from="${UCSDPFX}${hadooppath}"
    to="${UNLPFX}${grouppath}"
    cmd="gfal-copy -p -f -t 4200 --verbose $from $to  --checksum ADLER32"
    # echo $cmd # <-- we don't want to do this 
    echo "$from $to" # <-- simply echo out from and to and let FTS take care of copying
done < "filelist.txt"
```

Originally, I was going to spit out tons of `gfal-copy` commands and make the job manager myself
but FTS takes care of that for us, so we only need to echo out the 'from' and 'to' paths (with apppropriate
redirectors, of course). Execute such a script and redirect the output via `./script.sh > tosubmit.txt`.

Next, submit the transfer request (`-o` will overwrite files already existing in the output SE; `man fts-transfer-submit`
shows other options):
```
fts-transfer-submit -o -s https://fts3-pilot.cern.ch:8446 -f tosubmit.txt
```

This spits out a token like `10876666-a5cb-11e7-873a-02163e00a17a`, which is then put into a search box in
the [monitoring page](https://fts3-pilot.cern.ch:8449/fts3/ftsmon/#/).

Note: `fts-<TAB><TAB>` shows other FTS commands to cancel or get the status from the command line.

## Miscellaneous
SE names/redirectors can be found from 
```
cat /cvmfs/cms.cern.ch/SITECONF/T2_US_Nebraska/JobConfig/site-local-config.xml
cat /cvmfs/cms.cern.ch/SITECONF/T2_US_Nebraska/PhEDEx/storage.xml
```

More details are [here](https://wiki.chipp.ch/twiki/bin/view/CmsTier3/HowToAccessSe).