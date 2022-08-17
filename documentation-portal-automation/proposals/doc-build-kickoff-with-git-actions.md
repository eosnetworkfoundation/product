# Proposal: Kickoff Documentation Build on Git Actions

**Summary**: The purpose is to automate the update of documenation by integrating documenation builds with git actions. 
**Example**: Every pull request would trigger a git action, that would fire a URL to the documenation automation web service. The web service would then run a script to update the documenation. 

## Overview

One service will pull together documenation from various repos, process it, and assemble it into a single documenation portal. The portal currently processes markdown documents into HTML, and uses various software packages to create HTML documents from code.
* Doxygen for C/C++
* Typedoc for Typescript
* Javadoc for Java
* OpenAPI for HTTP methods 

The right hook for integration is when there is a major update to the code, or a new release is created. Both of these action originate in *github*. This sets up a situation where many different repos need to come together into a coheasive, single documentation portal. This proposal lays out the following integration and archiecture:
1. Git hub actions are the signal for document updates
2. There are many seperate repositories with documentation and code 
3. The best integrate is for the documentation portal pull down the repository and process any updates
4. Teams own repos, and they decide which actions should update the documentation

## Architecture Overview
```mermaid
graph LR;
    Repo1-->DocService;
    Repo2-->DocService;
    Repo3-->DocService;
    DocService-->RedisJobQueue;
    DocService-->HTML;
```
* Repos -> notify -> Documenation Portal
* Documentation Portal 
   - clones repo
   - creates updated documentation
   - outputs documentation as HTML files into staging area 
* Documentation Portal copies HTML files from staging area to publically accessible location  

## API
API is a HTTPS request. You can write your own script or utilize a market place script like [http-request-action](https://github.com/fjogeleit/http-request-action). 

### Required Parameters

### Optional Parameters

### Authentication

## Documentation Versions 

## Dependancies and PreConditions 

## Getting Status

## Long Running and Duplicate Requests

## Canceling Requests

## Deleting Documenation Versions
Not supported, currently no way to delete a specific version. This requires a manual intervention to delete a specific version of the documenation for a given repo. 
