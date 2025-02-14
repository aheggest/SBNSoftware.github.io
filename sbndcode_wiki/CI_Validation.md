---
layout: page
title: CI Validation
---

# CI Validation

This page describes the CI validation based upon the *sbnci* package and utilising the *lar_ci* package.   

To get back to the main CI wiki page, click [here](Continuous_integration.html).


## Using the validation

### Available Workflows
- There is currently just one available validation workflow for SBND. It analyses the reconstruction outputs. If you're interested in developing new validation workflows then read the developers section below and feel free to get in contact with the CI & Validation working group via Chris Hilgenburg (chilgenb@umn.edu).
- The reconstruction validation utilises a set of input detsim files with the SAM definition `sbnd_ci_bnb_nue_detsim_v09_26_00`. This definition consists of 5k BNB neutrino events and 5k intrinsic nue events. This dataset needs periodic updates to reflect changes in the gen, g4 and detsim stages.
- You can run the validation with any combination of feature branches from the following repositories: `sbndcode`, `sbndutil`, `sbnci` (and `larpandoracontent` if you use `CI_VALIDATION_SBND_pandora` as your workflow). If you require use of feature branch from any other repository contact Henry Lay (h.lay@lancaster.ac.uk) to have a custom repository added. Longer term there are plans to allow command line requests of any of the required products.
- Before running any CI jobs you need to setup `lar_ci` and provide yourself with the correct proxies:

   `setup lar_ci`
 
   `setup cigetcert`
 
   `cigetcert -s 'fifebatch.fnal.gov'`
 
   `voms-proxy-init -noregen -rfc -voms 'fermilab:/fermilab/sbnd/Role=Analysis'`
 
   ....or equivalent!
   
- I would recommend a test submission first, this will allow you to test that your combination of feature branches work together and don't cause an easy-to-fix failure for all 10k events on the grid. Note that the validation plots will be meangingless, this is a test that the machinery works. This submission can be followed on the [test dashboard.](https://dbweb9.fnal.gov:8443/TestCI/app/ns:SBND/view_builds/index)
- The test trigger uses the following command:

**IMPROTANT NOTE** As a temporary addition this workflow requires the extra tag `--version feature/hlay_sbnd_pfp_validation` for all trigger commands
   
   `trigger --build-delay 0 --jobname sbnd_ci --workflow CI_VALIDATION_SBND_test --gridwf-cfg cfg/grid_workflow_sbnd_mc_reco_all_test.cfg --revision "repo@branch" --testmode`
 
- The key point here is the `--revisions` tag which is where you specify the branches you want to test. For example: `--revisions "SBNSoftware/sbndcode@feature/hlay_vertexing henrylay97/LArContent>larpandoracontent@feature/hlay_vertex_bdt_changes"` would provide the two custom branches for `sbndcode` and `larpandoracontent` respectively. Note the different syntaxes for specifying a branch in the main repository and for using a branch in a forked repository and directing it towards the correct product name.
- Once your test job has succeeded (or you're feeling brave) then the full reconstruction validation can be triggered with the following command:
   
   `trigger --build-delay 0 --jobname sbnd_ci --workflow CI_VALIDATION_SBND --gridwf-cfg cfg/grid_workflow_sbnd_mc_reco_all.cfg --revision "repo@branch"`

- Other options for the trigger command can be found as usual by running `trigger -h`.
- The results of your trigger can be followed on the [dashboard.](https://dbweb8.fnal.gov:8443/LarCI/app/ns:sbnd/view_builds/index)

## Developing the validation

### The *sbnci* Package
- The *sbnci* package can be found [here](https://github.com/SBNSoftware/sbnci).
- It is a UPS package that currently depends on sbndcode, but is expected to depend on icaruscode in the future.
- The package is written to run validation scripts on the output of the simulation chain, and to compare these validation plots with specified reference plots.
- There are 4 main directories found in `sbnci/sbnci` for running this ci validation: *Modules*, *PlottingScripts*, *scripts* and *thresholds*.
- The main recipe for this package is to create ROOT TTrees from an analysis module, create plots of these variables, and then compare them to reference plots.

#### *Modules*
The *Modules* directory contains LArSoft analysis modules for making ROOT TTrees from the simulations, e.g. it currently contains the Shower Validation Module. 
These modules should probably move to sbndcode, icaruscode or sbncode at some point down the road. 

#### *PlottingScripts*
The *PlottingScripts* directory contains the Contains ROOT macro (.C) files reading in the `.root` files from the module outputs and making a `.root` file 
containing histograms and separate `.png` files for each plot to be displayed online. This directory also contains the comparison script 
`CompareDataDistributions.C` which has been ported over having been written by Vito Di Benedetto for the DUNE CI validation. 

#### *scripts*
The *scripts* directory contains the `.sh` scripts used to run the plotting macros in *PlottingScripts*, setup the appropriate reference files 
and run the comparison script.

#### *thresholds*
The *thresholds* directory contains the `.txt` files with chi-squared thresholds for the plot comparisons in each validation. Each plot is given two thresholds, this defines "good", "okay" and "bad" agreement. These scales are also used to colour code the chi-squared tables and the plots on the CI dashboard.


### The *lar_ci* Package
- The *lar_ci* package is written by Vito Di Benedetto for running all of the LAr experiment CI's at Fermilab, 
it can be found [here](https://cdcvs.fnal.gov/redmine/projects/lar_ci/).
- The main part of *lar_ci* that we need to be aware of for the CI validation is `lar_ci/cfg`, this directory holds all of the config files that tell the CI 
which files to run and which settings to use.


### Updating the CI validation.

- Updating the CI validation to include a new set of validation plots requires updating both *sbnci* and *lar_ci*. 

#### Updates to *sbnci*

- The first update will be to include the LArSoft analysis modules inside `sbnci/sbnci/Modules`. As *sbnci* depends on *sbndcode* (and *icaruscode* in the future)
regular LArSoft analysis modules can be used and run here.
- You will need to provide a prolog-type fcl in the same folder as your new module to define a relevant table of fcl parameters. You should also provide a job fcl in the *JobConfigurations* folder which can be used to run your module.
- Next a new ROOT macro will be needed in `sbnci/sbnci/PlottingScripts`. The file *sbnciplot_showervalidation.C* can be used as an example, the structure of the 
file should match with the ROOT plots saved inside gDirectories. The output file should always have the name `ci_validation_histos.root` for the comparison 
script to work.  The validation output files from different CI modules are all kept in separate directories so they won’t get confused.
- Lastly a new bash script should be created inside the `sbnci/sbnci/scripts` to drive the validation steps. The script `showervalidation.sh` can be used as
an example for this. It should provide `sbnciplots.sh` with the desired Root macro from `sbnci/sbnci/PlottingScripts` and the input file which is passed into `showervalidation.sh` as the first argument, this is set in the `lar_ci` config file.

#### Updates to *lar_ci*

- The change required in the *lar_ci* package is to create a new config file for the new validation workflow, this should be located inside `lar_ci/cfg`. 
A good example to use here is `grid_workflow_sbnd_mc_reco_all.cfg`, note this workflow also details how to use multiple analysis modules in the same workflow. 
- There are a lot of parameters to set in the config file
   - *validation_process* and *validation_tag* provide a name for your validation workflow
   - *validation_function* provides the name of the validation script (from *sbnci/scripts*) followed (just a comma inbetween, no gaps) by the name of the input file (this needs to match the output of your *[mergeana]* stage later on, i.e. the tree file
   - You can have multiple *validation_functionX* parameters (*X=1,2,3...*), you will need a name for each in front of the script name deliminated by a *#* symbol
   - The *ci_extra_vars_X* parameters allow you to define the enviroment variables needed later. This includes the reference version, the location of reference histogram file(s) and the location of the threshold file.
   - The desired simulation chain can be set in the *[sim]* and *[reco]* sections. The *[reco]* section should also include the fcl to be used to run the analysis module (e.g. `allrecovalidation_job_sbnd.fcl`). The files do not have to be simulated from scratch, a SAM definition can be used with the *input_defname* tag.
   - Make sure (particularly with multiple validations) that you get the formatting of *output_to_transfer* in the final stage correct to get all of the histogram files correctly transferred.

### Running the CI Validation

Before triggering any CI tests you need to get the right certificates and the lar_ci commands. To get these use these commands:

`setup lar_ci`
 
`setup cigetcert`
 
`cigetcert -s 'fifebatch.fnal.gov'`
 
`voms-proxy-init -noregen -rfc -voms 'fermilab:/fermilab/sbnd/Role=Analysis'`
 
 ....or equivalent!
  
The CI validation can then be triggered using the following command (this is for the combined reconstruction workflow):

`trigger --build-delay 0 --workflow CI_VALIDATION_SBND --gridwf-cfg cfg/grid_workflow_sbnd_mc_all_reco.cfg --jobname sbnd_ci`

To run tests using a feature branch of *lar_ci* (e.g. `feature/hlay_sbnd_pfp_validation` here) use:

`trigger --build-delay 0 --workflow CI_VALIDATION_SBND --gridwf-cfg cfg/grid_workflow_sbnd_mc_all_reco.cfg --version feature/hlay_sbnd_pfp_validation --jobname sbnd_ci`

And to include any other feature branches or specific versions us the `--revisions` option, e.g.:

`trigger --build-delay 0 --workflow CI_VALIDATION_SBND --gridwf-cfg cfg/grid_workflow_sbnd_mc_all_reco.cfg --version feature/hlay_sbnd_pfp_validation --jobname sbnd_ci --revisions "SBNSoftware/sbncode@v09_10_01"`

For more info check out the [lar_ci wiki](https://cdcvs.fnal.gov/redmine/projects/lar_ci/wiki).
For help with importing a new module/workflow feel free to contact me (`h.lay@lancaster.ac.uk`) and I'll do my best to answer any questions!
