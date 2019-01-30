---
layout: default
title: ClusterFuzz
parent: Production setup
permalink: /production-setup/clusterfuzz/
nav_order: 1
---

# Setting up a production project
This document walks you through the process of setting up a production project
using ClusterFuzz.

- TOC
{:toc}

---

## Create a new Google Cloud project

Follow [these instructions](https://cloud.google.com/resource-manager/docs/creating-managing-projects)
to create a new Google Cloud Project.

Verify that your project is successfully created using

```bash
$ gcloud projects describe <your project id>
```

Export the project id in environment for later use:

```bash
$ export CLOUD_PROJECT_ID=<your project id>
```

## Create OAuth credentials
Follow [these instructions](https://developers.google.com/identity/protocols/OAuth2InstalledApp#creatingcred)
to create OAuth credentials for our project setup script. When prompted for an
credential type, choose `OAuth client ID -> Other` and provide a name.

Download these credentials as JSON and place it somewhere safe. Export the path in environment
for later use:

e.g.

```bash
$ export CLIENT_SECRETS_PATH=/path/to/your/client_secrets.json
```

## Run the project setup script
Now you can run our project setup script to automate the process of setting up
a production instance of ClusterFuzz.

This script also creates a config directory for you, which contains some default
settings for your deployment and can be later updated.

```bash
$ mkdir /path/to/myconfig  # Any directory outside the ClusterFuzz source repository will work.
$ export CONFIG_DIR=/path/to/myconfig
$ python butler.py create_config --oauth-client-secrets-path=$CLIENT_SECRETS_PATH --project-id=$CLOUD_PROJECT_ID $CONFIG_DIR
```

This can take a few minutes to finish, so please be patient.

Check out the configuration yaml files in `/path/to/myconfig` directory and change the defaults
to suit your usecases. Some common configuration items include:
* Change the default project name using `env.PROJECT_NAME` attribute in `project.yaml`.
* Add access for all users of a domain using `whitelisted_domains` attribute in `gae/auth.yaml`.
* Use a custom domain for hosting (instead of `appspot.com`) using `domains` attribute in
`gae/config.yaml`.

It's recommended to check your `/path/to/myconfig` directory into your own
version control to track your configuration changes and to prevent loss.

## Verification

To verify that your project is successfully deployed.

- Verify that your application is accessible on `https://<your project id>.appspot.com`. If you see
  an error on missing datastore indexes, this usually takes a few minutes to be generated after the
  deployment finished. You can check the status
  [here](https://appengine.google.com/datastore/indexes).

- Verify that the bots are successfully created [here](https://console.cloud.google.com/compute/instances).
  Bots are automatically created via a cron that runs every 30 minutes. For the first time, you can
  manually force it by visiting the `Cron jobs` page [here](https://console.cloud.google.com/appengine/cronjobs)
  and then running the `/manage-vms` cron job. The default settings create 1 regular linux bot and
  2 [preemptible](https://cloud.google.com/preemptible-vms/) linux bots on GCE. This can be
  configured by modifying `/path/to/myconfig/gce/clusters.yaml`.

  If you plan to add windows bot(s), you need set the admin password in the `windows-password`
  project metadata attribute. The password must meet the windows password policy requirements. This
  allows you to rdp into the bot(s) with the `clusterfuzz` username (admin) and your configured
  password.

  ```bash
  $ gcloud compute project-info add-metadata --metadata-from-file=windows-password=/path/to/password-file --project=$CLOUD_PROJECT_ID
  ```

## Deploying new changes
Now that the initial setup is complete, you may deploy further changes by
running:

```bash
$ python butler.py deploy --config-dir=$CONFIG_DIR --prod --force
```

## Configuring number of bots
By default we create 1 non-preemptible Linux VM and 2 [preemptible] Linux VMs on
[Google Compute Engine]. This can be controlled via
`/path/to/myconfig/gce/clusters.yaml`.

Once you make changes to the `clusters.yaml` file, you must re-deploy by
following the [previous section](#deploying-new-changes). An App Engine cron job
will periodically read the contents of this file and create or delete new
instances as necessary.

### Other cloud providers
Note that bots do not have to run on Google Compute Engine. It is possible to
run your own machines or machines with another cloud provider. To do so, those
machines must be running with a [service account] to access the necessary
Google services such as Cloud Datastore and Cloud Storage.

We provide [Docker images] for running ClusterFuzz bots.

[Google Compute Engine]: https://cloud.google.com/compute/
[service account]: https://cloud.google.com/iam/docs/creating-managing-service-account-keys
[Docker images]: https://github.com/google/clusterfuzz/tree/master/docker
[preemptible]: {{ site.baseurl }}/architecture/#bots