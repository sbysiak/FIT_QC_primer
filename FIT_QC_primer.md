* [For users](#for-users)
   * [QCG, QCDB and CCDB](#qcg-qcdb-and-ccdb)
      * [Links](links)
      * [Accessing objects in QCDB](#accessing-objects-in-qcdb)
      * [Layouts](#layouts)
   * [Other useful links](#other-useful-links)


* [FIT data structure](#fit-data-structure)
   * [Digits](#digits)
* [QC framework structure](#qc-framework-structure)
   * [QcTasks](#qctasks)
   * [PostProcessingTasks](#postprocessingtasks)
   * [Checkers](#checkers)
* [FIT QC tasks](#fit-qc-tasks)
  * [Configuration](#configuration)
    * [Description of selected parameters](#description-of-selected-parameters)
    * [Note on Trigger Rates](#note-on-trigger-rates)
* [Exemplary commands](#exemplary-commands)
    * [Simulation and digitization of data](#simulation-and-digitization-of-data)
    * [Running QC](#running-qc)
    * [Accessing data from EOS](#accessing-data-from-eos)
    * [Converting compressed timeframes CTF](#converting-compressed-timeframes-ctf)
* [Time Calibration workflow](#time-calibration-workflow)
    * [Running the workflows for FT0/FV0](#running-the-workflows-for-ft0-and-fv0)
    * [Running calibration QC for FT0/FV0](#running-calibration-qc-for-ft0-and-fv0)
    * [Related classes are in O2](#related-classes-are-in-o2)
    * [Configuration options](#configuration-options)
* [How to create and use workflows](#how-to-create-and-use-workflows)
    * [The ControlWorkflows](#the-controlworkflows)
    * [The ECS site](#the-ecs-site)
* [TODO List](#todo-list)




<div style="page-break-after: always;"></div>

# For users


## QCG, QCDB and CCDB

QCG = QC GUI  
QCDB = database containing QC objects, i.e. Monitoring Objects (MO) and Quality Objects (QO)  
CCDB = Conditions and Calibration DB

### Links
* test databases:

  * test **QCG**: https://qcg-test.cern.ch/?page=objectTree
  * test **CCDB/QCDB**: http://ccdb-test.cern.ch:8080/ ( or http://ccdb-test.cern.ch:8080/browse/  to see directory tree)


* production databases:
  * **QCG:** https://ali-qcg.cern.ch/?page=objectTree
  * **QCDB:** http://ali-qcdb.cern.ch:8083/
  * **CCDB:** http://alice-ccdb.cern.ch/


### Accessing objects in QCDB
For example to see instances of object called `Amp_channel0` created by `FV0::DigitQcTask` go to:  
http://ccdb-test.cern.ch:8080/browse/qc/FV0/MO/DigitQcTask/Amp_channel0
![ccdb_links](graphics/screens/ccdb_links.png)

One can find the path to the object by clicking *`i`* icon in qcg ![qcg_i_icon](graphics/screens/qcg_i_icon.png)

### Layouts
Layouts are used to create interactive dashboards containing several plots and organized in desired way. They are created in simple drag-and-drop manner. Layouts contain tabs, one can also specify certain plotting options for each object.
One can create new layout by clicking `+` on the left panel in qcg:
![create_layout](graphics/screens/create_layout.png)

Request to support config files for layouts is pending: https://alice.its.cern.ch/jira/browse/OGUI-890.

## Other useful links

* QC repo: https://github.com/AliceO2Group/QualityControl
* documentation for shifters: https://alice-qc-shifter.docs.cern.ch/
* accessing GPN protected websites from outside of CERN: ~~https://gitlab.cern.ch/bvonhall/dynamic-forwarding/-/blob/master/README.md~~  
* or https://estarter.github.io/linux/cern/2014/12/17/access-internal-website-via-ssh-proxy.html
* FoxyProxy could keep the proxy settings: https://addons.mozilla.org/hu/firefox/addon/foxyproxy-standard/
* Example JSON file to access CERN CCDB:
``` {
  "logging": {
    "size": 100,
    "active": false
  },
  "mode": "disabled",
  "x71e991636990457871": {
    "type": 3,
    "color": "#66cc66",
    "title": "CERN",
    "active": true,
    "address": "127.0.0.1",
    "port": 1080,
    "proxyDNS": true,
    "username": "",
    "password": "",
    "whitePatterns": [
      {
        "title": "all URLs",
        "active": true,
        "pattern": "*",
        "type": 1,
        "protocols": 1
      }
    ],
    "blackPatterns": [],
    "pacURL": "",
    "index": 9007199254740990
  },
  "browserVersion": "94.0.1",
  "foxyProxyVersion": "7.5.1",
  "foxyProxyEdition": "standard"
}
```

-------------------------------

<div style="page-break-after: always;"></div>

# FIT data structure:

## Digits
So far all FIT QC process data in `digit` format. The structure is following (for FV0 - FT0 and FDD may slightly differ):

![alt_text](graphics/screens/digits_structure.png)

In short, FV0DigitCh contains (channelID, time, charge) for each hit coming from PM and FV0DigitBC contains:
- InteractionRecord, i.e. precise time information via storing Orbit and BC (convertable to nanosec w.r.t certain time 0 by `o2::InteractionRecord::bc2ns`)
- information from TCM: triggerSignals (bit-by-bit info about fired triggers) and several quantities computed on TCM like `nChanA` (number of fired channels on A side), `amplA` (sum of amplitudes on side A **divided by 8**). For FT0 there is also `timeA` (average time on A side) and equivalent fields for C side.


<div style="page-break-after: always;"></div>

# QC framework structure

QC framework defines 3 main classes described in next sections: QcTasks, PostProcessingTask and Checker, which are base for the code developed by each detector team.

There are two types of QC objects: MonitoringObjects (MO) and QualityObjects (QO).
MO are ROOT objects, typically histograms or graphs. QO are basically numbers specifying quality of given MO (1=Good/2=Medium/3=Bad and 10=Null = the worst). Qos and be [aggregated](https://github.com/AliceO2Group/QualityControl/blob/master/doc/ModulesDevelopment.md#quality-aggregation).

*As of today (15.11.2021), the policy for storage is following: everything is kept for 24h, after that only one version of each MO/QO per RUN is preserved. Trending can be used to keep information about the time evolution with granularity greater than 1/h.*

## QcTasks:
They take as an input all data in form of digits so are able to compute everything. They create and publish MOs.

## PostProcessingTasks:
Take as the input MO and create new MO. They can work asynchronously. Useful if one wants to combine objects from different sources (e.g. BC pattern from CCDB).

Especially useful example of postprocessing provided in Common part is [TrendingTask](https://github.com/AliceO2Group/QualityControl/blob/master/doc/PostProcessing.md#the-trendingtask-class). One can create trending plot for most commonly used quantities (like mean, stddev, entries of histogram) by adding just several lines to the config file.

## Checkers:
Take as the input MO (or list of MOs) and determines its quality encapsulated into QO.

Useful part of the checker is `beautify()` function (called only if the input is single MO). It allows to modify visual aspects of the input MO - one can for instance modify colors to reflect the quality of the object or add `TPaveText`, which will provide a message or instructions for the shifter.


<div style="page-break-after: always;"></div>

# FIT QC tasks

List of tasks implemented for FIT (FV0 and FT0 for now):
- DigitQcTask -- main QcTasks, dumps to 1- and 2D histograms most of the information available in digits. In particular it generates separately BC distributions for: all events, for each trigger and for each FEE piece (PM and TCM).
It also contains functionality from old `TriggerQcTask` -- taking information from PMs and repeating the logic implemented on TCM to validate the triggers

- DigitQcTaskLaser and TH1ReductorLaser -- similar to `DigitQcTask` with limited functionality, to be used in laser runs. Reductor is responsible for fitting amplitude distribution for each channel, including reference channel, which contains two peaks.

- PostProcTask -- postprocessing task which computes trigger rates and combines registered BC distribution with the LHC filling scheme (aka BC pattern) and finds the events which are out-of-the-bunch.

- OutOfBunchCollCheck - compares ratio of out-of-the-bunch events to all events with warning/error thresholds.

- CFDEffCheck -- for each channel computes efficiency of getting charge information when time information was available (charge is not available - and set to zero - in case of time miscalibration).

- GenericCheck -- checker which implements series of simple checks, like histogram (1D/2D) mean within predefined range, stddev below certain threshold, last point of graph within range, overflow bin to integral ratio below threshold. It can be also easily extended for any check which requires comparison of value extracted from single MO with some threshold. Each single check is activated by specifying its warning and error thresholds in the config

- CalibrationTask and ChannelTimeCalibrationCheck -- check performance of time offset calibration procedure

## Configuration

### Description of selected parameters:

- `DigitQcTask`:
  - "ChannelIDs" - selects channels for which separate ampl/time histogram will be generated. Meant to limit number of produces objects.
  - "ChannelIDsAmpVsTime" - same but for AmpVsTime 2D histograms per channel. This histogram is more important because this correlation cannot be retrieved from 2D: TimeVsChannel or AmpVsChannel
  - "binning_\<MoName\>" - parameter to modify binning of histogram
  - triggers settings for trigger validation in software, description for FT0:
    - "trgThresholdTimeLow/High" - time window for vertex trigger
    - "trgModeSide" - mode for central and semi-central triggers, may operate on either side, their sum or AND condition for both sides, options: "A", "C", "A+C", "A&C"
    - "trgModeThresholdVar" - mode for central and semi-central triggers, which can operate either on amplitudes or number of fired channels, options: "Ampl", "Nchannels"
    - "trgThreshold(S)Cen[A/C/Sum]" - trigger threshold for (semi-)central trigger, expressed in ADC channels or number of channels depending on "trgModeThresholdVar"
- `CFDEffCheck`:
  - "deadChannelMap" - list of channels which are excluded from the check (they will not raise any alarm)
- `PostProcTask`:
  - "timestampSourceLhcIf", chooses which instance of `GRPLHCIFData` (containing BC pattern) should be used by selecting source of timestamp used to query the CCDB, see also: [this topic](https://alice-talk.web.cern.ch/t/access-to-timestamp-of-data/1309/2) for some discussion. Options:
    - "last" - picks the last available object from the CCDB
    - "trigger" - uses object which is valid at the timestamp provided by the `Trigger` class object passed to `PostProcTask::update`
    - "metadata" - uses object valid at the timestamp extracted from the metadata of MO "BCvsTriggers" (key="TFcreationTime"). This field contains timestamp propagated from `ProcessingContext::services().get<o2::framework::TimingInfo>().creation` in `DigitQcTask::monitorData()`. Once this is filled properly for , this options could be used also in online running.
    - "cycleDurationMoName" - name of MO which should be used in trigger rates calculations. See: [Note on Trigger Rates](#note-on-trigger-rates)
    - "init/updateTrigger" - for now (2022.10.05) it's important to trigger on the last object added to the object manager in `DigitQcTask`, otherwise it may result in picking object from different cycles due to tiny differences in their timestamps. Currently, it's [FV0/FT0/FDD]/MO/DigitQcTask/TriggersCorrelation. The same is valid for any PP task, in particular for `TrendingTask`. [This jira ticker](https://alice.its.cern.ch/jira/browse/QC-826) should fix it.
- `GenericCheck`: each simple check has two parameters which are warning and error thresholds. If parameter contains "Min" ("Max") then the value has the lower (upper) restriction. For instance "thresholdWarningMaxStddevX" and "thresholdErrorMaxStddevX" put on limit on the maximum allowed standard deviation on x-axis (no limit from the bottom). Window of allowed values is obtained as two separate checks, e.g.
      "thresholdWarningMinMeanX":"-40",
      "thresholdErrorMinMeanX":"-50",
      "thresholdWarningMaxMeanX":"40",
      "thresholdErrorMaxMeanX":"50"
corresponds to condition: Warning if abs(mean(x))>40 and  Error if abs(mean(x))>50


### Note on Trigger Rates

Trigger rates are computed by taking number of entries in each bin in `qc/FV0/MO/DigitQcTask/Triggers` by cycle duration. Currently the cycle duration is computed using 3 different methods (after tests with real beam one method should be chosen and the rest removed):
1. convert InteractionRecord to nanosec and find its max and min value within one cycle. Then duration = max - min.
2. convert InteractionRecord to nanosec and compute duration (as above) **for each processed timeframe (TF)**. Cycle duration = sum of durations of all TFs.
3. count TFs. Cycle duration = (#TF) * (#orbits per TF) * (orbit duration)

(#orbits per TF) = parameter (128 or 256).  
(orbit duration) = `o2::constants::lhc::LHCOrbitNS` = 88924.596 ns = 88 µs


First method fails if subsampling is used so methods 2. and 3. are prefered (cycle duration is similar as if no subsumpling is used but number of triggers is reduced).

In case of no subsampling there is still small difference between 1. and 2.: the reason is illustrated below: (duration total) is not exactly equal to (duration1+duration2+duration3) due to small spaces between first collision in TF *n+1* and last collision in TF *n*. The difference is small but visible in case of 1 kHz laser.

![alt_text](graphics/screens/cycle_duration.png)

<div style="page-break-after: always;"></div>

# Exemplary commands

### Simulation and digitization of data
1. Generate events and transport particles through detectors  
`o2-sim -g pythia6 -e TGeant3 -m FV0 FT0 FDD -j 2 -n 100`
2. Digitize hits only from selected detector (in my case it was FT0)  
`o2-sim-digitizer-workflow --onlyDet FT0 -b`  
Now file ft0digits.root should be generated in current workdir, one can use this file or follow next
steps to get more experiment like data
3. Convert MC digits to RAW data format as in experiment  
`o2-ft0-digi2raw`
4. Convert back RAW data format to digits  
`o2-raw-file-reader-workflow --input-conf FT0raw.cfg | o2-ft0-flp-dpl-workflow -b`

### Running QC

- on digits:  
`o2-fv0-digit-reader-workflow --fv0-digit-infile o2_fv0digits.root | o2-qc --config json:///qc_config.json -b`

- on digits in raw-TF files:  
`o2-raw-tf-reader-workflow --input-data run0504494/ | o2-fv0-digits-writer-workflow --disable-mc -b`

- on RAW in raw-TF files:  
`o2-raw-tf-reader-workflow --raw-only-det FV0 --input-data run0504494/ | o2-fv0-flp-dpl-workflow -b`


### Accessing data from EOS
listing files:  
- `xrdfs root://eosaliceo2.cern.ch ls -l /eos/aliceo2/raw/2021/OCT/505673/`

copying to local:  
- `xrdcp  -r --parallel 4  root://eosaliceo2.cern.ch/////eos/aliceo2/raw/2021/OCT/505673/raw/0640/ ./`


### Converting compressed timeframes CTF

- to digits (for FV0 trigger information may not be filled):  
`o2-ctf-reader-workflow --onlyDet FV0 --ctf-input path/to/o2_ctf_file.root -b --ctf-dict path/to/ctf_dictionary.root | o2-fv0-digits-writer-workflow --disable-mc -b`

- to reco (to be validated):  
`o2-ctf-reader-workflow --onlyDet FV0 --ctf-input path/to/o2_ctf_file.root -b --ctf-dict path/to/ctf_dictionary.root | o2-fv0-digits-writer-workflow --disable-mc -b`

<div style="page-break-after: always;"></div>

# Time Calibration workflow

Workflow to calibrate the measured time-distributions of particles by FT0/FV0 detectors. The measured time might be shifted with respect to the LHC clock. Calibration workflow produces these correction shifts (offsets) for each detector channel and stores them as ROOT objects (vectors) into CCDB  (http://ccdb-test.cern.ch:8080/browse/FV0/Calibration/ChannelTimeOffset). Later on for calibrating data, these offsets are applied to the incoming data. This procedure is checked by running the QC:

### Running the workflows for FT0 and FV0:

- `o2-ft0-digits-reader-workflow --ft0-digit-infile <path-to-digits-file> | o2-calibration-ft0-tf-processor | o2-calibration-ft0-channel-offset-calibration | o2-calibration-ccdb-populator-workflow`
- `o2-fv0-digit-reader-workflow --fv0-digit-infile <path-to-digits-file> | o2-calibration-fv0-tf-processor | o2-calibration-fv0-channel-offset-calibration | o2-calibration-ccdb-populator-workflow`

### Running calibration QC for FT0 and FV0:

- `o2-ft0-digits-reader-workflow --ft0-digit-infile <digit-root-file> | o2-qc --config json:///$HOME/alice/QualityControl/Modules/FV0/fv0-calibration-config.json -b`
- `o2-fv0-digit-reader-workflow --fv0-digit-infile <digit-root-file> | o2-qc --config json:///$HOME/alice/QualityControl/Modules/FT0/ft0-calibration-config.json -b`

### Related classes are in O2:
O2/Detectors/FIT/FV0/calibration/src/FV0ChannelTimeTimeSlotContainer.cxx (similar for FT0)
- To calculate the offset values (gaus-fit is implemented)
- To find relavent classes for workflows: Check CMakeList.txt & search there for 'o2_add_executable'

for example:  
`o2_add_executable(fv0-channel-offset-calibration
  COMPONENT_NAME calibration
  SOURCES workflow/FV0ChannelTimeCalibration-Workflow.cxx`

### Configuration options:
While running the calibration workflow you can specify following options to have better calibration (add these options after o2-calibration-fv0-channel-offset-calibration ):
- `--tf-per-slot=N` : It will start the calibration after N timeframes are accumulated (N should be such number which can have sufficient data for calibration)
- `--time-calib-fitting-nbins=M` : This option for specifying histogram range for fitting gaussian .i.e M bins per side of the histogram maxima. (This will help to have robust fitting)
- `--updateInterval` : Options available but not used. Check the details here https://github.com/AliceO2Group/AliceO2/tree/dev/Detectors/Calibration#timeslotcalibrationinput-container
- `--max-delay`


# How to create and use workflows

In this section we describe how to create a workflow and how to use it in the [ECS](https://ali-ecs.cern.ch/?page=newEnvironment). As an example, we give the details of the
creation of the FV0 laser QC.

## The ControlWorkflows

First of all, have a local clone from the [ControlWorklows](https://github.com/<your_git_user_name>/ControlWorkflows) repository. Once it is forked on the site, in bash, it can be cloned with
`git clone https://github.com/AliceO2Group/ControlWorkflows.git` command. I also did `git remote -v` and `git remote add origin git@github.com:<your_git_user_name>/ControlWorkflows`.
May be these are not necessary.

The repository contains a quite extensive README which I recommend to read. Besides, there are the three main directories:
* scripts: this directory contains the scripts which generate the workflows and the `.json` files for the configurations of the workflows in the `scripts/etc` subdirectory. This is where your `.json` files should go.
* tasks: the generation script will produce `.yaml` files that corresponds to your tasks in the `.json` which go here.
* workflows: the generation script will produce `.yaml` file that contains your tasks and goes here.

All you need to do is the followings:

* copy your `.json` file into the `scripts/etc`
* write your on generation script (or just copy and modify an existing one) and run it
* the name of the new workflow also should be added to `workflows/readout-dataflow.yaml` file. Find the group of workflows for your detector and add it there.
* Push the changes into your repository
    * git status   #shows which are the files that are modified or created, i.e., to be pushed
    * git add <filename>
    * git commit -m "Something descriptive"
    * (for the first time) git remote add origin git@github.com:<your_git_user_name>/<your_repo> (e.g.: git remote add origin git@github.com:sandor-lokos/ControlWorkflows)
    * git push -u origin master
* write an e-mail to a corresponding person to add your git repository to ECS (when I did this the person was Roberto Divia (Roberto.Divia@cern.ch))
* upload your .json file to [Consul](https://ali-consul-ui.cern.ch/ui/alice-o2-cluster/kv/o2/components/qc/ANY/any/) with the naming convention you find there.
E.g. in my case `fv0-digits-qc-laser.json` transformed to `fv0-digits-qc-laser-alio2-cr1-flp180.json`. The postfix is related to the FLP version (I think).

And you are all set and ready to use your workflow on the [ECS](https://ali-ecs.cern.ch/?page=newEnvironment). Let's see how that works.

## The ECS site

On the [ECS site](https://ali-ecs.cern.ch/?page=newEnvironment), in the left hand menu, the top is the **+CREATE**. There under Repository you should find your own. Select it!

From this point, the settings are depending on what workflow you have and what do you want to do. I will describe my case of FV0 laser QC but note that your case could be different.

* Revision: master (or whatever branch you have), Workflow: readout-dataflow, Detector selection: FV0, FLP selection: alio2-cr1-flp180 (or similar)
* Under Advanced configuration   Key: dpl_workflow  value: fv0-digits-qc-laser and then click +
* EPN Workflows: Expert, then # of EPNs: 1, Extra ENV variables: WORKFLOW_DETECTORS_FLP_PROCESSING=FV0 (at the moment, we have this but it hopefully will be changed)
* TRG: fv0_cal.par or fv0_cont.par or something else
* Click on Create

After awhile your task will be Configured (STANDBY -> DEPLOYED -> CONFIGURED). If the status turns to be <span style="color:red"> ERROR text</span> then something went wrong with the configurations.
If it is <span style="color:#ffca33"> CONFIGURED text</span> then in the upper left corner you should see the a Start button. Click on that and wait for a couple of seconds. Your workflow will start. If you have any output
histogram(s) which goes to the [QCG database](https://ali-qcg.cern.ch/?page=objectTree) then you can check if they appear or not. If not, then check the database settings compared to other `.json` files available on
[Consul](https://ali-consul-ui.cern.ch/ui/alice-o2-cluster/kv/o2/components/qc/ANY/any/) .

# TODO List
* improve triggers validation in software by implementing exactly the same logic as on TCM including information from bits (docs needed)
* add plot with time deltas between channels, filled on event-by-event basis (to estimate time resolution)
* unify settings in TriggerQcTask to be similar to the menu in ControlServer (e.g. thresholds in number of ADC channels VS in MIPs)
