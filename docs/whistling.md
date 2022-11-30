# Whistle 
NCPI Whistler provides some nice railing to help move your data along its way from flat, tabular files into rich FHIR resources living inside a server that provides a REST API for access. However, the real transformation from tabular into FHIR is done using [whistle](https://github.com/GoogleCloudPlatform/healthcare-data-harmonization), a language designed by google to transform data into FHIR Resources. Whistler provides some assistance in that regards, but most of the work will require effort on the part of the ETL author to build out most of the transforms. While it isn't the goal of this tutorial to teach you how to write Whistle, it should give you a reasonable idea of how to do most of what you want to do and where to find more information. 

## Projector Initialization
Whistler provides a simple script to generate some whistle code to assist in some of the more basic tasks like building a common framework for identifiers among other things. This script requires a configuration file and accepts an optional flag, --modules/-m, allowing the user to select between *core*, *dd* and *ALL* (ALL is the default). 

The *core* just creates whistle code that provides some useful functions that are appropriate for datasets of any kind. The *dd* option builds the code necessary to construct various data-dictionary resources consistent with the metadata module from the [NCPI FHIR Implementation Guide](https://nih-ncpi.github.io/ncpi-fhir-ig/study_metadata.html). 

Because the *dd* option depends on *core*, there isn't a difference between *ALL* and *dd*. However, we may add additional modules in the future. 

```bash
$ init-play study.yaml
Selected modules: core,dd

Writing data to projector
Creating files for module, core
Creating files for module, dd
```
If you run the command from the text box above, *init-play study.yaml*, you should see the response listed below it. Now, if you use linux ls to list the contents of the projector directory, you should see something like this:

```bash
$ ls projector/
__create_data_dictionary_conceptmap.wstl
__create_data_dictionary_definitions.wstl
__create_data_dictionary_terminologies.wstl
_age_at_extension.wstl
_build_coding.wstl
_build_terminology_id.wstl
_convert_for_valueset_url.wstl
_ethnicty_extension.wstl
_harmonize_as_code.wstl
_harmonize_as_coding.wstl
     ...
_replace_in_string.wstl
_self_only.wstl
_study_meta.wstl
wlib_dd_conceptmap.wstl
wlib_dd_concept_valusets.wstl
wlib_dd_tables_and_vars.wstl
wlib_dd_terms_codesystem.wstl
wlib_dd_terms_valueset.wstl
```
The ellipses, ..., is where I've deleted some lines from the output to be more concise. As you can see, there are a lot of files present. The directory, *projector*, was the directory we specified in the configuration file for *projector_lib*. 

The funny naming convention used is intended to group different types of files together. The files starting with two underscores, such as *__create_data_dictionary_conceptmap.wstl*, are high level functions that can be called to produce a set of resources. In that particular case, it creates the [Study Data Dictionary ConceptMap](https://nih-ncpi.github.io/ncpi-fhir-ig/study_metadata.html#study-data-dictionary-data-table) which is one of the components specified in the NCPI FHIR IG. 

Files with a single underscore, such as *_replace_in_string.wstl* provides a single function for performing string replacement. Some of the functions are simply utilities, such as the string replace function, while others, such as *_key_identifier.wstl* provide a common interface for creating system/value pairs for FHIR Resource Identifiers to ensure they are consistent. 

Finally, there are those last few that start with the letters, *wlib_dd* are a bit different. Unlike the generic functions defined in files that start with an underscore, these are created dynamically based on the contents of the data dictionaries listed in the configuration file. If you look through those files, you may recognize some of the information found in either the configuration file or some of one of those data-dictionary files in our dataset. 

> It is strongly encouraged to never change any of these files. If your study adds new tables or you have to change something in the study portion of the configuration file, you may need to rerun the init-play program once again and it does not think twice about overwriting existing files. If you wish to change the behavior of one of those functions, the file should be renamed before you make the changes (or better yet, create a new function and use that one instead). 

Earlier, I mentioned that those files that start with 2 underscores contained wrapper functions. In order for those functions to get called, you have to create a master whistle function to call them (and any other whistle code you want to execute). 

To stick with the goofy file naming convention of using multiple underscores to indicate higher level functionality, I typically put my topmost level function in a file called, *___transform.wstl*. You can name your file whatever you like, but it should contain at least one whistle function. For now, let's just have it call those wrapper functions: 

```javascript
def Transform_Dataset(resource) {
    $this: CreateDataDictionaryTerminologies(resource);
    $this: CreateDataDictionaryConceptMap(resource);
    $this: CreateDataDictionaryDefinitions(resource);
}
```
And with those 5 lines, you've written your first whistle code! This defines a function, *Transform_Dataset*, which accepts a single argument, *resource*. The function definition probably looks familiar to anyone who's written javascript before, however, the contents of the function are obviously not javascript. Those three lines are simply mapping the output from a function call to $this. The variable, $this, refers to whatever the current object is. If you were to trace the function calls associated with CreateDataDictionaryTerminologies, you end up inside the *wlib_dd_terms_codesystem.wstl* function at the end of which you can see the following lines: 
```wlib_dd_terms_codesystem.wstl
def ProcessCodeSystem(study, cs_entry) {
    var codesystem: BuildCodeSystem(study, cs_entry);
    if (codesystem?) {
        out ddmeta: codesystem;
        out ddmeta: BuildValueSet(study, cs_entry);
    } 
}
```
This function assigns the output of the function *BuildCodeSystem* to a local variable, *codesystem*. Then, if that returns something other than *null*, it maps that information to the output property, *ddmeta*. It then calls the function, *BuildValueSet* and maps that data to the same output property. 

Most programmers might believe that the output from that BuildValueSet function overwrites the codesystem, but whistle will represent them as an array containing the output from both of the two functions, *BuildCodeSystem* and *BuildValueSet*. And, if you were to call this function multiple times with different cs_entry values, the final value for **ddmeta* will be a list containing all of the CodeSystems and ValueSets produced by each of those calls. Also, there is no need to check if the function returns null. Whistle silently skips null values, making these calls much easier to read without all of the conditional decoration required by more conventional languages. 

As for what that variable *ddmeta* looks like at the end of all of this, it is just a property at the root level of the JSON containing all of the FHIR resources that are produced. In the Whistler program, *play*, this is referred to as a module. If we were to run the newly created function, our final FHIR output file would have 2 of these modules. *ddmeta* and *harmony*, which is created by the function, *CreateDataDictionaryConceptMap*. 

Before we do that, however, we still have one last thing to do before we have a fully functional Whistler pipeline. 

## The Whistle Entrypoint
The whistle entrypoint is just the whistle code that kicks everything off. While it can be more complex, I typically just have a single whistle call inside. This file should exist outside the projector library and must be specified in the configuration under the property, *whistle_src*. It simply calls the function we created above:
```_entry.wstl
$this: Transform_Dataset($root);
```
This should look familiar, except for a couple of things: 
1. There is no function. This is just bare whistle code mappings.  
2. $root is a variable that represents the JSON object that is passed to whistle. It is just the structured extractions from the CSVs, data-dictionaries and study details you described in your configuration file. 

## Generating FHIR Resources
Now that we have our entrypoint, we can build some FHIR resources!

```bash
$ play study.yaml
Writing Harmony ConceptMap: harmony/data-harmony.json
Whistle Path: /usr/local/bin/whistle
🎶 Beautifully played.🎵
Resulting File: output/whistle-output/tut.output.json
Module Summary

Module Name                      Resource Type            #         % of Total
-------------------------------  ------------------------ --------- ----------
ddmeta                           CodeSystem               21         100.00
ddmeta                           ValueSet                 21         100.00
```
If you see something similar to the output above, then all went well and you have some new files you can inspect. Before we get into those files, however, let's go over what we are seeing here. 

The actual script to run the pipeline is *play*, which takes at the very least, one or more configuration files. When you run it, the first thing it does is to compile the harmony csv file into a JSON file containing the dataset's [Harmony ConceptMap](https://nih-ncpi.github.io/ncpi-fhir-ig/StructureDefinition-study-data-dictionary-harmony.html). 

The next bit of info is just the path Whistler has identified for whistle. For this run, I'm using the dockerized version of Whistler, so it lives inside the /usr/local/bin directory. However, if you installed whistle according to the official instructions, then it probably ended up someplace different. 

The next line just confirms that whistle didn't encounter any issues when compiling your FHIR Resources. The next line informs you where those resources can be found. In this case, that file is *output/whistle-output/tut.output.json*. 

Before you open that file up, let's look at the module report that is shown at the very final portion of the output. If you remember from when we were digging through some of the autogenerated functions, there were lines that said something like *out ddmeta: codesystem*. Well, that ddmeta property ended up receiving 21 CodeSystems and 21 ValueSets. 

When you open that file up, you will see that there is one root level property, ddmeta, which is a list that contains a number of JSON objects, 21 of which are CodeSystems and 21 are ValueSets. 

If you have been paying attention, you'll probably wonder where those harmony files ended up. Well, if you don't use the *ALL: true* entry in your active tables, you have to explicitly add harmony along with the normal tables. So, add two more lines to that *active_tables* property to your configuration file:
```study.yaml
# Set any dataset entry to false to filter it out from Whistle's input JSON
active_tables:
  subject: true
  family: true
  conditions: true
  sample: true
  sequencing: true
  discovery: true
  harmony: true
  data-dictionary: true
```
and then rerun that play:
```bash
$ play study.yaml
Writing Harmony ConceptMap: harmony/data-harmony.json
Whistle Path: /usr/local/bin/whistle
🎶 Beautifully played.🎵
Resulting File: output/whistle-output/tut.output.json
Module Summary

Module Name                      Resource Type            #         % of Total
-------------------------------  ------------------------ --------- ----------
ddmeta                           ActivityDefinition       6          100.00
ddmeta                           CodeSystem               21         100.00
ddmeta                           ObservationDefinition    75         100.00
ddmeta                           ValueSet                 21          91.30
harmony                          ConceptMap               1          100.00
harmony                          ValueSet                 2            8.70
```
Those ConceptMap and ValueSet resources for the harmony module were triggered due to the *harmony: true* setting in the active_tables. The ActivityDefinition and ObservationDefinition resources are the result of the *data-dictionary: true* setting. Those two resource types conform to the [Study Data Dictionary](https://nih-ncpi.github.io/ncpi-fhir-ig/study_metadata.html#study-data-dictionary-data-table) resources defined in the NCPI FHIR IG.  

It's worth noting that play does its best to only compile FHIR Resources when needed. So, if you were to rerun play again, you would see the following response: 
```bash
$ play study.yaml
Writing Harmony ConceptMap: harmony/data-harmony.json
Skipping whistle since none of the input has changed
```
This is especially helpful when you are testing out some changes and decide that you are ready to load the data into a different FHIR server. If your whistle code worked fine for DEV, then there is no need to rebuild those resources before you load them into QA. 

You can always force play to recompile your FHIR resources even if it thinks nothing has changed using the "-f" or "--force" arguments. 

## Tabular Data in FHIR
Before we move on to writing actual Whistle code, we can use Whistler to generate some more whistle code. This time, we'll have it create the Whistle code necessary to transform our data tables into [FHIR Observations](https://nih-ncpi.github.io/ncpi-fhir-ig/tabular.html) as defined by the NCPI FHIR IG. 

There are [good reasons](https://nih-ncpi.github.io/ncpi-fhir-ig/using_source_data.html) to capture tabular data in FHIR even if those CSV files can be downloaded elsewhere, but it may not be something everyone agrees with. For that reason, as well as the fact that there are actually two recommended approaches to storing these data, Whistler requires a separate script be run to generate the whistle code. For this example, we'll stick with the slightly more straightforward approach, which is using an Observation for each row of data. To do that, we simply run a single program: 
```bash
$ buildsrcobs study.yaml
File created: projector/source_data_observations.wstl
```
That file, source_data_observations.wstl, is more akin to the wlib_dd* files in that it is based on the contents of the data-dictionaries specified by the study configuration file. There is also a single top level function from that file that we'll need to add to our ___transform.wstl file, *BuildRawDataObs*.

```___transform.wstl
def Transform_Dataset(resource) {
    $this: CreateDataDictionaryTerminologies(resource);
    $this: CreateDataDictionaryConceptMap(resource);
    $this: CreateDataDictionaryDefinitions(resource);
    $this: BuildRawDataObs(resource);
}
```
With this completed, we can run play once again: 

```bash
$ initplay study.yaml
Writing Harmony ConceptMap: harmony/data-harmony.json
Whistle Path: /home/est/bin/whistle
🎶 Beautifully played.🎵
Resulting File: output/whistle-output/tut.output.json
Module Summary

Module Name                      Resource Type            #         % of Total
-------------------------------  ------------------------ --------- ----------
ddmeta                           ActivityDefinition       6          100.00
ddmeta                           CodeSystem               21         100.00
ddmeta                           ObservationDefinition    75         100.00
ddmeta                           ValueSet                 21          91.30
harmony                          ConceptMap               1          100.00
harmony                          ValueSet                 2            8.70
source_data                      Observation              106        100.00
```
With this new version of ___transformation.wstl, play reports we now have a new module, *source_data* which contains 106 Observation resources. Each of those Observations contains all of the non-missing data from one of the rows in our 6 tables. 

If you open the JSON file specified as the Resulting file, output/whistle-output/tut.output.json, you will see there are now 3 root level properties, ddmeta, harmony and source_data. Each of those are lists containing a number of FHIR Resources. 

## Writing Whistle Code

