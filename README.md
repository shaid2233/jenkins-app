
# Jenkins CI/CD Pipeline Setup

This README file provides a step-by-step guide on how to set up a Jenkins CI/CD pipeline for a project.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Initial Jenkins Setup](#initial-jenkins-setup)
4. [Testing Node.js on Jenkins](#testing-nodejs-on-jenkins)
5. [Workspace Synchronization](#workspace-synchronization)
6. [Jenkins Pipeline Integration with GitHub](#jenkins-pipeline-integration-with-github)
7. [Build Stage](#build-stage)
8. [Troubleshooting](#troubleshooting)
9. [Test Stage](#test-stage)
10. [JUnit Test Report](#junit-test-report)
11. [Playwright End-to-End (E2E) Tests](#playwright-end-to-end-e2e-tests)
12. [Running Stages in Parallel](#running-stages-in-parallel)
13. [Linting](#linting)
14. [Continuous Deployment (CD) with Jenkins and Netlify](#continuous-deployment-cd-with-jenkins-and-netlify)
15. [Build Triggers](#build-triggers)
    - [Periodic Builds](#periodic-builds)
    - [Poll SCM](#poll-scm)
16. [Post-Deployment Tests](#post-deployment-tests)
17. [Staging Deployment](#staging-deployment)
18. [Manual Approval Before Production Deployment](#manual-approval-before-production-deployment)

## Prerequisites

- Docker Desktop installed on your machine
- GitHub account

## Installation

1. Create a new folder on your Windows 10 machine.

2. Clone the Jenkins image into the newly created folder.

3. Open a terminal, navigate to the folder containing the Dockerfile, and run the following command to build the Jenkins image:
   ```
   docker build -t my-jenkins .
   ```

4. Once the build is finished, run the following command to start the Jenkins container:
   ```
   docker-compose up -d
   ```

5. Access Jenkins by opening a web browser and navigating to `http://localhost:8080`.

## Initial Jenkins Setup

1. To obtain the initial Jenkins password, access the Jenkins image in Docker under "View Details".

2. You can find the initial password in the logs or by accessing the terminal of the Jenkins image and using the `cat` command to locate the password.

3. Continue with the Jenkins setup process.

## Testing Node.js on Jenkins

Before deploying your backend and frontend project to Jenkins, you need to test Node.js on your Jenkins server.

1. Create a new pipeline in Jenkins.

2. Install the Docker Pipeline plugin if you haven't already done so.

3. Ensure that Docker Desktop is running.

4. Configure the pipeline with the following code:
   ```groovy
   pipeline {
       agent any
       stages {
           stage('Without Docker') {
               steps {
                   echo 'Without Docker'
               }
           }
           stage('With Docker') {
               agent {
                   docker {
                       image 'node:18-alpine'
                   }
               }
               steps {
                   echo 'With Docker'
               }
           }
       }
   }
   ```

5. In the pipeline, check the Node.js version by running `npm --version`. You should receive a response indicating the version.

## Workspace Synchronization

1. Create a workspace in Jenkins. You can do this by using the `touch` and `ls -la` commands.

2. To use a single workspace with Docker, set `reuseNode = true` in the pipeline configuration.

## Jenkins Pipeline Integration with GitHub

1. Create a new pipeline in Jenkins and select "Pipeline from SCM" (Source Code Management).

2. Provide the URL of the GitHub repository that you forked for your project.

3. Since the repository is public, you can leave the credentials as "none".

4. Specify the branch to build as "main" instead of "master".

5. In your IDE (e.g., GitHub Codespaces), create a simple Jenkinsfile and commit the changes to your GitHub repository.

6. Run the pipeline in Jenkins to ensure that it is communicating with your GitHub repository.

**Note:** The Jenkinsfile must be named exactly `Jenkinsfile` (case-sensitive). Committing the file to GitHub may cause issues if not named correctly.

## Build Stage

1. Create a build stage in the pipeline using the `node:18-alpine` Docker image.

2. Configure the build stage with the following steps:
   ```groovy
   steps {
       sh '''
           ls -la
           node --version
           npm --version
           npm ci
           npm run build
           ls -la
       '''
   }
   ```

   In the build stage, you want to check the npm version and build the project.

   **Note:** We use `npm ci` instead of `npm install` because we don't have access to the `node_modules` folder.

3. After a successful pipeline run, access the workspace under the job to verify that the expected files are created.

## Troubleshooting

If you encounter the error "Cannot connect to the Docker daemon at tcp://docker:2376. Is the docker daemon running?", follow these steps to reset Jenkins and Docker:

1. Navigate to the project folder.

2. Run the following commands to stop and recreate the containers:
   ```
   docker compose down
   docker compose up -d
   ```

## Test Stage

1. Configure the test stage to run after the `npm ci` command in the build stage.

2. Update the test stage in the pipeline as follows:
   ```groovy
   stage('Test') {
       agent {
           docker {
               image 'node:18-alpine'
               reuseNode true
           }
       }
       steps {
           sh '''
               test -f build/index.html
               npm test
           '''
       }
   }
   ```

## JUnit Test Report

Jenkins can generate a test report based on the JUnit XML format. To configure the test report:

1. Create a post-action in the pipeline.

2. Set the post action to always run.

3. Specify the path to the test report XML file.

4. The configuration should look like this:
   ```groovy
   post {
       always {
           junit 'test-results/*.xml'
       }
   }
   ```

## Playwright End-to-End (E2E) Tests

To run Playwright E2E tests for your website:

1. Install the `serve` package globally:
   ```
   npm install -g serve
   ```

2. Add the E2E test stage to your Jenkinsfile using the `mcr.microsoft.com/playwright:v1.39.0-jammy` Docker image.

3. Run the tests using the `npm test` command.

4. To generate an HTML report for the E2E tests, add the following post stage to the pipeline:
   ```groovy
   post {
       always {
           publishHTML([
               allowMissing: false,
               alwaysLinkToLastBuild: false,
               keepAll: false,
               reportDir: 'playwright-report',
               reportFiles: 'index.html',
               reportName: 'Playwright HTML Report',
               reportTitles: '',
               useWrapperFileDirectly: true
           ])
       }
   }
   ```

   **Note:** The Blue Ocean plugin is required to generate the HTML report.

## Running Stages in Parallel

You can nest stages within another stage to run them in parallel.

## Linting

To check for code quality and potential issues, you can integrate a linter into your pipeline.

## Continuous Deployment (CD) with Jenkins and Netlify

1. Sign up for a Netlify account.

2. Deploy the `build` folder of your project to Netlify.

3. Install the Netlify CLI tool in the `node_modules/bin` directory.

4. Configure environment variables in the deploy stage of your pipeline:
   ```groovy
   environment {
       NETLIFY_SITE_ID = 'your-site-id'
   }
   ```

5. Generate a personal access token in Netlify under User Settings > Applications.

6. Store the Netlify token securely in Jenkins as a credential.

7. Use the Netlify token in the pipeline:
   ```groovy
   environment {
       NETLIFY_SITE_ID = 'your-site-id'
       NETLIFY_AUTH_TOKEN = credentials('netlify-token')
   }
   ```

8. Deploy to production using the following command:
   ```
   node_modules/.bin/netlify deploy --dir=build --prod
   ```

   This command takes the `build` folder and deploys it to production.

9. Check the pipeline logs to verify that the deployment was successful.

## Build Triggers

You can configure Jenkins to automatically trigger the pipeline based on a schedule or changes in the GitHub repository.

### Periodic Builds

1. Configure the pipeline in Jenkins.

2. Go to "Advanced Project Options" > "Build Triggers".

3. Select "Build periodically" and specify the schedule using Jenkins cron syntax.

   **Note:** Use the `H` parameter to balance the load between all running jobs.

### Poll SCM

To automatically trigger the pipeline when changes are made in the GitHub repository:

1. Configure the pipeline in Jenkins.

2. Go to "Advanced Project Options" > "Build Triggers".

3. Select "Poll SCM" and specify the schedule using Jenkins cron syntax.

   This will poll the GitHub repository for changes in the Jenkinsfile and automatically run the pipeline if changes are detected.

## Post-Deployment Tests

1. Create a new environment variable called `CI_ENVIRONMENT_URL` to store the URL of the deployed website.

2. Add a new stage in the pipeline to run post-deployment tests using Playwright:
   ```groovy
   stage('Prod E2E') {
       agent {
           docker {
               image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
               reuseNode true
           }
       }
       environment {
           CI_ENVIRONMENT_URL = 'YOUR_NETLIFY_SITE_URL'
       }
       steps {
           sh '''
               npx playwright test --reporter=html
           '''
       }
       post {
           always {
               publishHTML([
                   allowMissing: false,
                   alwaysLinkToLastBuild: false,
                   keepAll: false,
                   reportDir: 'playwright-report',
                   reportFiles: 'index.html',
                   reportName: 'Playwright E2E',
                   reportTitles: '',
                   useWrapperFileDirectly: true
               ])
           }
       }
   }
   ```

## Staging Deployment

Before deploying to production, you can deploy your website to a staging environment using a draft URL.

1. Add a new stage in the pipeline called "Deploy to Staging".

2. Use the following command to deploy to staging:
   ```
   node_modules/.bin/netlify deploy --dir=build
   ```

   This stage is similar to the "Deploy to Production" stage but without the `--prod` flag.

3. The draft URL allows you to verify that the website is working correctly before deploying to production.

## Manual Approval Before Production Deployment

To add a manual approval step before deploying to production:

1. Configure the pipeline in the Jenkinsfile as follows:
   ```groovy
   stage('Approval') {
       steps {
           timeout(time: 15, unit: 'MINUTES') {
               input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
           }
       }
   }
   ```

2. Commit the changes to the Jenkinsfile and push them to your GitHub repository.

3. The pipeline will now include a manual approval step before deploying to production.

That's it! You have now set up a Jenkins CI/CD pipeline for your project. Make sure to test the pipeline and verify that everything is working as expected.
```
