victims-server-openshift
========================

This repo allows you to deploy a new instance of the victims-web server on openshift.

## Running on OpenShift
### Prerequisites
1. You have a valid account with OpenShift https://www.openshift.com/get-started
2. You have the s2i tool installed locally https://github.com/openshift/source-to-image

### Log in to Openshift
```sh
oc login https://console.starter-us-east-1.openshift.com
```

### Create a new project
```sh
oc new-project victims
```

### (Optional) Deploy a database
It's recommended to use a MongoDB database hosted outside of Openshift. However for development purposes a temporary database can provisionsed inside Openshift using the provided template:
```sh
oc process -f mongodb-ephemeral.yaml | oc create -f -
```
*Due to the authentication mechanism being updated to SCRAM-SHA1 in Mongo 3.x, we have a requirement to use an earlier version of MongoDB, eg. 2.6*
*The emphemeral template doesn't use permanent storage with a persistent volume, all data will be lost when the pod is migrated!*

### Create the builder image 
```sh
make
```

This will build a local image with the name victims-rhel7

*A RHEL Subscription will be required to build this image. You can get a $0 subscription here: https://docs.openshift.org/latest/dev_guide/builds/index.html#defining-a-buildconfig*

### Build the victims-web image with s2i tool, using the previously built builder image
```sh
s2i build -c . --context-dir=src victims-rhel7 victims-web 
```
*The ''-c' argument tells s2i to use the local copy, not one stored in the local git repository*
*The --context-dir argument tells s2i to use the 'src' subdirectory of the code source (current directory when passing '.')

### Test the build image
It's now possible to run the image locally to test any changes:
```sh
docker run -it -e MONGODB_DB_HOST=<localhost> victims-web
```

### Push the builder image into Openshift
You'll need to login to the openshift docker registry using your token. The token can be obtained by first logging into Openshift:
```sh
oc login https://console.starter-us-east-1.openshift.com
```
*In this case we're using Openshift online, however if you have a custom install of Openshift Container Platform, please use that URL instead of starter-us-east-1.openshift.com*

Then obtain your token like so:
```sh
oc whoami -t
```

Then login to the docker registry:
```sh
docker login -u <openshift-username> -p <token> registry.starter-us-east-1.openshift.com
```

Push the build image to the registry:

```sh
sudo docker push registry.starter-us-east-1.openshift.com/victims/victims-rhel7
```

### Create the app using image
```sh
oc process -f web-bc.yaml | oc create -f -
```

### Importing data
1. Login to the server with `oc`
2. Setup port forward with `oc port-forward <mongodb-pod-name>
3. Download the data file you want to import to ```$OPENSHIFT_DATA_DIR```
4. Run the following command replacing ```$INPUTFILE``` with your file name.

```sh
mongoimport -d victims -c hashes --type json --file $OPENSHIFT_DATA_DIR/$INPUTFILE  -h localhost  -u admin -p $MONGODB_ADMIN_PASSWORD --port 27017
```
### Configuring the deploytment
#### Application Configuration
Any application configuration can be pushed by changing the ```configs/victimsweb.cfg``` file in the repo. This will be used instead of the ```application.cfg``` provided by victims-web.

#### Build Hook Configuration


