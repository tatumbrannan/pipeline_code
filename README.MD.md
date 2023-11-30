## Creating a GitLab Pipeline

The first step is to copy and paste the new pipeline code within the .gitlab-ci.yml file.

#### GitLab CICD pipelines

GitLab CICD pipelines are organized into jobs and stages. Stages are run in
series, as soon as one stage completes the next stage begins.
The deploy stage is used to deploy dbt to a given environment. Reusable code
has been placed in a number of hidden jobs/functions. These jobs have been
prefixed with a period "." and only run when called from, or extended from,
another job.

The notify stage runs after deploy and is used to send notifications.

stages:

- build
- test
- deploy

Run jobs within the Docker executor
image: tbrannan/upsell-cicd

Initializing the environment for dbt

.dbt_init: &dbt_init

- Creating the logs folder and empty gitlab-ci.log file to allow jobs to
- redirect output
- echo "Initializing the dbt environment"
- Current docker image is core 1.5.0 and plugin 1.5.0. If in a future

#### upgrade

if there's a desire run a different version than what's in the docker image,
the below code can be commented in to upgrade to the desired versions

```
   - echo "Updating dbt" | tee -a "$SENDGRID_GITLAB_CI_LOG"
   - pip install --upgrade dbt-core==1.5.0 dbt-bigquery==1.5.0 > ~/pip.log
  - pip install dbt-core
  - pip install dbt-bigquery
  - pip freeze
  - echo "Installing curl, file, zip, and jq"
  - >
    apt install -y curl \
                file \
                zip \
                jq
  - echo "Getting GitLab secure files"

build-job:
  stage: build
  script:
    - echo "Below is DIR"
    - echo $CI_PROJECT_DIR
    - echo "Compiling the dbt code..."
    - *dbt_init
    - echo $CI_PROJECT_DIR
    - echo ${CI_PROJECT_DIR}
    - dbt deps --profiles-dir ${CI_PROJECT_DIR}/.dbt --project-dir ${CI_PROJECT_DIR}
    - dbt clean --profiles-dir ${CI_PROJECT_DIR}/.dbt --project-dir ${CI_PROJECT_DIR}
    - dbt deps --profiles-dir ${CI_PROJECT_DIR}/.dbt --project-dir ${CI_PROJECT_DIR}
    - dbt seed --profiles-dir ${CI_PROJECT_DIR}/.dbt --project-dir ${CI_PROJECT_DIR}
    - dbt run --profiles-dir ${CI_PROJECT_DIR}/.dbt --project-dir ${CI_PROJECT_DIR}
    - echo "Compile complete."


lint-test-job:
  stage: test
  script:
    - *dbt_init
    - echo "Linting dbt code..."
    - dbt run --profiles-dir ${CI_PROJECT_DIR}/.dbt --project-dir ${CI_PROJECT_DIR} --full-refresh
    - echo "No lint issues found."

deploy-job:
  stage: deploy
  environment: production
  script:
    - *dbt_init
    - echo "Deploying dbt models..."
    - dbt build --profiles-dir  ${CI_PROJECT_DIR}/.dbt --project-dir ${CI_PROJECT_DIR}
    - echo "Application successfully deployed."

```

3. You will need to create a keyfile for this new pipeline and project for procurement-test.
   In order for the ci/cd job to run it will need credentials to access the project, a keyfile is a good way to do this.
   Within bigquery navigate to IAM &Admin then on the left hand side click on Service Accounts, Click on the "Create Service Account" button and fill out the information.
   Within the “Grant this service account access to project” section make sure to check the “BigQuery Job User”.
   Click on the three dots next to the new service account you created and click “manage keys”.
   Click on the drop down that says “add key” and press “create a new key” and choose json.
   Now you have a key that can be utilized.
   This key now has a key id and a key email that will be used in the next step.
4. You will now need to add the keyfile to the project.
   Create a folder called .secrets
   Add a file called keyfile.json
   Within that file, paste the key id that was created in step 3.
5. Navigate to the project’s profiles.yml
   You will be adding this project to the profiles.yml
   Copy and paste into the top of the profiles.yml

```
Project_name:
  target: dev
  outputs:
    dev:
      type: bigquery
      method: service-account
      project: project id
      schema: fin
      location: US
      threads: 16
      service_account: your keyfiles email
      keyfile: .secrets/keyfile.json
6. Update and add to your local profiles.yml


project_name:
  target: dev
  outputs:
    dev:
      type: bigquery
      method: service-account
      project: triple-backbone-406418
      schema: dbt_tbrannan
      location: US
      threads: 16
      service_account:  your keyfiles email
      keyfile: .secrets/keyfile.json
```

7. Test connection by running on your local device dbt debug, which will show whether your connection is working, and if it’s not it will tell you what connection is the problem.

8. Do a dbt build on your local and your new schema name will be present under the project procurement-test

> #### Each time anything is pushed to main there will be a pipeline job run.
