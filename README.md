# openshift-mysql-backup
Shell script to backup your OpenShift MySQL database

This script has been tested against OpenShift v3.1+

# OpenShift project Setup information

In order for you to initiate a backup through the OpenShift API you need to create a service account and give it the necessary permissions to remote shell into the pod. Once you have the account you can use a simple script to grab the token.

Note: The follow steps need access to the oc binary.

## Service account creation

create the service account:

```
echo '{"apiVersion":"v1","kind":"ServiceAccount","metadata":{"name":"scheduled-cron"}}' | oc create -f -
```

Add the permission:

```
oc policy add-role-to-user edit system:serviceaccount:`oc project -q`:scheduled-cron
```

Get the API Token associated with the service account:
```
SA_TOKEN=$(oc get sa/scheduled-cron --template='{{range .secrets}}{{printf "%s\n" .name}}{{end}}' | grep scheduled-cron-token |  tail -n 1)
API_TOKEN=$(oc get secrets ${SA_TOKEN} --template '{{.data.token}}' | base64 -d)
```
Echo the value for the configuration file or environment variable:

```
echo $API_TOKEN
```

# Configuration

## Overview
The configuration information can be passed in via two methods

* A configuration file that is sourced named `.openshift-mysql-backup.config`. This is the preferred method if executing the script in `crond`.
* Environment variables. This is the preferred method if using this script in a scheduler or a build tool like jenkins.

## Parameters

| Name | Example Value | Description |
| ---- | ----- | ----------- |
| BACKUP_LOC | /var/lib/mysql/data/backups | The location of the MySQL backup folder. You want this location to be backed up to a persistent volume |
| OSE_API | https://10.1.1.2:8443 | The OpenShift master API hostname:port |
| PROJECT_NAME | msyql-db-project | The namespace of your OpenShift project running the MySQL pod |
| POD_SELECTOR | mysql-55-centos7 | This is the OpenShift deployment config associated with MySQL pod. This is used to determine the name of the pod inside the script |
| VERIFY_API_TLS | yes or no | Helpful if you are going against a self-signed certificate associated with the API |
| API_TOKEN | eyJhbGciOiJS... | The API token associated with the service account. Used to authenticate the script against OpenShift |
| DB_BACKUP_NAME_PREFIX | MySQLBackupCustom | Used if you want a custom name for your backup file |
| DATABASES | db1 db2 | A list of database to be backed up separated by a white space. If you want all databases backed up use "all" |


# Future Improvements

* The permissions given to the service account seem heavy handed. Need to restrict the account further to just `rsh`.
* A option to mail on success or failure. Helpful for `crond` scheduling on a host not in OpenShift.
* build CentOS/RHEL 7 builder image that allows you to run this script as a container in OpenShift inside your project
* (ON HOLD) remove the need for oc binary and use `curl` instead to call the OpenShift API.

# Using curl notes

Started work on removing the need to maintain the `oc` binary. Note that it does use [jq](https://stedolan.github.io/jq/) to parse the json output.

Example:

```
DB_POD=$(curl --silent -k -H "Authorization: Bearer $API_TOKEN" $OSE_API/api/v1/namespaces/$PROJECT_NAME/pods\?labelSelector=deploymentconfig=$POD_SELECTOR | jq -r ".items[].metadata.name")

```

There is no method to easily call the exec command via curl. See this [request in the Kubernetes project](https://github.com/kubernetes/kubernetes/issues/30298). In the future calling Kubernetes exec might be easier to handle inside of curl.
