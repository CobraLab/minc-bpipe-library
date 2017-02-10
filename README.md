# Library of bpipe functions for processing minc files
====================================================

`minc-bpipe-library` provides a set of chainable minc file processing functions to (pre)process data. At the moment is our star preprocessing pipeline. By default it will perform: N4correction, Cutneck, Head and Brain masks (using BEAST) and registration to MNI (using ANTs). 

To run in any computer it requires [http://www.bic.mni.mcgill.ca/ServicesSoftware/ServicesSoftwareMincToolKit](http://www.bic.mni.mcgill.ca/ServicesSoftware/ServicesSoftwareMincToolKit), [https://github.com/ssadedin/bpipe/](https://github.com/ssadedin/bpipe/) and gnu-parallel.

To run it on Scinet see below.

To control which stages are run, edit ``pipeline.bpipe`` and add stage names using "+" to the "run" stage.

The default in ``pipeline.bpipe`` is what has experimentally determined to be a best-practice run.

Stages are listed in the ``minc-library.bpipe``, correction stages such as denoising, n3 and n4, and normalize
should be run before any other stages. Typically after this ``linear_antsRegistration`` would be run, which will
allow other processing such as VBM, cutneck, deface and beast to be done in MNI space.

For convenience, "segments" have been defined for some processing jobs (beast, cutneck, VBM) which are multi-step.
These may conflict with each other if you try to combine them, so instead you must specifiy their individual stages.

Stage options have been chosen based on best pratices from publications where applicable but can be changed.

Once you have defined your stages, you can run your pipeline on the SGE cluster with:
```
> git clone https://github.com/CobraLab/minc-bpipe-library.git
#Edit minc-bpipe-library/pipeline.bpipe as you see fit
> module load bpipe #needed to run bpipe
> module load minc-toolkit minc-toolkit-extras #needed for most stages
> cd /path/to/store/outputs
#Choose n to be the smaller of (4, 240)
> bpipe run -n<number> /path/to/pipeline.bpipe /path/to/inputs/*mnc
```

Output filenames from bpipe will be of the form ``<inputname>.stage1.stage2.stage3.ext`` where the last stage
name will be the last run. ``commandlog.txt`` will be generated by bpipe describing all the commands run for
the pipeline. This is a very good file to keep around as as a note of what happened to your data.

Some stages produce both ``mnc`` files and ``xfm`` files, see the ``$output`` variables for what is generated.

#Note on the space of output files
``minc-bpipe-library`` produces outputs in two spaces, the native space of the input image, and a LSQ12 registered
space of a target image, typically the MNI ICBM 09c template. Be aware of the difference between these spaces, most
pipelines (CIVET, MAGeT, etc) expect their input files to be in native space. Files with a ``register`` component
in their name indicate files which have been transformed into the template space, files without are in their native space.

The use of the ``clean_and_center`` stage does not change the space of the file through registration, but does uniformize
the direction cosines and the zero-point of the scan to the center of the image. Such modifications do impact orientation
and location but in a reversable way with no loss of information. If your dataset contains multiple modalities (T1, T2, fMRI)
you should not use ``clean_and_center`` as it will remove the intrinsic co-registration your scans have as a result of scanning
within a single session.

#Scinet Operation
Inputs are split into 8 file chunks and submitted as local bpipe jobs on scinet nodes

Steps

1. ``git clone https://github.com/CobraLab/minc-bpipe-library.git``
2. ``rm minc-bpipe-library/bpipe.config``
3. ``sed -i 's#/opt/quarantine#/project/m/mchakrav/quarantine#g' minc-bpipe-library/minc-library.bpipe``
4. ``mkdir bpipe-outputs && cd bpipe-outputs``
5. ``module load scinet-2017 qbatch/git``
6. Use ``../minc-bpipe-library/bpipe-batch.sh ../minc-bpipe-library/pipeline.bpipe /path/to/my/inputs/*.mnc > joblist`` to generate a joblist
7. Use ``qbatch -N myjobname --walltime=12:00:00 joblist`` to submit jobs to scinet queing system

## Example:

1. Create a folder for preprocessing (i.e. `mkdir preproc`) and cd into it `cd preproc`.

2. Point the `bpipe-batch.sh` to where `pipeline.bpipe` is installed and then to where the T1s you are using are.

`bpipe-batch.sh /home/m/mchakrav/egarza/bin/minc-bpipe-library/pipeline.bpipe /home/m/mchakrav/egarza/scratch/adhd_gen/preproc/files/*.mnc > joblist`

3. Check the joblist file using `nano`

4. Submit the jobs:

`qbatch joblist 1 12:00:00`

5. QC the resulting files. For MAGeT or CIVET you want to use the `n4corrected.cutneckapplyautocrop.mnc` in native space. You can also use the `.beastmask.mnc` files for CIVET.

## QC Generation
For pipelines at the CIC, QC images are automatically generated.

The script ``generate-bpipe-QC.sh`` is used to generate standardized views of the final outputs for quality control on SciNet:

```
#In your output directory
> module load scinet-dev
> mkdir QC
> for file in *linear_bestlinreg.cutneckapplyautocrop.beastnormalize.mnc; do ../minc-bpipe-library/generate-bpipe-QC.sh $file QC/$(basename $file .mnc).jpg; done
```
### QC example:

`for file in *linear_bestlinreg.cutneckapplyautocrop.beastnormalize.mnc; do ~/bin/minc-bpipe-library/generate-bpipe-QC.sh $file QC/$(basename $file .mnc).jpg;done`

