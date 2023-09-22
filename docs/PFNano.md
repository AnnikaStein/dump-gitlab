# PFNano

Click through the tabs for the basics, CRAB-specific FAQ, or developing features yourself, depending on what you want to do with PFNano.

=== "Main instructions (for Run 3)"

    ## Current PFNano status and plans
    PFNano is a collection of producers that update / add information on top of the regular NanoAOD data tier, related to flavour tagging algorithms and the low-level quantities associated with them - in particular PFCandidates, hence the name. It can be used to do basically all offline BTV software and calibration tasks, once configured and extended with the relevant variables. This enables looking not only at tagger scores, but also adds the individual tagger's inputs, allowing a) [tagger training](https://git.rwth-aachen.de/annika-stein/ai_safety_2021), b) [Run 3 commissioning](https://github.com/cms-btv-pog/BTVNanoCommissioning/tree/master), c) [SF derivation](https://github.com/AnnikaStein/CustomTagger-Data-MC), ... and probably much more, for example performing entire physics analyses that profit from low-level information.
    It has been used for evaluating data / MC of new algorithms including DeepJet (PyTorch version) with / without adversarial training, (Robust)ParTAK4 on the fly, even before integration in CMSSW was finished. Its flexibility also allows moving from the BTA framework to this one for performance / calibration.
    Mind that we regularly need to verify that producers, inputs and outputs are in line with the official NanoAOD and general CMSSW developments, as we want to avoid duplicate branches and not introduce compilation errors by accessing non-existing variables. An ordered list of variables, their status and what you need to do is [here](https://docs.google.com/spreadsheets/d/17BPqUe0Lx1LuNQhM9pqSunUMeYDCdqM49Nnctuti__I/edit?usp=sharing). The recommended branch for PFNano Run 3 commissioning / SFs in 2023 aligns with version 13 of the CMSSW release cycle, for 2022, the major version is 12. More details below.

    ## Instructions

    We start by checking out a recent CMSSW version. If you are using a non-lxplus site, you might need to setup cms- and grid-specific commands first. You can use this for convenience and save as `~/.cmsrc` for example, at NAF-DESY:

    ```shell
    # if you want to source grid-commands, crab3, cmsset for naf-cms, source ~/.cmsrc
    # should not be all necessary for lxplus
    source /cvmfs/grid.desy.de/etc/profile.d/grid-ui-env.sh # or /cvmfs/grid.cern.ch/centos7-umd4-ui-4_200423/etc/profile.d/setup-c7-ui-example.sh or something else depending on your site
    source /cvmfs/cms.cern.ch/common/crab-setup.sh prod
    source /cvmfs/cms.cern.ch/cmsset_default.sh
    ```

    In your regular shell, you can then do the following whenever you want to work with PFNano:
    ```shell
    source ~/.cmsrc
    ```

    Now the site should be configured similar to lxplus. Next, we checkout a recent CMSSW version:

    ```shell
    cmsrel CMSSW_13_0_6
    cd CMSSW_13_0_6/src
    cmsenv
    scram b -j 8
    ```

    !!! note
        PFNano also works with older versions, but with recent ones like 130x / 131x we get new taggers and inputs related to updated jet collections and tunes. Of course the output will still depend on what you give the tool as input - i.e., the MiniAOD version. It may be necessary to re-run producers on top of existing MiniAOD from scratch, if the files do not contain the latest developments yet supported by 130x / 131x (think 2022 data, mc). For 2023 PromptReco, this should not be necessary, as 130x is anyway used for that chain.

    Now get PFNano as a package in your CMSSW environment:

    ```shell
    git clone https://github.com/AnnikaStein/PFNano-1.git PhysicsTools/PFNano #change once new developments are officially included: git clone https://github.com/cms-jet/PFNano.git PhysicsTools/PFNano for now just take my development branch and follow along
    cd PhysicsTools/PFNano
    git fetch
    git switch 130x # or another branch, as advised to you
    cd ../..
    scram b -j 8
    ```

    To setup testing, move to test directory and activate proxy:

    ```shell
    cd PhysicsTools/PFNano/test
    voms-proxy-init --voms cms:/cms/dcms --valid 192:00 # cms:/cms/dcms only if you have a German grid certificate, get prio to run at German sites; use cms only otherwise
    ```

    ### Using as is

    With the configurations provided in the test directory, you can already test the functionality and compatability with the current version. Just run

    ```shell
    cmsRun nano_mc_Run3_EE_NANO.py
    ```

    as an example, or one of the other configurations.

    Note: such configurations are created via cmsDriver:

    ```shell
    cmsDriver.py nano_mc_Run3_EE --mc --eventcontent NANOAODSIM --datatier NANOAODSIM \
    --step NANO --conditions 126X_mcRun3_2022_realistic_postEE_v1 --era Run3,run3_nanoAOD_124 \
    --customise_commands="process.add_(cms.Service('InitRootHandlers', EnableIMT = cms.untracked.bool(False)));process.MessageLogger.cerr.FwkReport.reportEvery=1000;process.NANOAODSIMoutput.fakeNameForCrab = cms.untracked.bool(True);" \
    --nThreads 4 -n -1 \
    --filein /store/mc/Run3Summer22EEMiniAODv3/QCD_PT-80to120_MuEnrichedPt5_TuneCP5_13p6TeV_pythia8/MINIAODSIM/124X_mcRun3_2022_realistic_postEE_v1-v1/2550000/eddaff63-eb30-4155-afdc-3db5b07105b8.root \
    --fileout file:nano_mcRun3_EE.root \
    --customise="PhysicsTools/PFNano/pfnano_cff.PFnano_customizeMC_add_DeepJet_and_Truth" --no_exec

    cmsDriver.py nano_mc_Run3 --mc --eventcontent NANOAODSIM --datatier NANOAODSIM \
    --step NANO --conditions 126X_mcRun3_2022_realistic_v2 --era Run3,run3_nanoAOD_124 \
    --customise_commands="process.add_(cms.Service('InitRootHandlers', EnableIMT = cms.untracked.bool(False)));process.MessageLogger.cerr.FwkReport.reportEvery=1000;process.NANOAODSIMoutput.fakeNameForCrab = cms.untracked.bool(True);" \
    --nThreads 4 -n -1 \
    --filein /store/mc/Run3Summer22MiniAODv3/QCD_PT-15to20_MuEnrichedPt5_TuneCP5_13p6TeV_pythia8/MINIAODSIM/124X_mcRun3_2022_realistic_v12-v1/30000/8590bc1e-abd3-4be4-a068-16f4cb6b4994.root \
    --fileout file:nano_mcRun3.root \
    --customise="PhysicsTools/PFNano/pfnano_cff.PFnano_customizeMC_add_DeepJet_and_Truth" --no_exec

    cmsDriver.py nano_data_2022ABCD --data --eventcontent NANOAOD --datatier NANOAOD \
    --step NANO --conditions 124X_dataRun3_v11 --era Run3,run3_nanoAOD_124 \
    --customise_commands="process.add_(cms.Service('InitRootHandlers', EnableIMT = cms.untracked.bool(False)));process.MessageLogger.cerr.FwkReport.reportEvery=1000;process.NANOAODoutput.fakeNameForCrab = cms.untracked.bool(True);" \
    --nThreads 4 -n -1 \
    --filein /store/data/Run2022C/DoubleMuon/MINIAOD/10Dec2022-v1/2820000/dea1757f-d2ef-467a-9062-714775d00e45.root \
    --fileout file:nano_data2022ABCD.root \
    --customise="PhysicsTools/PFNano/pfnano_cff.PFnano_customizeData_add_DeepJet" --no_exec
    ```

    !!! note
        In any case, make sure to run data configurations with data, and mc with simulation. That way, only the existing Products are accessed.

    Consult the recommendations of [PdmV](https://twiki.cern.ch/twiki/bin/viewauth/CMS/PdmV) for `conditions`, [XPOG](https://gitlab.cern.ch/cms-nanoAOD/nanoaod-doc/-/wikis/home)/[WorkBookNanoAOD](https://twiki.cern.ch/twiki/bin/view/CMSPublic/WorkBookNanoAOD) for `era`.

    #### When just testing locally

    - It’s not necessary to wait until the full file has been processed, with MC and truth information, that can take a while and is not necessary to check if everything works.
    - Just wait until you see the first lines showing up that reads something like „Begin processing the 501st record. Run 1, Event 3675502, LumiSection 3676 on stream 0 at 15-Dec-2022 00:22:22.266 CET”
    - If you are sure a couple hundred events run without crashes (warnings are OK! Especially at the beginning and then later related to JetFlavourClustering for AK8...), just do `Ctrl+c`, the file will be saved anyway with whatever has been processed until then. You can open the file with e.g. uproot and investigate what’s inside.
    - Alternatively, you can also ask `cmsRun` to only run over a certain number of events by modifying the process as follows:
    ```python
    process.maxEvents = cms.untracked.PSet(
        input = cms.untracked.int32(500), # or any number you deem sufficient
        output = cms.optional.untracked.allowed(cms.int32,cms.PSet)
    )
    ```


=== "Moving to CRAB"

    ## Submit entire datasets
    Submission is controlled via .yml cards, one per campaign of data/mc, year, config and output path. The handler (`crabby.py`) accesses a template (`template_crab.py`) and fills it with the relevant parameters. These need to be modified by you, according to your task. Make a copy of some `card_example_*.yml` that is close to what you want to achieve. Fill in the relevant entries:
    ```YAML
    workArea # new for each submission
    storageSite # you need access
    outLFNDirBase # match with your username, define some useful path
    voGroup # leave empty or put dcms for Germans
    tag_extension # ask BTV what to use here
    publication # leave this at True, please
    config # should match with the purpose of .yml card and workArea for bookkeeping purposes and to avoid mistakes
    data # should match with the above True or False
    lumiMask # leave empty for MC (unless doing a Recovery task), or put path for data-lumimask that fits with config and dataset
    datasets # DAS path - please: do not put a set of paths as .txt to not overload CRAB, though convenient for you, it's just going to slow down things for everyone or cause whole system to degrade
    ```
    No need to touch the other lines.

    ### Commands for submission

    Start with

    ```shell
    python crabby.py -c your_card.yml --make --test True
    ```
    Check the created work directory and make sure all entries make sense. Then actually submit this config.

    ```shell
    python crabby.py -c your_card.yml --submit
    ```

    ##### Here are some additional useful comments

    - You can always submit a test first to crab (`--test True`) and make sure everything runs as expected. Should this fail, there might be things to tune from your side, but it means you don’t waste thousands of jobs just to find this out. :-)    
    - When you are in PFNano’s test dir, use a command similar to this one to get more information on the job status: `[anstein@naf-cms16 test]$ crab status -d DataDoubleMuon17B_ULv2_with_ParT_yml/crab_DoubleMuon_Run2017B-UL2017_MiniAODv2-v1_MINIAOD`. Tip: `crab report` is more detailed and also writes useful .json files which can later be used in case of having to perform a recovery task, or when calculating the lumis you ran over.
    - Note down the url which links to monit-grafana.
    - It can take a while (~0.5-1h) until your task shows up in monit-grafana, also `crab status` may lag a bit behind.
    - Check if the jobs run through without errors.

    #### If tests were successful

    - If not done so yet: set `publication` to `True` so that everyone can use the samples (activated by default, just don’t change it). That way, we can access your files from DAS, and everyone can read them.

    Create a new submission config (change the `workArea` in the .yml to have a distinct new one).

    ```shell
    python crabby.py -c your_card.yml --make
    ```

    Check the created work directory and make sure all entries make sense. Then actually submit this config.

    ```shell
    python crabby.py -c your_card.yml --submit
    ```

    #### More information on monitoring, bookkeeping and performing the entire production

    - If you don’t trust the status there or on monit-grafana because they are slightly different, you can also check the number of processed and already stored files similar to this: `xrdfs grid-cms-xrootd.physik.rwth-aachen.de ls /store/user/anstein/PFNano/DoubleMuon/Run2017B-UL2017_MiniAODv2-v1_PFNanoFromMiniV2WithParT/221214_191826/0000 | wc -l` independent of crab commands (this also counts one log file in the directory, so subtract 1 from the number, repeat for every available four-digit serial number, `0000`, `0001` ... if they exist)
    - Don’t submit the next dataset before the previous one isn’t finished, similar, don’t submit an entire stack of datasets via a `.txt` file. It’s just a recommendation, but backed with the opinion of crab experts, now that a lot of resources are used for Run 3. It might look convenient for you at first, but will slow things down for everyone. Be a good person, try not to end up in [cms-talk...](https://cms-talk.web.cern.ch/t/crab-jobs-for-private-mc-production-causing-issues/24021) after submitting too much at once!
    - Once everything is finished, please point to the resulting DAS entry [(similar to this one)](https://cmsweb.cern.ch/das/request?input=%2FDoubleMuon%2Fanstein-Run2017B-UL2017_MiniAODv2-v1_PFNanoFromMiniV2WithParT-1279add48bc42ba03d94c7bdc1821b2e%2FUSER&instance=prod%2Fphys03)
    - Failed jobs can happen (they will!) - we just need to understand why, how frequent, at which site, with which code. Then we can take action, change something, resubmit.

    ### Quick summary
    Learn how to run locally, adjust a bit the configuration to use your own username for the file paths and modify the dataset, submit via crab, then be patient and trust the process. :)

    [All samples of this campaign at DAS](tba)


    ## Typical CRAB issues and how to deal with them

    ### Trying to submit
    ??? question "How to mitigate SUBMITFAILED if more than 100000 lumi entries per block?"

        Modify this in the submit file:

        ```python
        config.Data.splitting = 'FileBased'
        config.Data.unitsPerJob = 1
        ```

        (will process every file individually and does not need lookup of lumis)

    ### Temporary problems after submission, most likely related to sites, Locality
    ??? question "[DO RESUBMIT] CRAB retry is not enough, can I run the failed jobs again, maybe it works now?"

        Run `crab resubmit` for the task in question (same syntax as for `crab status`, just replace `status` with `resubmit`)

    ??? question "[DO RECOVERY] What if the jobs only try to run on one site, leading to long waiting times? What if they always fail at the same site?"

        Take note of this behaviour, if it persists, collect the lumisections you did not yet run over via `crab report`, put that `/afs/or/whatever/path/to/task/workdir/somecrypticstuff/notFinishedLumis.json` as lumimask, issue `crab kill` for the original task, run `crabby.py ... --make` over the modified yaml. Modify this in the submit file:

        ```python
        config.Data.ignoreLocality = True
        config.Site.whitelist = ['T2_US_*']
        # not strictly necessary, but if you know that there is a site which is really bad, there's maintenance, network problems or whatever:
        # config.Site.blacklist = ['T2_XY_WXYZ']
        ```

        and then submit this new task (=recovery task).
        (will start pretty much immediately, have a look at DAS where the dataset is located first, then find out T2s close to the dataset, access will be via xrootd - the `T2_US_*` is just an example, specify whatever you deem useful, combination of multiple site is possible in that list, and as you see wildcards work as well)

    ## Known errors that occured during Run3 processing

    ??? question "[DO RECOVERY] I got an `InvalidReference` error, what to do, it affects a significant amount of jobs?"

        This is a known problem related to the NanoAOD-step, when the fatJetTable is created. It has nothing to do with our own code, and for an event where this occurs, we can just skip the entire event and continue business as usual with the remainder of the root file. Just modify the one thing in the test configuration (`test/whatever_NANO.py`):

        ```python
        SkipEvent = cms.untracked.vstring('InvalidReference')
        ```

        and create a recovery task as introduced above (report -> missing lumis as custom lumimask -> kill original task -> submit new task using the slightly modified config). That way, we don't lose entire files, but only $\mathcal{O}(1)$ events per file.

    ## For data

    Please calculate the luminosity via [brilcalc](https://github.com/cms-jet/PFNano/tree/12_4_8#running-brilcalc), use these [recommendations](https://twiki.cern.ch/twiki/bin/viewauth/CMS/LumiRecommendationsRun3).

=== "Developing"

    ## PFNano and NanoAOD

    ??? question "How to make sure I don't introduce duplicate branches?"

        Open your newly created `PFNano.root` file, e.g. with uproot / awkward. Run this small snippet and make sure the output is empty:

        ```python
        import uproot
        f = uproot.open('PFNano.root') # (change file name to yours of course)
        tree = f['Events']
        keys = tree.keys(['Jet*']) # (or other collections you want to cross-check)

        def duplicates(mykeys):
            seen = set()
            for x in mykeys:
                if x in seen:
                    print(x)
                seen.add(x)
            return

        duplicates(keys)
        ```

    ??? question "How can I add my custom variables?"

        It depends. Either modify `python/addBTV.py` and the corresponding customise commands of the `pfnano_cff.py`, or rewrite / add a C++ producer related to specific collections in CMSSW, e.g. the TagInfoCollection for a tagger you are interested in. Make sure to compile and get rid of all errors! Then submit a PR to the PFNano repo on GitHub, such that someone can review the addition.

    ## Documentation

    ??? question "How and where to document my new branches?"

        Use the tools provided with NanoAOD (you might need to do `git cms-addpkg PhysicsTools/NanoAOD` inside your `/src` directory to see and understand how the tools work and what options you have), get websites for content and file sizes like so:
        ```shell
        python PhysicsTools/NanoAOD/test/inspectNanoFile.py NANOAOD.root -s website_with_collectionsize.html -d website_with_collectiondescription.html
        ```

        Upload to your web.cern.ch page.
