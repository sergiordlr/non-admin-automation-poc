# non-admin-automation-poc
Prove of concept for non-admin CAM feature automation


## Warning

If you are using 3.x clusters in your CAM deployment, make sure to configure "allowall" identity provider in them. If you are using any other type of identity provider the  automation will not be able to create the users. 

If you don't want to use "allowall" identity provider, then create the user in those 3.x clusters before running the automation.

In 4.x clusters the automation can create the users with no problem.

## Before Automation

To run this automation you need before to:

1. Install non-admin konveyor banch in target cluster, with controller: true
2. Install non-admin konveyor branch in source cluster
3. Add source cluster to target cluster's CAM installation
4. Add a replication repository to CAM

Once those steps are done, we can continue with the automation.

## Automation


### Login to the cluster that is running the controller pod.

```
oc login .....
```

### Create the user and configure CAM for this user

Create the user, the user's namespace and the migration tokens for all the clusters involved

User's namespace for cam will be `{{user}}-cam`. The MigTokens will be created inside this namespace

``` 
# Default user is `testuser`. You can -e user=myuser -e password=mypass to use any other user/pass
ansible-playbook install_cam_for_user.yml -e @config/defaults.yml
``` 


### Run the migration

Deploy a Redis pod and a configuration map to configure it. It will be deployed in namespace `ocp-24997-confmap`.

Then migrate the application and verify that the application was migrated properly.


``` 
# Default user is `testuser`. You can -e user=myuser -e password=mypass to use any other user/pass
ansible-playbook ocp-24997-confmap.yml -e @config/defaults.yml
``` 


### Helper commands

```
oc get MigToken -n testuser-cam
```

```
oc get MigPlan -n testuser-cam
```

```
oc get MigMigration -n testuser-cam
```

```
oc get all -n ocp-24997-confmap
```

### Inventory

An invetory will be generated by reading the information stored in openshift-migation namespace. It should be generated via dyanamic invetory and not using a role like this, but like this is faster to develop for the poc.

These are the groups:
```
                "groups": {
                  ### All cluster
                    "all": [
                        "host", 
                        "source-cluster"
                    ], 
                  ### The cluster running the controller pod
                    "controller_cluster": [
                        "host"
                    ], 
                  ### All clusters that are host clusters (isHost). Theoretically there should be only one
                    "host_clusters": [
                        "host"
                    ], 
                  ### The cluster running the application what will be migrated
                    "source_cluster": [
                        "source-cluster"
                    ], 
                  ### All source clusters
                    "source_clusters": [
                        "source-cluster"
                    ], 
                  ### The cluster where the application will be migrated to
                    "target_cluster": [
                        "host"
                    ], 
                    "ungrouped": []
                }, 

```

All hosts connections will be local, but every host will have the url and tokens to access them via k8s module. This information is stored in its hostvars

url: The url to access this openshift

token: The token that authenticates the user in this host

admin_token: The token that has admin permissions in this host (this token will be migration-controller SA in source-clusters). If migration-controller SA is not enough, it will look for a helper SA. If this helper SA exists then it will use it.
