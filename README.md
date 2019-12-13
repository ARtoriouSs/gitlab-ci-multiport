# GtLab CI auto-deployment example

Here is devOps files for application with Vue.js frontend and Ruby on Rails API. It assumes frontend to be placed in frontend/ directory, and Rails API to be in a project root.

It will automatically deploy pushes to the master branch on the main port (80), pushes to the dev branch on additional port (81), and all other branches on one of the additional ports (82-89) in order to always have working backup-application. It also can be a default production-staging schema.

This is just example files, not for direct use.
