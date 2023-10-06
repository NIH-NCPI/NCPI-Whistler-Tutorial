# Whistle 
NCPI Whistler provides some nice railing to help move your data along its way from flat, tabular files into rich FHIR resources living inside a server that provides a REST API for access. However, the real transformation from tabular into FHIR is done using [whistle](https://github.com/GoogleCloudPlatform/healthcare-data-harmonization), a language designed by google to transform data into FHIR Resources. Whistler provides some assistance in that regards, but most of the work will require effort on the part of the ETL author to build out most of the transforms. While it isn't the goal of this tutorial to teach you how to write Whistle, it should give you a reasonable idea of how to do most of what you want to do and where to find more information. 

## Projector Initialization
Whistler provides a simple script to generate some whistle code to assist in some of the more basic tasks like building a common framework for identifiers among other things. This script requires a configuration file and accepts an optional flag, --modules/-m, allowing the user to select between *core*, *dd* and *ALL* (ALL is the default). 

The *core* just creates whistle code that provides some useful functions that are appropriate for datasets of any kind. The *dd* option builds the code necessary to construct various data-dictionary resources consistent with the metadata module from the [NCPI FHIR Implementation Guide](https://nih-ncpi.github.io/ncpi-fhir-ig/study_metadata.html). 

Because the *dd* option depends on *core*, there isn't a difference between *ALL* and *dd*. However, we may add additional modules in the future. 

```bash
$ init-play study.yaml --no-profiles
Selected modules: core,dd

Writing data to projector
Creating files for module, core
Creating files for module, dd
```
If you run the command from the text box above, *init-play study.yaml --no-profiles*, you should see the response listed below it. The first argument, *study.yaml* is your configuration file. This informs Whistler where to write the files and, for the dd components, which variables to build resources for. That last argument, *--no-profiles*, just prevents Whistler from attempting to use NCPI FHIR IG profiles when generating these resources. In general, we recommend using those profiles, however, loading the IG is a bit outside the scope of this tutorial, so we are happy with just using vanilla resources. We will discuss profiles later on in this tutorial. 

Now, if you use linux ls to list the contents of the projector directory, you should see something like this:

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
The ellipses, ..., is where I've deleted some lines from the output to be more concise. As you can see, there are a lot of files present. The directory, *projector*, was the directory we specified in the configuration file for *projector_lib*. The flag, --no-profile tells init-play to generate *generic* FHIR resources as opposed to utilizing the NCPI FHIR IG profiles. This makes it possible to validate and load resources into a FHIR Server without having to have loaded the IG first. 

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
In order for whistle to understand how to begin the transformation, we have to point it to the *entrypoint* or *main* function. This is simply a file that kicks everything off and, while it could be longer, ours can be as simple as a single line. 

In the study.yaml file, we had the following:
```study.yaml
whistle_src: _entry.wstl
```
That is how we specify which file will be passed to whistle to begin running things. So, we now need to create this file and provide that function call:
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
$ buildsrcobs study.yaml --no-profiles
File created: projector/source_data_observations.wstl
Please add the function, BuildRawDataObs to your main transform library function
```
Once you run the command above, it asks you to add a new function to your *main transform library function*. This is the function we are calling from the *_entry.wstl* file and is defined inside the file, projector/___transform.wstl. This is just a helper to allow you to quickly get the code generated to be a part of the whistle compilation. 

That file, source_data_observations.wstl, is more akin to the wlib_dd* files in that it is based on the contents of the data-dictionaries specified by the study configuration file. The top level function inside that file, *BuildRawDataObs*, is the function need to add to our Transform_dataset function (inside the ___transform.wstl file) to tie this in with everything else. To do this, simply add it to the bottom of that function as shown below:

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
$ play study.yaml
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
While the whistle code we've used so far could technically suffice for loading research data into FHIR, it wouldn't be particularly interoperable. To do that, we have to write some whistle code specific to the types of resources we want to use to represent our data. While this could be automated to some degree, each dataset format is different and will present unique challenges that can't be solved in a purely generic way like the resources created by the autogenerated code we've already used. So, at least for now, the rest is up to the ETL author. 

### Whistle Code for ResearchStudy
An easy one to start with is the ResearchStudy. We've defined everything we need inside our configuration file, and those properties can be found inside the *study* property of the *$root* object we saw back when we wrote our entrypoint whistle code. To see exactly what we have, we need to open the file that will be passed to whistle. That file can be found as *output/whistle-input/tut.json*. What we are looking for is the *study* property, which happens to be the very first property. 
```tut.json
  "study": {
    "id": "TUT",
    "accession": null,
    "title": "NCPI Whistler Tutorial",
    "desc": "This is just some fake data",
    "identifier-prefix": "https://some-place.org/tut/fhir",
    "url": "https://some-place.org/studies/tut",
    "publisher": "NCPI FHIR Working Group",
    "data-dictionary": [
        (... lots of stuff ...)
    ]
  }
```
The parts of that object we are interested in are right there at the top. We'll be referencing those inside the whistle code. 

> A note about whistle code: the bodies of whistle functions look very similar to JSON objects, which makes writing these transformations very straightforward. However, to make life easier, whistle's property names are written as if they are variable names rather than JSON Object properties. This makes the code much more readable. For assigning a string to a member property, however, you'll still need to enclose the string in quotes. 

Let's have a look at the basic Resource without any data assigned to it before filling it in with our actual data. Copy the following lines into a new file, *projector/study.wstl*.
```JSON
def Study(study) {
    // FHIR Identifier has system/value pairs. We'll be using the study_id
    // and, optionally, the DbGAP accession id if it were present as 
    // identifiers
    identifier: ;

    // The title and description will come from our study object
    title: ;
    description: ;

    // We can assign a FHIR Group resource to our enrollment 
    enrollment:  ;

    // We'll link the study's website here
    relatedArtifact:  ;
    
    status: "completed"  ;
    resourceType: "ResearchStudy";
}
```
Comments are permitted, making it possible to document your whistle code. Whitespace is tolerated, as well, to help with readability. If you are familiar with JSON and FHIR Resources, I think what we have above will be fairly familiar despite the missing data points. 

Each of those property assignments are being done to the *this* object which is what typically is returned by these Whistle functions. So, what is returned by the Study function we defined above is a single object with 7 properties including an *identifier* and a *title*. 

The first thing to note is the single argument passed into the function, *study*. We'll go over where that comes from in a bit, but know that it is that study object from the root level of the whistle input file mentioned earlier. If you look back at the snippet shown above, you can see a few properties that are obvious fits for properties in our example FHIR resource. The example below shows those easy ones filled in:
```JSON
def Study(study) {
    // Tag the resource with the study ID
    meta.tag[]: StudyMeta(study);

    // FHIR Identifier has system/value pairs. We'll be using the study_id
    // and, optionally, the DbGAP accession id if it were present as 
    // identifiers
    identifier: ;

    // The title and description will come from our study object
    title: study.title;
    description: study.desc;

    // We can assign a FHIR Group resource to our enrollment 
    enrollment:  ;

    // We'll link the study's website here
    relatedArtifact:  ;
    
    status: "completed"  ;
    resourceType: "ResearchStudy";
}
```
When the code above is processed by whistle, the value for *study.title* will be assigned to the resource's *title* property, as will the value of *study.desc* be assigned to *description*. This makes these transformations transparent, especially for these most basic assignments. 

We've also added something that is normally considered an optional property, meta.tag. This is a simple way of identifying the resources Whistler creates as belonging to this particular study. We do this because it allows whistler to only catalog resources that reside in a target FHIR server that are associated with this particular study. This function is defined in the file, *_study_meta.wstl*. For more information about the meta.tag property, please see [the manual](https://nih-ncpi.github.io/ncpi-whistler/#/ref/whistle_projections?id=the-importance-of-the-metatag-property). 

For the others, we'll need some functions, one of which was provided when we ran *init-play* earlier in the tutorial. If you look in the projector directory, you'll see a file, *_key_identifier.wstl* which defines a single function, Key_Identifier, shown below: 
```_key_identifier.wstl
def Key_Identifier(study, required resourceType, required value) {
    value: value;
    system: $StrCat(study.identifier-prefix, "/", $ToLower(resourceType));
}
```
This is a simple function that makes building Identifiers consistent. Whistler depends on these identifiers matching for referential integrity, so it is vital that resources that reference other resources be able to reliably build these identifiers. We will use this function to define our ResearchStudy identifier.
```JSON
def Study(study) {
    // FHIR Identifier has system/value pairs. We'll be using the study_id
    // and, optionally, the DbGAP accession id if it were present as 
    // identifiers
    identifier[]: Key_Identifier(study, "ResearchStudy", study.id);

```
That will build a system that is based on the identifier prefix we provided in the configuration as well as the study's ID value, also from the configuration file. If you look at the FHIR Spec for [ResearchStudy](https://hl7.org/fhir/researchstudy.html#resource), you will notice that the *identifier* property is an array. To be correct, our identifier must be written to an array. Without the square braces, *identifier* would just be an object, which is not valid for this resource. For more information about that operator, see the message further down the page. 

This function is also used for referencing our enrollment, though, it is used indirectly via the function, *Reference_Key_Identifier* which can be found in the *_reference_key_identifier.wstl* whistle file. 
```JSON
def Reference_Key_Identifier(study, required resourceType, required value) {
    identifier: Key_Identifier(study, resourceType, value);
}
```
This function defines something that isn't actually a legal FHIR structure, but a Whistler workaround for referential integrity. When whistler is ready to load this resource, it will use that Identifier to work out the correct resource to reference to. If that resource has already been loaded, it will already know what to replace that identifier property with. If not, it push the study resource to the back of the queue and try again later. 

As a bit of a side-note, when whistler first connects to the FHIR server, it will enumerate all resources that could possibly meet the study's criteria and builds up the ID list. So, even if you didn't instruct Whistle to build the resource being referenced, if it exists in the server, Whistler will be able to correctly provide the reference. 

This is exactly what we need for our enrollment property. We want to add a reference to an enrollment group containing all subjects in our study as described in the [IG's StudyGroup](https://nih-ncpi.github.io/ncpi-fhir-ig/StructureDefinition-study-group.html). 

```JSON
    // We can assign a FHIR Group resource to our enrollment 
    enrollment[0]:  Reference_Key_Identifier(study, "Group", study.id);
```
This assignment assumes that when we define that Group resource, that we do so using the Key_Identifier using the same parameters as we used for the reference. 

That leaves one last property for us to define, relatedArtifact. We could write this directly inside the Study function, but realistically, it could be used for more than one value, so it makes better sense to write a new function to do the work for us. Luckily, it's very easy to do. Since this is specific to the ResearchStudy, why don't we just add it to the same file we have been writing to for the Study function: 

```JSON
def StudyArtifact(artifact_type, artifact_label, required artifact_url) {
    type: artifact_type;
    label: artifact_label;
    url: artifact_url;
}
```
This is a pretty boring function, but a) it is reusable, and b) it is clear as to what it does--creating an object with the three properties that were passed to the function. That *required* decorator for the *artifact_url* argument tells whistle to return *NULL* if artifact_url doesn't exist. Whistle quitely drops anything that is *NULL*, so if our study configuration didn't define this URL, that object would not appear in the final FHIR Resource. 

Now that we have that function, we can add it to Study function body as needed:
```JSON
    // We'll link the study's website here
    relatedArtifact[]:  StudyArtifact("documentation", "Study Website", study.url);
```
If we had more URLs, we could add those as well using different labels. 

> *Whistle Side Note:* that empty pair of squared brackets at the end of relatedArtifact instructions whistle that relatedArtifact should be a list and for it to add the return of the function call to the list. This is very helpful if the return could be NIL. It also allows you to make repeated function calls to the same list (each time with empty square braces) knowing that any of those calls that produce non-NULL values will be appended to the list. 

The final Study function should now look like this:
```JSON
def Study(study) {
    // Tag the resource with the study ID
    meta.tag[]: StudyMeta(study);

    // FHIR Identifier has system/value pairs. We'll be using the study_id
    // and, optionally, the DbGAP accession id if it were present as 
    // identifiers
    identifier[]: Key_Identifier(study, "ResearchStudy", study.id);

    // The title and description will come from our study object
    title: study.title;
    description: study.desc;

    // We can assign a FHIR Group resource to our enrollment 
    enrollment[0]:  Reference_Key_Identifier(study, "Group", study.id);

    // We'll link the study's website here
    relatedArtifact[]:  StudyArtifact("documentation", "Study Website", study.url);
    
    status: "completed"  ;
    resourceType: "ResearchStudy";
}
```

Now, there is one last function we should add to this file. While it probably isn't necessary for this particular resource, since there will likely only ever be a single study produced by a single study config, it's good practice to be consistent. This function will work like those that we ran earlier from the Transform_Data function except this one will call our new Study function:

```JSON
def ProcessStudy(study) {
    out research_study: Study(study);
}
```
This should be familiar enough. Basically, we accept something called *study* which is then passed to our new function, *Study*. The return value for that is stored in something called *research_study*. This *out research_study* treats the result differently than what we've seen before. Rather than appending to a local property called research_study, it works directly on *research_study* on the main output object. As a result of a successful Study call, there will be a new module in the final output called *research_study*. 

We are almost done with the research study. The final step is to call this new Process function from our Transform_Dataset function. We'll add this as the last line in that function: 

```___transform.wstl
def Transform_Dataset(resource) {
    $this: CreateDataDictionaryTerminologies(resource);
    $this: CreateDataDictionaryConceptMap(resource);
    $this: CreateDataDictionaryDefinitions(resource);
    $this: BuildRawDataObs(resource);

    // Project Specific functions
    $this: ProcessStudy(resource.study);
}
```
If you compare that new call to the previous ones, we are not passing the entire input object, but only the study portion. That's why we were referring to the incoming data as *study* in those various function parameters. 

At this point, we can run play and look at our handiwork:
```bash
$ play study.yaml
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
research_study                   ResearchStudy            1          100.00
source_data                      Observation              106        100.00
```
That second to the last line there confirms that we did, in fact, succeed in creating a ResearchStudy resource. If you want to have a look at it, open up the output file, *output/whistle-output/tut.output.json*, and look for the property, research_study. It should look like this:
```JSON
  "research_study": [
    {
      "description": "This is just some fake data",
      "enrollment": [
        {
          "identifier": {
            "system": "https://some-place.org/tut/fhir/group",
            "value": "TUT"
          }
        }
      ],
      "identifier": {
        "system": "https://some-place.org/tut/fhir/researchstudy",
        "value": "TUT"
      },
      "meta": {
        "tag": [
          {
            "code": "TUT",
            "system": "https://some-place.org/tut/fhir/researchstudy"
          }
        ]
      },
      "relatedArtifact": [
        {
          "label": "Study Website",
          "type": "documentation",
          "url": "https://some-place.org/studies/tut"
        }
      ],
      "resourceType": "ResearchStudy",
      "status": "completed",
      "title": "NCPI Whistler Tutorial"
    }
  ],
```
Keep in mind that *research_study* is a property of the root object that happens to be a list, and isn't actually FHIR in any way. For this situation, it only holds a single resource, but most of the time, there could be quite a few resources inside those root level lists. That one object there, however, should look familiar since it's what we've been working on. Everything you see should validate just fine against a vanilla R4 FHIR server, except that enrollment property. For Whistler to be able to load this ResearchStudy, we'll need to define that as well. 

## Whistle Code for StudyGroup
The NCPI FHIR IG specifies that there *must* be at least one *StudyGroup* for a formal NCPI Research Study. The [StudyGroup](https://nih-ncpi.github.io/ncpi-fhir-ig/StructureDefinition-study-group.html) is defined as a Group containing human study participants (FHIR Patients). To create this group, copy the lines below into a file, *projector/group.wstl*.

```group.wstl
// Description: Build basic reference to a Patient 
//
//  study - This is the full study object created by Whistler
//  subject - Must have a subject_id
def Reference_Enrollment(study, subject) {
    if ($IsNotNil(subject.subject_id)) {
        entity: Reference_Key_Identifier(study, "Patient", subject.subject_id);
    }
}

def StudyGroup(study, subjects) {
    meta.tag[]: StudyMeta(study);

    identifier[]: Key_Identifier(study, "Group", study.id);
    resourceType: "Group";
    type: "person";
    actual: true;

    member: Reference_Enrollment(study, subjects[]);
    quantity: $ListLen(subjects[*]);
}

// Description: Wrapper to create the group(s)
//
//  study - This is the full study object created by Whistler
//  subjects - All of the subjects participating in this study
def ProcessStudyGroup(study, subjects) {
    out research_study: StudyGroup(study, subjects);
}
```
Let's skip that first function for now. For the second function, we see a few familiar pieces here: meta.tag, identifier. Hopefully, the constants make sense as well. The type is "person". The members will be enumerated, so we set *actual* to true. And the resource type is "Group", which is a standard [FHIR Resource](https://hl7.org/fhir/group.html). 

In addition to these, that function still has a couple of things we haven't seen:
> member: Reference_Enrollment(study, subjects[]);

This line is a bit different from some of the others. If you don't already know Whistle, you may be a bit confused by the syntax, *subjects[]*. When whistle encounters this line, it will call that *Reference_Enrollment* once for each subject in the *subjects* list. So, if there are 10 subjects in our dataset, there will be 10 different calls.

That *Reference_Enrollment* call simply builds identifiers for one subject and assigns it to the property, *entity*. So, if we have 10 subjects, when whistle is run *member* will have a property called *entity* which will be a list of 10 reference identifiers. Whistle doesn't overwrite values, but instead aggregates them. So, rather than overwriting member.entity with each of the 10 runs, it converts entity into a list after the second return and appends each subsequent call to the end. 

> quantity: $ListLen(subjects[*]);
This line introduces a couple more tricks. First, there is the function $ListLen which is a whistle [built-in function](https://github.com/GoogleCloudPlatform/healthcare-data-harmonization/blob/master/mapping_language/doc/builtins.md#listlen). Even more interesting, however, is the syntax, *subjects[*]*. In the *Reference_Enrollment* call, we saw the empty square braces which instructed whistle to pass the contents of the list one at a time. When you include the asterisk inside it as we see here, it tells whistle to pass them all together. So, the function ListLen will receive all 10 of our subjects and return the correct number. 

> out research_study: StudyGroup(study, subjects);
This should look pretty familiar now. We are just passing on the study and subjects to the new StudyGroup function and adding them to out's research_study. 

And, as before, we have to add this new Process function to our Transform_dataset function. However, there is a bit of a wrinkle with this one. Have a look at the new code below with the new lines being there at the end of the otherwise familiar Transform_Dataset function:
```___transform.wstl
def Transform_Dataset(resource) {
    $this: CreateDataDictionaryTerminologies(resource);
    $this: CreateDataDictionaryConceptMap(resource);
    $this: CreateDataDictionaryDefinitions(resource);
    $this: BuildRawDataObs(resource);

    // Project Specific functions
    $this: ProcessStudy(resource.study);

    // Assemble our list of subjects by concatenating the various family members
    var subjects: $Unique($Flatten(resource.family[*].subject));

    // Build the StudyGroup
    $this: ProcessStudyGroup(resource.study, subjects);
}
```
There is quite a bit of new stuff going on with the second from the last command there: 
> var subjects: $Unique($Flatten(resource.family[*].subject));
Earlier we learned that the whistle operator "[\*]" effectively aggregates all of a list's contents together. However, since we don't actually want the family objects, we aggregate the families' subjects. And, because each of those family's have an array of subjects, we end up with three arrays of subjects, so we use the whistle builtin, [$Flatten](https://github.com/GoogleCloudPlatform/healthcare-data-harmonization/blob/master/mapping_language/doc/builtins.md#flatten) to turn those 3 nested arrays into one. Then, while it isn't necessary for this particular dataset, we use the builtin *$Unique* to make sure we don't get any duplicates. If this were real data, it does seem reasonable that some people may end up in multiple families and we definitely wouldn't want them to be duplicated in the final result. 

| A note about passing lists to functions. In the example above where we have, *quantity: $ListLen(subjects[*]);*, that was used as an example. It would have worked fine as simply *subjects*, which is what you see in subsequent function calls. It just seemed to be a gentler way to introduce the syntax for *families[*].subjects*, which would not have worked without the asterisk and square braces. 

With this addition in place, we can now run play once again:
```bash
$ play study.yaml
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
research_study                   Group                    1          100.00
research_study                   ResearchStudy            1          100.00
source_data                      Observation              106        100.00
```
And if we view open that output file up, we can see our shiny new Study Group. 
```JSON
    {
      "actual": true,
      "identifier": [
        {
          "system": "https://some-place.org/tut/fhir/group",
          "value": "TUT"
        }
      ],
      "member": [
        {
          "entity": {
            "identifier": {
              "system": "https://some-place.org/tut/fhir/patient",
              "value": "fd-sub1"
            }
          }
        },
        {
          "entity": {
            "identifier": {
              "system": "https://some-place.org/tut/fhir/patient",
              "value": "fd-sub2"
            }
          }
        },
        {
          "entity": {
            "identifier": {
              "system": "https://some-place.org/tut/fhir/patient",
              "value": "fd-sub3"
            }
          }
        },
        (**SEVERAL MORE "entity" ENTRIES**)
      ],
      "meta": {
        "tag": [
          {
            "code": "TUT",
            "system": "https://some-place.org/tut/fhir/researchstudy"
          }
        ]
      },
      "quantity": 9,
      "resourceType": "Group",
      "type": "person"
    }
  ],
```
As you can see, our function did what we wanted by adding all 9 of our subjects as member objects with an "entity" property that will get transformed into a proper reference when we load this data into a real FHIR server. You can also see the results of the $ListLen call, which is 9. 

## FHIR Patients as Our Subjects
FHIR Was initially designed as a common API for accessing EHR data. So, rather than participants or subjects, we have *Patient* resources. To create these key resources, we need to create another set of functions, and we'll write them to a file named, *projector/subject.wstl*. 

```subject.wstl
def Subject(study, subject) {
    meta.tag[]: StudyMeta(study);
    identifier[]: Key_Identifier(study, "Patient", subject.subject_id);
    gender (if subject.sex ~= "."): HarmonizeAsCode(subject.sex, "sex");
    extension[]: RaceExtension(subject.ancestry);
    extension[]: EthnicityExtension(subject.ethnicity);
    resourceType : "Patient"
}

def ProcessSubject(study, subject) {
    out patient: Subject(study, subject);
}
```
Some of this should look familiar, but we have several new features to discuss: 

### Inline If Statements
One of the major goals of Whistle is to make the language itself as unobtrusive as possible so that the transformations can be as apparent to the casual reader as possible. We've already seen some of this by using the different forms of the square brackets to eliminate what would normally be a loop. This is also apparent with the way whistle drops empty properties and returns without any intervention on the part of the ETL Author. 

To make conditionals cleaner, whistle recommends an inline application of the if statement, which is what we see in the assignment of gender: 
> gender (if subject.sex ~= "."): HarmonizeAsCode(subject.sex, "sex");
That if inside the parenthesis tells whistle only to execute the HarmonizeAsCode function if the subject.sex is not ".". That "." is just a possible value that someone was using to annotate missingness in their data. This would be dataset specific, since some folks may have a specific code to signify missingness, or just permit the value to be blank. 

That line also has something else that is new, the function *HarmonizeAsCode*. This is another Whistler function that was provided by the init-play call. As you might have guessed, it can be found in the file, projector/_harmonize_as_code.wstl. At the core, these functions use whistle's $HarmonizeCode builtin to convert the value for the subject's sex to the appropriate code specified by the FHIR Spec. We haven't really dug into the harmony file so this provides a good opportunity to touch a bit on that. 

If you were to open the file, harmony/data-harmony.csv, the first few lines look like this:
```csv
local code,text,table_name,parent_varname,local code system,code,display,code system,comment
Male,Male,subject,Sex,sex,male,Male,http://hl7.org/fhir/administrative-gender,
Female,Female,subject,Sex,sex,female,Female,http://hl7.org/fhir/administrative-gender,
```
Those lines tell whistle that the local code, *Male*, should be mapped to the remote code, "male". Normally, a Harmony call returns one or more terms including the code, display and system, however, because the gender require codes from a particularly system, http://hl7.org/fhir/administrative-gender, all we really need is the code. Hence the use of that particular function. 

### The Harmony's "local code system" value is key
When you are calling one of the Harmony functions provided by Whistler, such as HarmonizeAsCode, the spelling for that second parameter is vital. If you were to misuse case or use *gender* for the local code system, the Harmony call would not find a match and would return NIL. 

> **About Harmony** Whistler provides a number of helper functions that make your whistle code easier to ready, but those functions assume that you have only one Harmony file and that it is named *data-harmony.csv*. For more information about Whistler Harmony files and their importance in FHIR ETL, take a look at the [reference manual](https://nih-ncpi.github.io/ncpi-whistler/#/harmony).

OK, in order for whistle to actually use the new Subject function, we have to tie it into our main function, Transform_Dataset. As before, we've added it at the bottom of the function:
```___transform.wstl
def Transform_Dataset(resource) {
    $this: CreateDataDictionaryTerminologies(resource);
    $this: CreateDataDictionaryConceptMap(resource);
    $this: CreateDataDictionaryDefinitions(resource); 
    $this: BuildRawDataObs(resource);

    // Project Specific functions
    $this: ProcessStudy(resource.study);

    // Assemble our list of subjects by concatenating the various family members
    var subjects: $Unique($Flatten(resource.family[*].subject));

    // Build the StudyGroup
    $this: ProcessStudyGroup(resource.study, subjects);

    // Build the Patient Resources
    $this: ProcessSubject(resource.study, resource.subject[]);
}
```
That should look pretty familiar by now, so I won't spend much time discussing it. However, know that the function took a single Subject at a time, and we have called it using resource.subject[], which means that the ProcessSubject function will be called 9 times. Also, the ProcessSubject is sending those out to the patient property on the main out structure. 

Let's run play one more time and then have a look at some of those subjects:
```bash
$ play study.yaml
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
patient                          Patient                  9          100.00
research_study                   Group                    1          100.00
research_study                   ResearchStudy            1          100.00
source_data                      Observation              106        100.00
```
If you open the output file, output/whistle-output/tut.output.json, and search for the root property, "patient", you should 9 patients. Here is the first one:
```output/whistle-output/tut.output.json
{
    "extension": [
    {
        "extension": [
        {
            "url": "ombCategory",
            "valueCoding": {
            "code": "2028-9",
            "display": "White",
            "system": "urn:oid:2.16.840.1.113883.6.238",
            "version": "v1"
            }
        },
        {
            "url": "text",
            "valueString": "White"
        }
        ],
        "url": "http://hl7.org/fhir/us/core/StructureDefinition/us-core-race"
    },
    {
        "extension": [
        {
            "url": "ombCategory",
            "valueCoding": {
            "code": "2135-2",
            "display": "Hispanic or Latino",
            "system": "urn:oid:2.16.840.1.113883.6.238",
            "version": "v1"
            }
        },
        {
            "url": "text",
            "valueString": "Hispanic or Latino"
        }
        ],
        "url": "http://hl7.org/fhir/us/core/StructureDefinition/us-core-ethnicity"
    }
    ],
    "gender": "male",
    "identifier": [
    {
        "system": "https://some-place.org/tut/fhir/patient",
        "value": "fd-sub1"
    }
    ],
    "meta": {
    "tag": [
        {
        "code": "TUT",
        "system": "https://some-place.org/tut/fhir/researchstudy"
        }
    ]
    },
    "resourceType": "Patient"
}
```
Thanks to those extensions, this looks a lot longer than it really is. However, despite all the extra stuff, I hope you can see that we have a White, Hispanic Male. 

## FHIR Conditions for Research
We have a table, conditions.csv which contains just a few columns: *subject_id*, *condition_code*, *condition_name* and *present_absent*. To represent the conditions that are present, we'll use the FHIR Condition resource. To transform entries from this table into FHIR, we might use something like the whistle code below:
```projector/condition.wstl
// Build our Condition Identifiers
def BuildConditionIdentifier(study, required condition_code, required subject_id) {
    $this: Key_Identifier(study, "Condition", $StrCat(subject_id, ".", condition_code));
}

// Build the Condition Resource
def Condition(study, condition) {
    // Skip this entirely if there is no condition_code
    if (condition.condition_code?) {
        // HarmonizeMapped filters out *self* references 
        var coding: HarmonizeMapped(condition.condition_code, "condition_code");

        // Skip this if our condition didn't map to anything
        if (coding) {
            meta.tag[]: StudyMeta(study);
            identifier[]: BuildConditionIdentifier(study, condition.condition_code, condition.subject_id);
            resourceType: "Condition";
            subject: Reference_Key_Identifier(study, "Patient", condition.subject_id);
            verificationStatus.coding[]: BuildCoding("confirmed", "Confirmed", "http://terminology.hl7.org/CodeSystem/condition-ver-status");
            verificationStatus.text: "Present"
            code.coding: coding;
            code.text: condition.condition_name;
        }
    }
}

def ProcessCondition(study, condition) {
    out condition: Condition(study, condition);
}
```
### BuildConditionIdentifier
That first function is a bit of a convenience function. In order to ensure that each identifier for our conditions is unique, we need to use two variables: *subject_id* and *condition_code*. That builtin, *$StrCat* just concatenates each of its arguments together. So, $StrCat("Whistle", " ", "is", " ", "awesome") would return a single string, "Whistle is awesome". 

We could just bypass the extra function and directly perform that string concatenation each time we need it, but that would make the code a bit less readable and could add a bit of risk, since order of those arguments is absolutely necessary for them to be correct. 

The last thing to note is the use of *required* in the argument list. When a function call is made and any of the *required* arguments are NIL, the function is not executed. Now, the bad thing here might be that the identifier end up as NIL if either of those are missing, but Whistler will actually halt if it encounters a resource that should have an identifier that doesn't. 

### Condition
Most of this should look familiar, however there are a few new things to discuss here: 
> if (condition.condition_code?) {
That question mark after condition_code is basically testing for something other than NIL. So, if there is no condition_code for some reason, we don't really need to do anything more after this point. 

> var coding: HarmonizeMapped(condition.condition_code, "condition_code");
HarmonizeMapped is a convenience function that only returns those translations (from your harmony file) that match some external source. We'll discuss this a bit more in depth later. 

> verificationStatus.coding[]: BuildCoding("confirmed", "Confirmed", "http://terminology.hl7.org/CodeSystem/condition-ver-status");
> verificationStatus.text: "Present"
The function, *BuildCoding* is another convenience function that basically just assembles a valid FHIR Coding out of the three arguments. We could have done that inline but this makes things a bit easier to read. 

As for setting the verificationStatus.text to "Present", that is just a convention that we recommend. If you have something that is transformed to a formal term, it's generally helpful to provide the source data, or something akin to the source value for the text. In this case, since it was HPO Present, we simply indicate that the Condition is present, which is what we are trying to say with that *confirmed* code. 

### ProcessCondition
This should look very familiar by now. Nothing new here at all. 

> **A Note About Self Reference Harmony Entries** It may seem strange to have mappings back to the local code, but this is useful if there is valuable information from the data-dictionary that can be used to populate a FHIR resource. For instance, if you have a column that is basically a tick mark for some sort of disease, the data-dictionary may have a nice, human readable version that would work better in the text field than, oh..say 1 or True or whatever the tick is. In that case, you would use something like, *HarmonizeLocalDisplay* which would filter out real matches and only return the display value for the first entry. 
> 
> While these are helpful for highly specific use cases, the vast majority of uses will employ the *HarmonizeMapped* function or one of its derivatives (most of the Harmonize* functions call HarmonizeMapped and then do something with the return). These self reference entries do not contain valid systems and therefore, will not pass FHIR Validation. 

As before, we add a call to the new process function to the end of our Transform_Dataset function:
```projector/___transform.wstl
def Transform_Dataset(resource) {
    $this: CreateDataDictionaryTerminologies(resource);
    $this: CreateDataDictionaryConceptMap(resource);
    $this: CreateDataDictionaryDefinitions(resource);
    $this: BuildRawDataObs(resource);

    // Project Specific functions
    $this: ProcessStudy(resource.study);


    var subjects: $Unique($Flatten(resource.family[*].subject));
    // Build the StudyGroup
    $this: ProcessStudyGroup(resource.study, subjects);

    // Build the Patient Resources
    $this: ProcessSubject(resource.study, subjects[]);

    // Build the Condition Resources
    $this: ProcessCondition(resource.study, resource.conditions[]);
}
```
Hopefully, this addition needs no additional explanation. 

With that in place, we can now run play once again. 
```bash
$ play study.yaml 
Writing Harmony ConceptMap: harmony/data-harmony.json
Whistle Path: /home/est/bin/whistle
🎶 Beautifully played.🎵
Resulting File: output/whistle-output/tut.output.json
Module Summary

Module Name                      Resource Type            #         % of Total
-------------------------------  ------------------------ --------- ----------
condition                        Condition                64         100.00
ddmeta                           ActivityDefinition       6          100.00
ddmeta                           CodeSystem               22         100.00
ddmeta                           ObservationDefinition    75         100.00
ddmeta                           ValueSet                 22          91.67
harmony                          ConceptMap               1          100.00
harmony                          ValueSet                 2            8.33
patient                          Patient                  9          100.00
research_study                   Group                    1          100.00
research_study                   ResearchStudy            1          100.00
source_data                      Observation              106        100.00
```
Now we have another module which contains 64 conditions. If we open the output file, we can see our conditions under the root level "condition" list. 
```output/whistle-output/tut.output.json
{
  "code": {
    "coding": [
      {
        "code": "HP:0000076",
        "display": "Vesicoureteral reflux",
        "system": "http://purl.obolibrary.org/obo/hp.owl",
        "version": "v1"
      }
    ],
    "text": "Vesicoureteral reflux"
  },
  "identifier": [
    {
      "system": "https://some-place.org/tut/fhir/condition",
      "value": "fd-sub1.HP:0000076"
    }
  ],
  "meta": {
    "tag": [
      {
        "code": "TUT",
        "system": "https://some-place.org/tut/fhir/researchstudy"
      }
    ]
  },
  "resourceType": "Condition",
  "subject": {
    "identifier": {
      "system": "https://some-place.org/tut/fhir/patient",
      "value": "fd-sub1"
    }
  },
  "verificationStatus": {
    "coding": [
      {
        "code": "confirmed",
        "display": "Confirmed",
        "system": "http://terminology.hl7.org/CodeSystem/condition-ver-status"
      }
    ],
    "text": "Present"
  }
}
```
This should look clear enough. The code property contains a coding which consists of the output of the Harmony call and our verificationStatus is a full coding that contains the 3 parameters we used for the BuildCoding call. 

While there are a lot more resources that we should define if we want to build a complete ETL For the data at hand, but these examples should be sufficient for you to now build the rest on your own. Some tips as to what sort of resources would be recommended include:

* ResearchSubject which links Patients to a ResearchStudy via the FHIR convention.
* Observations which capture the HP Codes that are noted as *absent*.
* Observations and Groups to represent Families and the individual relationship pairs. 
* Sample resources
* File Metadata for sequencing data as DocumentReferences
* discovery information that conforms to the resources specified by [the Genomics Reporting IG](http://hl7.org/fhir/uv/genomics-reporting/index.html).

For those who have access to a FHIR server where they have permission to write data can continue on to [Loading Data Into FHIR](/loading).

If you aren't ready to actually load data into a FHIR Server, you can skip the load part and take a look at the [Final Thoughts](/final_thoughts).
