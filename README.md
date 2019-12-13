# GtLab CI auto-deployment example

## Important

This is just some files from one of the projects. This is not for direct use and even not the ideal version of the task it had to decide, I just hope it will be helpful for somebody who sort out with similar problem or with GitLab CI in general. 

## What is it?

Here is devOps files for application with Ruby on Rails API and Vue.js frontend (not a big deal for the process, can be React or some other framework as well). It assumes frontend files to be placed in `frontend/` directory, and Rails API to be in a project root. It designed for using GitLab registry to store docker images.

It will automatically deploy pushes to the master branch on the main port (80), pushes to the dev branch on additional port (81), and all other branches on one of the additional ports (82-89) in order to always have working backup-application. It also can be a default production-staging schema.

There are also some documentation about deployment process in [docs](docs/) folder (thanks to [Stas](https://github.com/stan1slaw)).
