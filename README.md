# Project PEDIA
Prioritization of Exome Data by Image Analysis (PEDIA) is to investigate the value of computer-assisted analysis of medical images and clinical features in the diagnostic workup of patients with rare genetic disorders. We integrate the facial gestalt analysis from DeepGestalt approach (https://www.nature.com/articles/s41591-018-0279-0) in Face2Gene with the other phenotypic analysis tools and deleteriousness scores from the molecular level to prioritize potential disease genes.

This tool is already integrated in Face2Gen LAB and https://pedia-study.org. For the user who would like to utilize it to analyze your patients, please contact Prof. Peter Krawitz (pkrawitz@uni-bonn.de) in Institute for Genomic Statistics and Bioinformatics for more details.

## General Information

The whole workflow of the PEDIA project uses [snakemake](https://snakemake.readthedocs.io/) to run a pipeline together with [conda/bioconda](https://bioconda.github.io/) to install the necessary programs. So pelase get familiar with both BEFORE starting the workflow. A good start is the [snakemake tutorial](https://snakemake.readthedocs.io/en/stable/tutorial/tutorial.html). 

### Requirment
   * python version >= 3.6
   * miniconda
   * snakemake
   * storage >= 30 GB (reference genomoe and external database such as ExAC and CADD)
   * **Key and secrect of your Face2Gene LAB (You are not able to use PEDIA without key and secret)**


### Install miniconda

Go have a look at the [miniconda website](https://conda.io/miniconda.html). Be sure that you choose the right version depending on your python version. 

### Install necessary software via conda
Now we will generate a shell environment with all necessary programs for downloading and process files. The software needed for the downloading is in the `environment.yaml` file. Conda can read it:

```
conda env create -f environment.yaml
```

Now we created an enviroment called `pedia`. We can activate it and we should have snakemake installed.

```
source activate pedia
snakemake -h
```
We can deactivate the environment using `source deactivate`. The command `conda env list` will list you all environments.

Now lets run the pedia download workflow. We can make a "dry run" using `snakemake -n` to see what snakemake would probably do.
```
cd data
snakemake -p -n all
```

### Setup classifier submodule
classifier is the submodule, so we have to clone it by the following command.
```
git submodule update --recursive
```

## Usage instructions
### Configuration

Most configuration options are in a `config.ini` file, with options commented.
A `config.ini.SAMPLE` in the project directory can be used as reference for
creating an own configuration.

To fetch the patient data from your Face2Gene LAB, please put the secret and key in the following setting in config.ini
```
[your_lab_name]
; not necessary to be the same name in Face2Gene LAB
lab_id = your lab id in Face2Gene LAB
key = your key in Face2Gene LAB 
secret = your secrect in Face2Gene LAB
download_path = process/lab/bonn (the folder you would like to save the downloaded JSON files)
```

### HGVS Error dict (optional)

HGVS variant overrides are specified in `hgvs_errors.json`. Which is per-default
searched for in the project root.

The hgvs version is specified in `lib/constants.py` and will cause an error if
an hgvs errors file of not at least the specified version is found.

The number can be lowered manually to accept older hgvs error files.

A version of 0 will accept no hgvs_errors file.

### Required external files
   * Go to data folder, and run 'snakemake all' to download all necessary files such as reference genome, population data.
   * Copy corrected JSON files to process/correct/ (optional)
   * Copy hgvs_errors.json to project folder (optional)

### Description of the subfolders

* **3_simulation** - 
	Scripts and pipelie about generation background populations and simulationg the data.
	In addition the spike in is made and molecular pathogenicity scores are added to the jsons.
* **data** - 
	It contains the files from external database (dbSNP, reference genome, CADD, ExAC). Besides, data/PEDIA/json/phenomized contains the JSON files after QC and phenomization. 
* **classifier** - 
 It is a submodule of PEDIA-workflow. The repository is here. (https://github.com/PEDIA-Charite/classifier). 
 * rest of the files and folders belong to preprocessing such as lib, helper, test
	All About the quality check and phenomization
	
## Running PEDIA
### Environment setup
   * Go to data folder, and run 'snakemake all' to download all necessary files such as reference genome, population data.
   ```
   cd data
   source activate pedia
   snakemake all
   ```
   * You could download the training data we used in PEDIA paper in the following link (https://pedia-study.org/pedia_services/download)
   * Copy jsons folder to 3_simulation. 3_simulation/jsons/1KG/CV_gestalt/* .json will be used for training data.

### Preprocessing
Since some steps depend on the existence of API keys, running the preprocess.py
script without a configuration file will **not work**.

The **preprocess.py** script contains most information necessary for running a
conversion of json files from your Face2Gene LAB to PEDIA format.

If you add ```-v your_vcf_file```, it will automatically trigger the whole workflow.

```
# do not forget to activate the previously created virtual environment

# get a list of usable options
./preprocess.py -h

# run complete process on all the cases in your lab
./preprocess.py -l lab_name_in_config.ini

# run complete process on a single case in your lab
./preprocess.py -l lab_name_in_config.ini --lab-case-id the_lab_case_id_of_your_case

# run for a single file (specifying output folder is beneficial)
./preprocess.py -s PATH_TO_FILE -o OUTPUT_FOLDER

# run for a single file on whole PEDIA workflow with -v and specify the VCF file
./preprocess.py -s PATH_TO_FILE -o OUTPUT_FOLDER -v your_vcf_file
./preprocess.py -l lab_name_in_config.ini --lab-case-id the_lab_case_id_of_your_case -v your_vcf_file
```
* **Example:**
You could use the example in tests/data/123.json and tests/data/123.vcf.gz
```
python3 preprocess.py -s tests/data/123.json -v tests/data/123.vcf.gz
```

### Run PEDIA pipeline
There are three steps to run pipeline.
1. Download cases and perform preprocessing

   ```
   python3 preprocess.py -l lab_name
   ```
   * config.yml contains the cases passed quality check. SIMPLE_SAMPLES is the case with disease-causing mutation but without real VCF file. VCF_SAMPLES is the case with real VCF file. TEST_SAMPLE is the case with real VCF but without disease-causing mutation.
   * process/lab/lab_name is the folder of cases downloaded from LAB (in config.ini).
   * data/PEDIA/jsons/phenomized is the folder which contains the JSON files passed QC.
   * data/PEDIA/mutations/case_id.vcf.gz  is the VCF file which contains disease-causing mutations of all cases.
   * data/PEDIA/vcfs/original is the folder which contains the VCF files. In mapping.py, we rename the filename of VCF files to case_id.vcf.gz and store to ../data/PEDIA/vcfs/original/. The new filename is added in vcf field of the JSON file. For example,
   ```
   "vcf": [
           "28827.vcf.gz"
       ],
   ```

1. Get JSON files of simulated cases and real cases

    To obtain the CADD scores of variants, we need to annotate the VCF files and retrieve the CADD score and append it to the geneList in JSON file. Now, we go to 3_simulation folder and activate simulation environment. 
   
    Note: you could skip this step by running the experiemnt in classifier. The classifier will trigger this subworkflow to generate JSON files.
   
    Before we start, we would like to explain the two experiments we want to conduct in this study. First one is that we want to perform cross-validation on all cases to evaluate the performance among three simulation samples (1KG, ExAC and IRAN). The second one is that we want to train the model with simulated cases and test on the real cases. To achieve these two goals, we have the following command to perform simulation and generate the final JSON files.


    * To perform the CV experiment, we run the following command to obtain the JSON files simulated from 1KG, ExAC and IRAN data. You could replace 1KG with ExAC and IRAN
    ```
    snakemake performanceEvaluation/data/CV/1KG.csv
    ```
   
    * To peform the second experiemnt, we run the following command to obtain the training and testing data sets. Generate the JSON files of **real cases** the output will be in 3_simulation/json_simulation/real/test
    ```
    snakemake createCsvRealTest
    ```
   
    * Generate the JSON files of **simulated cases** the output will be in 3_simulation/json_simulation/real/train/1KG. You could replace 1KG with ExAC and IRAN
    ```
    snakemake performanceEvaluation/data/Real/train_1KG.csv
    ```
    
    * The final JSON files are in 3_simulation/json_simulation folder.
        * 3_simulation/jsons/1KG is the folder for all cases simulated by 1KG.
        * 3_simulation/jsons/ExAC is the folder for all cases simulated by ExAC.
        * 3_simulation/jsons/real/train is the folder for the cases without simulated by 1KG, ExAC or IRAN. We also have three folder under this folder.
        * 3_simulation/jsons/real/test is the folder for the cases with real VCF file.
 
1. Go to classifier folder to classify the patient. Train with all cases and test on patient with **unknown diagnosis**. You will find the results in output/test/1KG/case_id/. Please find the more detail in
(https://github.com/PEDIA-Charite/classifier).
   ```
   snakemake output/test/1KG/21147/run.out
   ```
   
1. Cross-validation evaluation
   * Go to classifier folder.  Run 'source activate classifier' to enable the environment. If you haven't created the environment, please execute 'conda env create -f environment.ymal'.
   * Perform 10 times 10 fold cross-validation on 1KG simulation data, please execute this command 'snakemake ../output/cv/CV_1KG/run.log'. Please remind the working directory of classifier is in scripts folder, so to run on 1KG simulation you need to specify the output file 'output/cv/CV_1KG/run.log '.

   ```
   snakemake output/cv/CV_1KG/run.log
   ```
   
1. Train and test evaluation
   * Training set is in 3_simulation/json_simulation/real/train/1KG (1KG, ExAC and IRAN)
   * Testing set is in 3_simulation/json_simulation/real/test
   ```
   snakemake output/real_test/1KG/run.log
   ```

1. How to read the PEDIA results?
   * case_id.csv contains all genes with corresponding pedia scores in this case
   * case_id_pedia.json contains all scores and PEDIA scores
   * rank.csv contains the predicted rank of disease-causing gene of each case.

