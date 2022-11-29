# Before You Begin
Before you start the tutorial, be sure that you have 1) cloned this repository so that you have access to the example data files 2) installed NCPI Whistler on your system. 

## Installing Whistler
There are two ways of installing whistler: 
* [Installing Whistler as a Python Application](https://nih-ncpi.github.io/ncpi-whistler/#/whistle?id=whistle-installation)
* [Installing Whistler as a Docker App](https://github.com/NIH-NCPI/dockerized-whistle)

Both approaches will provide the same command line interface. When running the dockerized install script, stub scripts named after the main Whistler console scripts are created which run the image just like the regular application and passes the user's arguments to the container version of the application to result in identical behavior. There is one thing that should be considered, however, when using the dockerized version, those stub scripts pass only one directory to the docker container, ".". This means, everything that you need Whistler to access *must* be found within the directory you are running it from. That won't be a problem for this tutorial, but it is something that should be kept in mind when creating real projects. 

## Write Access to a FHIR Server
While Whistler can create FHIR Resources and not submit them to a FHIR Server via the REST API, the load functionality is an important component of NCPI Whistler and is covered in the tutorial. 

For the time being, this is beyond the scope of this tutorial. It is assumed that you will have access to a FHIR Server that conform to one of the 4 [authentication schemes](https://nih-ncpi.github.io/ncpi-whistler/#/ref/fhir_hosts?id=auth-types) Whistler supports.

## Git
I won't get into how to install git on your machine however, [google may be of some help](https://duckduckgo.com/?q=installing+git&ia=web). One thing to note is that the tutorial assumes you are using git via the command line. However, there is no reason a person familiar with a graphical version shouldn't be able to make sense of the commands and how to perform the same thing using their preferred interface. 

## Bash Commands
To keep the tutorial clean and simple, everything is written using bash commands. While that may not be your preferred method for running tools via the command-line, it is an option that is available to everyone whether they use Mac, Windows or Linux. 

## Next Step - Project Setup
To jump right into downloading the dataset and building out the configuration, read [The Setup](/the_setup).