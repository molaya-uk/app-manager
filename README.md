# App Manager
An Ansible based framework for managing app deployment for Splunk®.

## Introduction
The project contains a simple Ansible based framework that can be used to manage apps and add-ons for Splunk deployments. The main goal of the framework is to provide simple, flexible and extendable solution that can be easily adjusted for a particular platform.

## Features
The framework provides the following features:
- App Manifest - all app and add-on deployment activities are defined within a single manifest file.
- Integrating Templates with Apps and Add-ons - each app and add-on can contain a template directory with Ansible templates specific to that app. This way your templates are not separte for the apps.
- Easy to extend - the framework can be easily extended and adjusted to most platform designs. For example it can be adjusted to support Pre-production and Production environment by copying the existing Plays and providing custom *server_tags*, such as: prod_deployer and np_deployer.

## Overview
The solution consist of a single playbook (main.yml). The playbook contains 5 different plays, responsible for:
- Interactions with Git - pools Git repo containing Apps and Add-ons and places them on the Ansible server running the play.
- Managing deployer - copies Apps and Add-ons to the deployer and applies the bundle
- Managing master-apps - copies Apps and Add-ons to the deployer and applies the bundle.
- Managing deployment-apps - copies Apps and Add-ons to the deployment server and reloads the server
- Managing etc/apps - copies Apps and Add-ons to $SPLUNK_HOME/etc/apps directory on a host and restarts the server

Each play includes tasks responsible for cleaning the relevant directories and copying the apps over and applying the configuration.

The playbook utilizes **manifest.yml** file which is crucial for the framework. The Manifest contains information on where each app should be deployed. Each App that is managed by the playbook requies an entry in the manifest such as the one below:

      - description: A description of the app
        src: test_app_A
        dest: test_app_A
        template_dest: default
        server_tags: ['deployer', 'master', 'deployment' ]
        templates: ['app']

The following attributes are defined by this entry:
- description - The field is not used by the playbook and it is used to provide information on the app being deployed
- src - the path to the app in relation to *git_repo_destination* variable, which defines Git repo location on Ansible server.
- dest - defines the App name that should be used on the destination server
- server_tags - these tags define, where given app will be deployed. They are utilized by the plays defined in the playbook to    filter relevant apps, as seen on the code below:
  
      - include_tasks: "tasks/deployer_apps.yml"
        with_subelements:
        - "{{ manifest }}"   
        - server_tags
        when: item.1 == 'deployer' or item.1 == 'all'

The code includes deployer_apps.yml file that contains tasks to deploy apps, the tasks are invoked multiple times, each time the app in the manifest contains relevant server_tags.
- templates - a list of template files found under $APP_Name/$template_directory/. The files have implicit .j2 extension and are saved to either $APP_Name/local/ or $APP_Name/default directory with .conf extension. On invocation, the playbook accepts *template_definition* veriable, which is used to load template veriables. This allows to load different set of veriables for different environments. A more secure alternative is use of Ansible Vault.
- template_dest - specyfies whether templates should be saved to local or default directory within the app.








