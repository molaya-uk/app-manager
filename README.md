# App Manager
An Ansible based framework for managing app deployment for Splunk.

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

## Invocation
A sample invocation of the playbook could look as follow:

    ssh-agent bash
    ssh-add git_key
    ssh-add remote_server_key
    ansible-playbook -i hosts -e template_definition=production.yml main.yml
    
## App Structure
The App and Add-on structure can be amended to include Ansible templates. *header.yml* includes *template_directory* veriable that defines directory name to be used for templates for all the apps. The default value is *templates*. A sample app structure would be similar to the following:

    my_sample_app ---- default/app.conf
                  |
                  |--- local/server.conf
                  |          inputs.conf
                  |
                  |--- templates/passwords.j2
                  
## Extending the playbook to accomodate multiple environments
The playbook can be easily adjusted various deployment options. For example, the script could be extended to support two Indexer clusters (e.g., Prod and Pre-prod), mangaged via master-apps by copying and modyfing the existing play.

Default Play:

    - hosts: cluster_master
      tags: [ master ] 
      remote_user: "{{ remote_ansible_user }}"
      become: "{{ remote_become }}"
      become_user: "{{ remote_become_user }}"
      vars_files:
      - header.yml
      - manifest.yml
      tasks:
      - name: load the environment file
        include_vars:
          file: "{{ template_definition }}"
        when: template_definition is defined
      - include_tasks: "tasks/clear_master.yml"
        when: remove_before_copying
      - include_tasks: "tasks/master_apps.yml"
        with_subelements:
        - "{{ manifest }}"
        - server_tags
        when: item.1 == 'master' or item.1 == 'all'
      - include_tasks: "tasks/master_apply.yml"

Modyfied Play:

    - hosts: production_cluster_master
      tags: [ prod_master ] 
      remote_user: "{{ remote_ansible_user }}"
      become: "{{ remote_become }}"
      become_user: "{{ remote_become_user }}"
      vars_files:
      - header.yml
      - manifest.yml
      tasks:
      - name: load the environment file
        include_vars:
          file: "{{ template_definition }}"
        when: template_definition is defined
      - include_tasks: "tasks/clear_master.yml"
        when: remove_before_copying
      - include_tasks: "tasks/master_apps.yml"
        with_subelements:
        - "{{ manifest }}"
        - server_tags
        when: item.1 == 'prod_master' or item.1 == 'prod_all'
      - include_tasks: "tasks/master_apply.yml"
      
    - hosts: pre_cluster_master
      tags: [ pre_master ] 
      remote_user: "{{ remote_ansible_user }}"
      become: "{{ remote_become }}"
      become_user: "{{ remote_become_user }}"
      vars_files:
      - header.yml
      - manifest.yml
      tasks:
      - name: load the environment file
        include_vars:
          file: "{{ template_definition }}"
        when: template_definition is defined
      - include_tasks: "tasks/clear_master.yml"
        when: remove_before_copying
      - include_tasks: "tasks/master_apps.yml"
        with_subelements:
        - "{{ manifest }}"
        - server_tags
        when: item.1 == 'pre_master' or item.1 == 'pre_all'
      - include_tasks: "tasks/master_apply.yml"








