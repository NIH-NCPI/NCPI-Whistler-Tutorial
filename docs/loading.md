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

> play study.yaml -e local
