# NCPI-Whistler-Tutorial
This tutorial is intended to provide a step-by-step for creating a project to transform research data into FHIR resources and (optionally) load those resources into a FHIR server. 

The data that we'll be using is a slightly modified version of some mock data used as an example 
for the [CMG](https://www.genome.gov/Funded-Programs-Projects/NHGRI-Genome-Sequencing-Program/Centers-for-Mendelian-Genomics-CMG) model to be used to submit data to [the AnVIL Project](https://anvilproject.org/). 

With that said, let's get start!



## Whistler Configuration
After cloning this repo, you will find a directory, *data*. There should be 10 CSV files inside that directory which represent 5 different data tables, some of which we'll be transforming into FHIR resources. Those filenames ending with '-dd' are the respective data dictionaries for the data tables. Those without the '-dd' at the end of the filename are the actual data tables themselves. 

To start
