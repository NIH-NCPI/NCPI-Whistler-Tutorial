# Loading Data Into a FHIR Server
NCPI Whistler can optionally load the resources produced during the transformation into a FHIR Server. In the next section, we will instruct Whistler to do exactly that. However, for that to work, you must have access to a FHIR Server that you have write permission to and have properly configured your [*fhir_hosts*](https://nih-ncpi.github.io/ncpi-whistler/#/ref/fhir_hosts) file to properly authenticate to that server. 

Loading data using Whistler is simple to do and requires a single additional argument to *play* call. 

## --env / -e
The recommended way is to pass one of the standard programming environments via the --env flag. These include: *local*, *dev*, *qa* and *prod*. For these to work, you must provide a mapping for these environments to an entry in your *fhir_hosts* file inside your project configuration. The reason for the extra mapping is to allow you to have different servers for related substudies that use the same whistle projections. By mapping *qa* or *prod* to server configurations that are appropriate to only one of the sub-studies, you can more easily load data into the appropriate server without having to remember the exact entry's name. Also, while you can specify multiple project configurations on a single command call, you can only provide a single --env option. So, if you want to build whistle for 3 different substudies and load all of those data into QA, you only need to issue a single call. 

This only works if you've configured the [*env*](https://nih-ncpi.github.io/ncpi-whistler/#/ref/project_config?id=env) property inside your project's config. 

## --host 
The other option is to explicitly specify the entry's name associated with the server you want to load into from your *fhir_hosts* file. 

## Example env Configuration
If you used the example configuration described earlier, you should have the following in your configuration file:
```study.yaml
env:
  local: tutorial
  dev: tutorial
  qa: my-qa-server
  prod: my-prod-server
```
Assuming that your *fhir_hosts* has a valid entry for *tutorial*, then you should be able to use the following command to load your study data into FHIR:

```bash
$ play study.yaml -e local
Writing Harmony ConceptMap: harmony/data-harmony.json
Skipping whistle since none of the input has changed
Attempting to load 65 left-overs.
Load Summary

Module Name                      Resource Type            #         % of Total -------------------------------  ------------------------ --------- ---------- condition                        Condition                64         100.00    ddmeta                           ActivityDefinition       6          100.00    ddmeta                           CodeSystem               22         100.00    ddmeta                           ObservationDefinition    75         100.00    ddmeta                           ValueSet                 22          91.67    harmony                          ConceptMap               1          100.00    harmony                          ValueSet                 2            8.33    patient                          Patient                  9          100.00    research_study                   Group                    1          100.00    research_study                   ResearchStudy            1          100.00    source_data                      Observation              97         100.00    source_data                      Questionnaire            6          100.00    source_data                      QuestionnaireResponse    106        100.00    
0 unloaded resources written to output/whistle-output/invalid-references.json  
dumping IDs to file: output/whistle-output/study-ids.json
```
> Writing Harmony ConceptMap: harmony/data-harmony.json
One thing you may have noticed is that the harmony file always get rebuilt, even if nothing has changed since you last ran Whistler. There is a reason for that. You can share harmony files with other studies that use the same projector library which results in all of those studies reusing the same filename for the concept map that gets passed to Whistle. For the study's data-dictionary vocabulary system properties, the study should be included in the URL. If we didn't recompile that each time we run play, there could be some confusing references to terms that referenced different study IDs. 

> Skipping whistle since none of the input has changed
We've discussed this before. Basically, Whistler only compiles your resources if something has changed (or you explicitly tell it to using the --force/-f argument). 

> Attempting to load 65 left-overs.
This may seem a bit strange, but it's telling you that some resources were encountered that had references to something that wasn't currently loaded. So, those resources were shoved to the back of the queue. Because we didn't get any errors, that means by the time those resources came back around, their dependencies had been loaded and they loaded just fine. 

From there, majority is similar to what you have seen before after the whistle generation is completed. It may be helpful to note that this list only reflects the resources and modules actually loaded. As was mentioned earlier, you can indicate which modules to pass to Whistle in the configuration. However, for the load, you can explicitly select modules and resources to be as selective as you need. So, if you only wanted to load ValueSets from the data-dictionary, you could specify the *-m ddmeta* and *-r ValueSet* in your call to play and you should only see one of the lines shown above for the summary. This is helpful if you only need to update specific type of resources. 

> 0 unloaded resources written to output/whistle-output/invalid-references.json  
This second to the last line points to where you can find the list of invalid-references file. This is helpful if you have an error in your Whistle code that is preventing some resources references from being found. Generally, this is caused by some key resource from simply not being loaded, usually because the whistle hasn't been told do so. However, if there were errors encountered during the load, this file can be of help. 

> dumping IDs to file: output/whistle-output/study-ids.json
This file contains the IDs associated with any resources loaded for this execution. 

# Loading Multiple Studies in One Call
If you have more than one study that use the same projector library, you can have all of their configuration files present in the same project directory. Thanks to the *env* property, you can load many different datasets at once even if their target servers differ. This is helpful if you have to change something that affects multiple studies--you can do so without having to rerun the same command for each of the study configurations that are impacted by the change. 

To load more than one study at a time, simply provide more than one configuration file on the command and play will iterate over that list in sequential order. 

That's pretty much it for loading data.From here, you may want to have a look at some [final thoughts](/final_thoughts).