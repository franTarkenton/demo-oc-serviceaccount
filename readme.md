# Configure Service Account to connect to database in Openshift

High level overview, is we are going to walk through the steps to setup an
environment that allows us to:
* authenticate to openshift using a service account
* establish port forwarding for our database pod

##  Step 1) env config
For running this stuff on windows its much easier if you use the bash shell
that is installed with the git client. If git client is not installed recommend
starting there.

invoking a git cmd prompt with access to unixy toolsets:
`C:\Program Files\Git\git-cmd.exe`


## step 2) get the oc client s/w
* find the url from [here](https://developers.redhat.com/openshift/command-line-tools)
* create the env var OC_URL_SRC with the url to the oc command line
* use curl to download the s/w
* unzip
* add the path for oc to the PATH env var

```
SET OC_URL_SRC=<enter that path here>
curl %OC_URL_SRC% -o oc.zip
unzip oc.zip
SET PATH=%cd%;%PATH%
```

## step 3) create a service account

* log into oc
* identify the parameters you want to run the template with, and put them in a file called **params.txt**, example:

```
app_name=db_serviceaccount
license_plate=dzzd333-tst
sa_role_name=db-remoteaccess-role
sa_name=db-remoteaccess-sa
sa_role_binding_name=db-remoteaccess-rb
```

* process the template using the params from **params.txt**

`oc process -f serviceAccountTemplate.yaml --param-file=params.txt | oc create -f -`

* should create output that looks like:

```
serviceaccount/db-remoteaccess-sa created
role.authorization.openshift.io/db-remoteaccess-role created
rolebinding.rbac.authorization.k8s.io/db-remoteaccess-rb created
```

# step 4) log into oc using service account

In the previous command a service account was created, we now want
to retrieve the token that was created for it.
* log into openshift 4 web ui
* Choose the "Administrator" view (top left below hamburger menu)
* Select the project that you want to use
* User Management->Service Accounts, then select the service account we just created
* Bottom of service account page are two secrets that were automatically created, select the one of type: 'kubernetes.io/service-account-token'
* extract the last secret labelled 'token' and populate an env var called SA_KEY

... log in using service account

```
SET SA_KEY=<key goes here>
SET CLUSTER_URL=<cluster_URL>
oc login --token=%SA_KEY% --server=%CLUSTER_URL%
```

... you should now be logged in as your service account, you can verify using:

`oc whoami`

## Step 5) doing the port forwarding

first you need to figure out the name of the pod that is running your database.  You
can list your pods: 
`oc get pods`

Once you know the name of the pod with your database you can configure port forwarding
like this:
`oc port-forward <databaase pod> 5432:5432`

Ideally you want to be able to retrieve the name of the pod for your database in an
automated way.  some options to do that include:
* pipe the output of `oc get pods -o json` into jq to extract the pod name for your database
* could also use a python script to extract the db pod.

Also you want the windows equivalent of running the port forward command in the background
or on in a different cmd shell, that can be killed after the script is complete.  The 
following will start the command and run it in a separate process, but you then also
need to figure out how to get that task and kill it after your script has completed 
execution.

`start "oc-port-forward" cmd /k  "SET PATH=%cd%;%PATH% & oc port-forward <database pod> 5432:5432"`

then to kill the process:

`taskkill /FI "WindowTitle eq oc*" /T /F`


### Testing Database Connection

Having setup the port forwarding you can test your database connection using something like:
```
set dbuser=guylafleur
set dbpass=truenorth
set dbname=habsdb
psql -h localhost -p 5432 -d %dbname% -U %dbuser% %dbpass%
```


