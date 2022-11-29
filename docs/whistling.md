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
And with those 5 lines, you've written your first whistle code! This defines a function, *Transform_dataset*, which accepts a single argument, *resource*. The function definition probably looks familiar to anyone whose written javascript before, however, the contents of the function are obviously not javascript. 


