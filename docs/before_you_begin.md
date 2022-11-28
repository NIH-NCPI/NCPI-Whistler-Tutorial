# Before You Begin
Before you start the tutorial, be sure that you have 1) cloned this repository so that you have access to the example data files 2) installed NCPI Whistler on your system. 

## Installing Whistler
There are two ways of installing whistler: 
* [Installing Whistler as a Python Application](https://nih-ncpi.github.io/ncpi-whistler/#/whistle?id=whistle-installation)
* [Installing Whistler as a Docker App](https://github.com/NIH-NCPI/dockerized-whistle)

Both approaches will provide the same command line interface. When running the dockerized install script, stub scripts named after the main Whistler console scripts are created which run the image just like the regular application and passes the user's arguments to the container version of the application to result in identical behavior. There is one thing that should be considered, however, when using the dockerized version, those stub scripts pass only one directory to the docker container, ".". This means, everything that you need Whistler to access *must* be found within the directory you are running it from. That won't be a problem for this tutorial, but it is something that should be kept in mind when creating real projects. 

## Write Access to a FHIR Server
For the time being, this is beyond the scope of this tutorial. It is assumed that you will have access to a FHIR Server that conform to one of the 4 [authentication schemes](https://nih-ncpi.github.io/ncpi-whistler/#/ref/fhir_hosts?id=auth-types) Whistler supports.