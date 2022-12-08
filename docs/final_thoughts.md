# Final Thoughts
## Profiles
The NCPI FHIR Implementation Guide (IG) provides some recommended profiles with the goal of providing consistency for research data stored in FHIR. It is therefore, highly recommended that those storing research data in FHIR use those profiles. 

At the very least, loading the IG can be done by simply loading the individual profiles and CodeSystems and ValueSets via the standard FHIR REST API. Google provides [non-standard methods](https://cloud.google.com/healthcare-api/docs/how-tos/fhir-profiles), as do most of the other platforms. However, each of them will be specific to the platform. 

*TODO* put together a script that will load the NCPI FHIR IG into a target FHIR Server. 

## Other Topics
???