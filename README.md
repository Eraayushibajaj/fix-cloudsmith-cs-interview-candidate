**Welcome to Cloudsmith Support Assessment**

Please review the documentation provided by the recruiter for full instructions.

**Repository Overview**

This repository contains a simple Python package that can be built and published to Cloudsmith using GitHub Actions.

**Setup**

1. Clone the repository to your local machine
2. Create a new private repo in your Github namespace
3. Upload the code to the newly created Github repo
4. Start debugging!
5. Make sure to share access with the appointed people in the email
6. Include a markdown file which includes all 6 issues found 

**Build and Publish Process**

The process involves three workflows:

### 1. `build_package.yml`

* Triggered by a push or pull request to the main branch.
* Builds a Python package and saves it as an artifact in GitHub Actions.

### 2. `release_package.yml`

* Triggered by the `build_package.yml` workflow completing successfully.
* Downloads the artifact from GitHub Actions and pushes it to the staging repository on Cloudsmith.

### 3. `promote_package.yml`

* Triggered manually by the repository maintainer.
* Promotes the package from the staging repository to the production repository on Cloudsmith.

**Authentication**

OIDC Authentication must be used for the project, API Key solutions will be rejected.
