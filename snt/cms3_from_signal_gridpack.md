# CMS3 from signal gridpack

## Overview
Want to make
* m(stop) = 1600 GeV
* m(lsp) = 1 GeV
* number events = O(50k)
following the RunIISpring16 recipes.

## Instructions

### Making the FSPremix PSet

The workflow on McM is [here](https://cms-pdmv.cern.ch/mcm/chained_requests?contains=SUS-RunIISpring16MiniAODv2-00135&page=0&shown=15)
which has two steps (FSPremix, MiniAOD). Clicking on the first link in the chain column, then clicking on the down arrow in a circle (get setup) gives you
[this](https://cms-pdmv.cern.ch/mcm/public/restapi/requests/get_setup/SUS-RunIISpring16FSPremix-00017).

Copy the setup file locally, comment out the cmsDriver line for now, and execute it to set up the environment. Since this is a scan, the genfragment
will be a loop over PSets for all the mass points, and will thus result in a PSet that is a few MB large when running the cmsDriver command.
We only want one point, so inside `CMSSW_8_0_5_patch1/src/Configuration/GenProduction/python/SUS-RunIISpring16FSPremix-00017-fragment.py`, 
edit the loop over points:
```python
for point in mpoints:
  mstop, mlsp = point[0], point[1]
```
to become
```python
for point in [[1600,1,1.]]:
  mstop, mlsp = point[0], point[1]
```
the last element is the weight of the point, which doesn't matter if you're setting the xsec manually later.

Now back in the driver command, change
`--pileup_input dbs:/Neutrino_E-10_gun/RunIISpring16FSPremix-PUSpring16_80X_mcRun2_asymptotic_2016_v3-v1/GEN-SIM-DIGI-RAW`
to
`--pileup_input dbs:"/Neutrino_E-10_gun/RunIISpring16FSPremix-PUSpring16_80X_mcRun2_asymptotic_2016_v3-v1/GEN-SIM-DIGI-RAW site=T2_US_UCSD*" --pileup_dasoption="--limit 0"`
if there's still the issue with PU samples (otherwise keep it as is).

Now execute the cmsDriver command to get the first PSet.

### Making the miniAOD PSet

Repeat the step of getting the setup command starting from [here](https://cms-pdmv.cern.ch/mcm/chained_requests?contains=SUS-RunIISpring16MiniAODv2-00135&page=0&shown=15)
and going to the MiniAOD link. Clicking on the down arrow in a circle gives you
[this](https://cms-pdmv.cern.ch/mcm/public/restapi/requests/get_setup/SUS-RunIISpring16MiniAODv2-00135).

This one runs out of the box without modifications to give you another PSet.

### Combining the PSets into a workflow on Condor

#### Installing ProjectMetis

1. `git clone https://github.com/aminnj/ProjectMetis/`
2. `cd ProjectMetis; source setup.sh`

#### Submitting

1. Now make a submission script called `chain.py` with the following content, changing at the minimum the lines marked by `FIXME`.

```python
from metis.CMSSWTask import CMSSWTask
from metis.Sample import DirectorySample, DummySample
from metis.Path import Path
from metis.StatsParser import StatsParser
import time

# files will go into /hadoop/cms/store/user/<user>/<special_dir>/<unique_name>
# where <unique_name> is a combination of the dataset name and the proc_tag
proc_tag = "v3"
special_dir = "private/ProjectMetis/"
step1 = CMSSWTask(
        # keep STEP1 at the end of the dataset name because we use it in the next part
        sample = DummySample(N=1, dataset="/T2tt/mStop1600-LQ/STEP1"),
        tag = proc_tag,
        special_dir = special_dir,
        events_per_output = 750, # control the splitting of events per job
        total_nevents = 150000,  # and total nevents
        split_within_files = True,
        pset = "pset_lq_step1.py", # FIXME give path to FSPremix PSet from above
        cmssw_version = "CMSSW_8_0_5_patch1",
        scram_arch = "slc6_amd64_gcc530",
        )

step2 = CMSSWTask(
        sample = DirectorySample(
            location = step1.get_outputdir(),
            dataset = step1.get_sample().get_datasetname().replace("STEP1","STEP2"),
            ),
        tag = proc_tag,
        special_dir = special_dir,
        open_dataset = True,
        files_per_output = 10, # 10 files per job
        pset = "pset_lq_step2.py", # FIXME give path to miniAOD PSet from above
        cmssw_version = "CMSSW_8_0_5_patch1",
        scram_arch = "slc6_amd64_gcc530",
        )

for _ in range(25):
    total_summary = {}
    for task in [step1,step2]:
        task.process()
        summary = task.get_task_summary()
        total_summary[task.get_sample().get_datasetname()] = summary
    StatsParser(data=total_summary, webdir="~/public_html/dump/metis/").do()
    time.sleep(30*60)
```

2. Make sure you have a valid proxy.
3. Open up a screen, run `source setup.sh` again to be sure, and run `python chain.py` to submit jobs to Condor.
4. In a nutshell, this submits jobs to go from gridpack to miniAOD in two steps. Visit the link that
gets printed out to see a visual status of the progress, as well as the output location, if you click on the 
dataset names.
5. When step 2 is sufficiently done, proceed to making CMS3 from miniAOD.

### Making CMS3

Since these are 2016 samples, the legacy tools will have to be used. :(
In the future, everything could be done start-to-finish with Metis.

#### Installing AutoTwopler

1. `git clone https://github.com/cmstas/NtupleTools/`
2. `cd NtupleTools/AutoTwopler`
3. Make sure `params.py` has `campaign = "80X_moriond"`, as this is what was used for 2016 samples.
4. Run `source setup.sh` and make sure you have a valid proxy (take note of the dashboard URL).

#### Making text files

As this is a custom dataset, we need to make a filelist. Create a file list called something creative like `filelist.txt`.
It should contain a dummy dataset name for bookkeping purposes only (which I just copied from an official dataset name
and modified the masses), a splitting parameter (3 files per job), and the list of files, one per line. The list of files
is just an `ls` of the hadoop output of step 2 from before, but with `/hadoop/cms` stripped off, so that CRAB can accept
these files as xrootd-accessible.
```
dataset: /SMS-T2tt_mStop1600-mLSP1/RunIISummer16MiniAODv2-PUSummer16Fast_80X_mcRun2_asymptotic_2016_TrancheIV_v6-v1/MINIAODSIM
files_per_job: 3
/store/user/namin/private/ProjectMetis//T2tt_mStop1600-LQ_STEP2_v3/output_1.root
/store/user/namin/private/ProjectMetis//T2tt_mStop1600-LQ_STEP2_v3/output_2.root
/store/user/namin/private/ProjectMetis//T2tt_mStop1600-LQ_STEP2_v3/output_3.root
/store/user/namin/private/ProjectMetis//T2tt_mStop1600-LQ_STEP2_v3/output_4.root
```

Next, add a line to `instructions.txt` to represent a sample submission. Modify the first part to give the full
path to your file list from before. Next, the global tag, xsec, kfactor, filter efficiency, and list of SUSY parameters.
For a Spring16 campaign, only the filelist needs to be changed in the example below. (The number of SUSY parameters
must match the sample, of course -- otherwise CMS3 crashes :(.)
```
/home/users/namin/2016/moriond/NtupleTools/AutoTwopler/filelist_t2tt1600.txt 80X_mcRun2_asymptotic_2016_TrancheIV_v6 1 1 1 mChi, mLSP
```

#### Submitting

1. Make sure you have a valid proxy.
2. Open up a screen, run `source setup.sh` again to be sure, and run `python run.py instructions.txt` to submit CRAB jobs
3. AutoTwopler manages the CRAB jobs, merges the outputs, and the progress can be seen in the URL printed out when you run `source setup.sh`.

#### Success

If all goes well, there will be CMS3 in the `finaldir` (see the AutoTwopler dashboard and click the dataset).

