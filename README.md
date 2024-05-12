# rt

Set up a Repo to locally test and configure BackstopJS
Create or login to Azure DevOps and create a new repo.

Create a new repo in Azure Devops
Copy the URL from the rep and git clone it to your machine.

git clone https://<user>@dev.azure.com/<organisation>/<project>/_git/visual-regression
Install BackstopJS
Install nvm, npm and NodeJs, following docs at https://docs.npmjs.com/downloading-and-installing-node-js-and-npm

Install BackstopJS, following docs at https://github.com/garris/BackstopJS

nvm use node && npm i -g backstopjs
It is necessary to first install BackstopJS globally, if you install it locally, the next command will not work correctly.

After BackstopJS installed successfully, initialise a BackstopJS project

npm init && backstop init
Now you have the project structure with the following files and directories

backstop.json : config file for your tests
backstop_data : directory for reference and test images, and reports, and engine scripts
node_modules
package-lock.json
package.json
Configure BackstopJS
Engine scripts
Move the engine_scripts out of the backstop_data directory and correct the reference inside backstop.json.

mv ./backstop_data/engine_scripts ./engine_scripts
Find the following line in backstop.json …

"engine_scripts": "backstop_data/engine_scripts",
… and adapt it to

"engine_scripts": "engine_scripts",
The reason for moving the engine scripts is that we will want to expose the backstop_data directory at a later stage, but there is no need for us to expose our engine scripts at the same time.

backstop.json
The backstop.json contains information for your tests. You can configure multiple tests and test runs for different environments of your web application. We will setup 2 different config files, to illustrate how to handle multiple test runs on multiple environments. In this example, we will assume a production and a development environment.

Rename backstop.json to backstop.development.json. We will adapt this file first and later bulk rename references for production later.

mv backstop.json backstop.development.json
Inside backstop.development.json, find …

“id”: “backstop_default”,
… and adapt to

“id”: “dev”,
Afterwards adapt the viewports to suit the breakpoints of your web application, e.g.

"viewports": [
    {
      "label": "xs - portrait",
      "width": 320,
      "height": 568
    },
    {
      "label": "xs - landscape",
      "width": 568,
      "height": 320
    },
    {
      "label": "s - portrait",
      "width": 576,
      "height": 1024
    },
    {
      "label": "s - landscape",
      "width": 1024,
      "height": 576
    },
    {
      "label": "m - portrait",
      "width":  769,
      "height": 1024
    },
    {
      "label": "m - landscape",
      "width":  1024,
      "height": 769
    },
    {
      "label": "l",
      "width": 992,
      "height": 1024
    },
    {
      "label": "xl",
      "width": 1200,
      "height": 1024
    },
    {
      "label": "xxl",
      "width": 1400,
      "height": 1024
    }
  ],
Afterwards adapt the report object …

“report”: [“browser”],
… to make it contain test results in Xunit.

“report”: [“browser”,“CI”],
Make sure it uses puppeteer and Google Chrome as browser engine API and browser engine respectively. Also adapt the engine options to remove the unsafe ‘no-sandbox’ option and to make sure your server ignore any HTTPs errors that might occur. Adapt …

“engineOptions”: { “args”: [“ — no-sandbox”] },
… to

"engine": "puppeteer",
"engineOptions": { "ignoreHTTPSErrors": true },
Change the destination for the results at

“html_report”: “backstop_data/html_report/development”,
“ci_report”: “backstop_data/ci_report/development”
Engine Scripts
Inside your engine scripts you can set more variables for Puppeteer. To avoid being detected as a bot, you can add the following user agents to Puppeteer inside the onBefore.js.

module.exports = async (page, scenario, vp) => {
  await require('./loadCookies')(page, scenario);
  await page.setExtraHTTPHeaders({'Accept-Language': 'en-US'});
if (vp.label === 'xs - portrait' || vp.label === 'xs - landscape' || vp.label === 's - portrait' || vp.label === 's - landscape' || vp.label === 'm - portrait' || vp.label === 'm - landscape') {
    await page.setUserAgent('Mozilla/5.0 (Linux; Android 10) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.5195.79 Mobile Safari/537.36');
  }
if (vp.label === 'l' || vp.label === 'xl' || vp.label === 'xxl') {
    await page.setUserAgent('Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36');
  }
};
BackstopJS scenarios
In the same backstop.json file, head over to the scenarios object. Add a scenario for every web page of your web app you wish to test. The options for ignoring selectors, hover actions, click actions, the treshold (scoring strictness) etc can be found in BackstopJS docs at https://github.com/garris/BackstopJS.

Once your file is ready, duplicate it and rename the “development” substring to “production”. Inside the file, rename all URLs with the development domain to the production domain. Also adapt the reference URIs for reference and test images, and html reports to “production”.

Package.json
Inside the packages.json, adapt the scripts object to include multiple instructions, e.g.

{
  "name": "visual-regression-tests",
  "version": "1.0.0",
  "dependencies": {
    "backstopjs": "6.1.1"
  },
  "scripts": {
    "test-development": "backstop test - config='backstop.development.json'",
    "test-production": "backstop test - config='backstop.production.json'",
    "approve-development": "backstop approve - config='backstop.development.json'",
    "approve-production": "backstop approve - config='backstop.production.json'"
  },
  "private": true,
  "license": "ISC"
}
This adaptation will allow you to the following commands in your pipeline:

npm run test-development
or

npm run approve-development
Test run
Run a backstop test with npm run, approve that test and run another test. Debug and adapt the scenarios until you are satisfied with the results. Once you are finished, you are ready to automate the tool with a pipeline.

Create an Azure DevOps Pipeline
Inside your repo, create a YAML config file for your Azure DevOps Pipeline.

touch ./pipeline.yml
Start with defining the triggers, e.g. prevent the pipeline from running after pushes or merges.

trigger: none
pr: none
Continue with defining the schedules, e.g. run the pipeline with repo code from the main branch, daily at 05:30 UTC.

schedules:
- cron: '30 5 * * 1,2,3,4,5'
  displayName: 'Daily schedule'
  branches:
    include:
      - main
  always: true
Define the Azure resource — your RHEL machine — that will run this pipeline.

pool:
  name: 'RHEL'
  demands: 'azure'
Add parameters so you can choose which BackstopJS type or run you want to execute (test vs approve).

parameters:
- name: backstopType
  displayName: Backstop
  type: string
  default: test
  values:
    - test
    - approve
Add the development stage and define two jobs, a test and an approve job. Set the conditions so that the test run can be executed by the schedule we defined earlier. The approve run is something you will want to manually control. Also prevent the Azure Agent from cleaning up the repo after it is done, you will want the data (reference and test images, and reports) to persist. Then do an npm install and run one of the tasks we defined in the backstop.development.json. Make sure this task can continue on errors, otherwise any “failed” report (detecting a visual difference between the reference and test image) will write an error message, causing Azure to fail the task and stop the pipeline. Finally, publish the test results in Xunit, so Azure can parse them and provide them to you in the Test Plans section.

- stage: 'dev'
  displayName: 'DEV'
  dependsOn: []
  jobs:
    - job: 'test_dev'
      displayName: 'Test'
      condition: and(${{ eq(parameters.backstopType, 'test') }}, in(variables['Build.Reason'], 'Manual', 'Schedule'))
      steps:
        - checkout: self
          clean: false
          displayName: 'Checkout Repo'
        - script: |
            npm i
          displayName: 'Install BackstopJS'
        - script: |
            npm run test-dev
          displayName: 'Run tests'
          continueOnError: true
            
       - task: PublishTestResults@2
          displayName: 'Publish test results' 
          inputs:
            testResultsFormat: 'xUnit'
            testResultsFiles: '**/xunit.xml'
            failTaskOnFailedTests: true
            searchFolder: '$(Build.SourcesDirectory)'
            testRunTitle: 'BackstopJS DEV test'
    - job: 'approve_dev'
      displayName: 'Approve'
      condition: and(${{ eq(parameters.backstopType, 'approve') }}, succeededOrFailed())
      steps:
        - checkout: self
          clean: false
          displayName: 'Checkout Repo'
        - script:
            npm i
          displayName: 'Install BackstopJS'
        - script: |
            npm run approve-dev
          displayName: 'Approve tests'
Repeat the same for the production stage. Afterwards, adapt the condition to suit your useage: e,g, if it does not require the dev stage to run first and if the schedule does not need to run the production stage:

condition: and(succeededOrFailed(), ne(variables['Build.Reason'], 'Schedule'))
Put it all together and your pipeline.yml will be finalized, e.g.

trigger: none
pr: none

schedules:
- cron: '30 5 * * 1,2,3,4,5'
  displayName: 'Daily schedule'
  branches:
    include:
      - main
  always: true

pool:
  name: 'RHEL'
  demands: 'azure'

variables:
  npm_config_cache: $(Pipeline.Workspace)/.npm

parameters:
- name: backstopType
  displayName: Backstop
  type: string
  default: test
  values:
    - test
    - approve

stages:
- stage: 'dev'
  displayName: 'DEV'
  dependsOn: []
  jobs:
    - job: 'test_dev'
      displayName: 'Test'
      condition: and(${{ eq(parameters.backstopType, 'test') }}, in(variables['Build.Reason'], 'Manual', 'Schedule'))
      steps:
        - checkout: self
          clean: false
          displayName: 'Checkout Repo'
        - script: |
            npm i
          displayName: 'Install BackstopJS'
        - script: |
            npm run test-dev
          displayName: 'Run tests'
          continueOnError: true
            
        - task: PublishTestResults@2
          displayName: 'Publish test results' 
          inputs:
            testResultsFormat: 'xUnit'
            testResultsFiles: '**/xunit.xml'
            failTaskOnFailedTests: true
            searchFolder: '$(Build.SourcesDirectory)'
            testRunTitle: 'BackstopJS DEV test'
    - job: 'approve_dev'
      displayName: 'Approve'
      condition: and(${{ eq(parameters.backstopType, 'approve') }}, succeededOrFailed())
      steps:
        - checkout: self
          clean: false
          displayName: 'Checkout Repo'
        - script:
            npm i
          displayName: 'Install BackstopJS'
        - script: |
            npm run approve-dev
          displayName: 'Approve tests'

- stage: 'prd'
  displayName: 'PRD'
  dependsOn: []
  condition: and(succeededOrFailed(), ne(variables['Build.Reason'], 'Schedule'))
  jobs:
    - job: 'test_prd'
      displayName: 'Test'
      condition: and(${{ eq(parameters.backstopType, 'test') }}, succeededOrFailed())
      steps:
        - checkout: self
          clean: false
          displayName: 'Checkout Repo'
        - script: |
            npm i
          displayName: 'Install BackstopJS'
        - script: |
            npm run test-prd
          displayName: 'Run tests' 
          continueOnError: true
        
        - task: PublishTestResults@2
          displayName: 'Publish test results' 
          inputs:
            testResultsFormat: 'xUnit'
            testResultsFiles: '**/xunit.xml'
            failTaskOnFailedTests: true
            searchFolder: '$(Build.SourcesDirectory)/$(workingDir)'
            testRunTitle: 'BackstopJS PRD test'
    - job: 'approve_prd'
      displayName: 'Approve'
      condition: and(${{ eq(parameters.backstopType, 'approve') }}, succeededOrFailed())
      steps:
        - checkout: self
          clean: false
          displayName: 'Checkout Repo'
        - script: |
            npm i
          displayName: 'Install BackstopJS'
        - script: |
            npm run approve-prd
          displayName: 'Approve tests'
Upload all your code to Azure DevOps.
Afterwards, head to the Pipelines section. Here, create a new pipeline from the YAML inside your repo. ./pipeline.yml





Create an Azure DevOps Pipeline with an existing YAML config file inside your repo.
Create an Azure Agent
In this blogpost, we will use a Red Hat Enterprise Linux or CentOS (virtual) machine for our self-hosted Azure Agent.

Connect to your RHEL (virtual) machine.

Create a dedicated azure user for executing the Azure Agent’s tasks.

sudo useradd -m azure
Also create a dedicated directory for your azure user, this can be its home directory, but it can also be another directory, e.g.

sudo mkdir /mnt/visual-regression-pipeline
Chown the directory you just created.

sudo chown -R azure:azure /mnt/visual-regression-pipeline
We are chowning recursively, in case you already added files there. Although unnecessary now, you can reuse this command if you ever run into permissions troubles later on.

Create a PAT (Personal Access Token) on Azure DevOps for your user.



Create a PAT on Azure DevOps
Go to your Project Settings and find the section Agent Pools under Pipelines. Create a new Agent Pool and setup a new Agent. Follow the steps in the “Get the agent” popup on Azure Devops.



Create a new Azure Agent Pool and a new Azure Agent.
Once the Azure user is installed, go to Agent Pools, Agents and click the agent you just created. Go to capabilities and add a user-defined capability. In the popup, add “azure” and set its value to “true”.


Add a user-defined capability
You can start the azure listening agent on the RHEL machine by executing the run.sh inside the directory you installed it.

/mnt/visual-regression-pipeline/run.sh
You can also run the Azure listening agent in the background with nohup.

nohup ./run.sh &
You can run the script on boot, by adding an instruction to your root user’s crontab, e.g.

Go sudo …

sudo -i
… open your crontab with nano …

EDITOR=nano crontab -e
… and add the following lines.

@reboot runuser -l azure -c 'nohup ./run.sh &';
Alternatively, ifyou want to start the Azure listening agent every time on boot, you can try to setup the service with the script provided by Azure. However, as the service is started by root and it is only its worker processes that are executed by the defined user, it may cause issues for the Chrome engine, which is not allowed to run as root.

Run the pipeline you created from inside Azure DevOps and see if everything is working correctly.


Example of a successful run
Setup a web server (NginX)
In this blogpost, we will setup NginX as web server because it outperforms Apache: it use less disk space, it can handle more requests and it is more secure by design.

Installation
Install the EPEL config package via DNF.

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
Install the EPEL repository via DNF …

sudo dnf install epel-release
… and update your packages.

sudo dnf update
Install Nginx …

sudo dnf install nginx
… and verify the installation.

sudo nginx -v
Configuration
Configure Nginx to serve from the right directory.
Inside /etc/nginx/conf.d, create a new config file, e.g.

touch ~/etc/nginx/conf.d/visual-regression-pipeline.conf
Inside visual-regression-pipeline.conf add a basic Nginx config to listen to port 4000. If you need more security, adapt the location allow and deny rules.

server {
        listen 4000;
        listen [::]:4000 ipv6only=on;
        access_log /var/log/nginx/visual-regression-pipeline/access.log;
        error_log  /var/log/nginx/visual-regression-pipeline/error.log;
root /mnt/visual-regression-pipeline/1/s/backstop_data/;
        location / {
                allow all;
        }
}
The location /mnt/visual-regression-pipeline/1/s/backstop_data/; is the likely location for your Azure Pipeline. If you find it runs in another directory, change it here. Make sure you put the backstop_data directory as root, otherwise you will have issues later on serving images inside the reports.

If your RHEL machine and development server are both located behind a proxy — e.g. behind a VPN — you might need to set bypass proxy exceptions for your machine, e.g.

Create a proxy.sh file inside /etc/profile.d/

sudo touch /etc/profile.d/proxy.sh &&
echo "export no_proxy=localhost,development.domain.com" >> /etc/profile.d/proxy.sh
This creates a file called proxy.sh inside /etc/profile.d. Upon startup of your machine, any *.sh file inside /etc/profile.d should get loaded, effectively setting the proxy bypass for the defined domains.

Permissions
Set user permissions, SELinux rules and Firewall rules to allow your html reports from inside /backstop_data to be served.

Usergroup
Add the nginx user to the azure group, so it can access all (or most) paths in the Azure Agent directories.

sudo usermod -a -G azure nginx
In case of a mistake — like adding nginx to wheel — , remove nginx from a group with the following line, e.g.

sudo gpasswd -d nginx wheel
Firewall
Allow traffic to pass the RHEL dynamic firewall, e.g. port 4000

sudo firewall-cmd --zone=public --add-port=4000/tcp --permanent
Afterwards, restart the firewall daemon.

sudo systemctl restart firewalld
SELinux
Allow traffic to pass SELinux on a port, e.g. port 4000

sudo semanage port -a -t http_port_t  -p tcp 4000
Allow the files inside the visual regression repo to be served, e.g. for the path /mnt/visual-regression-pipeline/backstop_data/.

semanage fcontext -a -t httpd_sys_content_t "/mnt/visual-regression-pipeline/backstop_data/(/.*)?"
Verify the context has been added inside the Azure working directory with:

ls -lZ /mnt/visual-regression-pipeline/1/s/visual-regression-tests/backstop_data/
HTML Reports
Reports from your test runs should now be available at localhost:4000/html_report/development/index.html & localhost:4000/html_report/production/index.html
