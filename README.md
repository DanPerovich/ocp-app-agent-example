## Intro
With this project, you can:
1. Create a development Openshift environment on your Mac laptop
2. Test sidecar deployment pattern for adding the New Relic Java APM agent to an existing Java application container
3. Test init deployment patterns for adding the New Relic Java APM agent to an existing Java application container

This project is based of the excellent work and [writeup by Mangirdas Judeikis](https://blog.openshift.com/patterns-application-augmentation-openshift/) on the Red Hat OpenShift blog.  The instructions have been tested, refined, and updated to bring them in alignment with current versions of OpenShift, Minishift, Docker, K8s, and New Relic.


## Prerequisites 
You will need an Openshift Environment running somewhere. [Minishift](https://github.com/minishift/minishift) is your friend in this case.  You can also get a free trial license to Red Hat OpenShift Online, but it comes with a handful of limitations (ex. only one project at a time during the trial)


Installing Minishift:
1. Install xhyve driver (manual approach): 
https://docs.okd.io/latest/minishift/getting-started/setting-up-virtualization-environment.html#xhyve-manual-install
2. Install Minishift (manual approach): 
https://docs.okd.io/latest/minishift/getting-started/installing.html#installing-manually
3. Follow the Minishift quick start guide to start Minishift and deploy a sample app: https://docs.okd.io/latest/minishift/getting-started/quickstart.html

I recommend always taking the manual approach to installing Minishift and its dependencies.  The brew method was pulling some versions with known bugs when I was running through all this.

You may need to uninstall VirtualBox for Minishift to work.  There is a [yet to be released fix](https://github.com/machine-drivers/docker-machine-driver-xhyve/issues/213) for xhyve that incorrectly tests for older versions of VirtualBox and creates a false positive for the latest version of VirtualBox.

Download and unzip this repo to your workstation:
```
wget https://github.com/DanPerovich/ocp-app-agent-example/archive/master.zip
```

Download and install the missing image streams to your Minishift environment once it is up and running:
```
wget https://raw.githubusercontent.com/jboss-openshift/application-templates/ose-v1.4.5/jboss-image-streams.json
oc create -f ./jboss-image-streams.json -n openshift
## If minishift complains about not being an admin execute the next three commands:
oc login -u system:admin
oc create -f ./jboss-image-streams.json -n openshift
oc login -u developer
```


## Sidecar Container New Relic Java APM Example for Openshift

This example shows how to run a New Relic Java APM agent as a sidecar in the Openshift deployment with your app server. The benefit of running it as a [sidecar](http://blog.kubernetes.io/2015/06/the-distributed-system-toolkit-patterns.html) is that you lifecycle the components separately and avoid big bang changes.

The downside with this approach is that you need to run 2 container at all times, even if the second container does not do anything apart from providing libraries.  This paradigm make sense when 2 processes run independently and need to communicate via API or filesystem. 

1. Create a new project
```
oc new-project sidecar-mon

```
2. Build the New Relic sidecar container
```
oc new-build https://github.com/DanPerovich/ocp-app-agent-example --context-dir=sidecar-pattern/container --name=newrelic-sidecar
```
3. Create a Secret and ConfigMap for the agent configuration
```
oc create secret generic newrelic-apikey --from-literal=API_KEY=<use your APM license key here>
cd ocp-app-agent-example-master/sidecar-pattern/container/newrelic
oc create configmap newrelic-config --from-file=newrelic.yml
```
4. Deploy
```
oc process -f https://raw.githubusercontent.com/DanPerovich/ocp-app-agent-example/master/sidecar-pattern/template/template.yaml -p APPLICATION_NAME=example-app | oc create -f -
oc start-build example-app
```
5. Determine app URL to test in your browser
```
oc status
```
6. Verify data is flowing into New Relic UI


## Init Container New Relic Java APM Agent Example for Openshift

Init containers are great when we need to deliver static content or do main container pre-configuration.  This can be abused so make sure you are not doing something which would brake 12 factor app development principals.

In this scenario we will deploy the same application, but our init container will deploy the New Relic Java APM agent directly to the main container.  In this way we will not be running a second container all the time.  It will be used only to provide the required artifacts. 

1. Create a new project
```
oc new-project init-mon

```
2. Build the New Relic sidecar container
```
oc new-build https://github.com/DanPerovich/ocp-app-agent-example --context-dir=init-pattern/container/ --name=newrelic-init
```
3. Create a Secret and ConfigMap for the agent configuration
```
oc create secret generic newrelic-apikey --from-literal=API_KEY=<use your APM license key here>
cd ocp-app-agent-example-master/init-pattern/container/newrelic
oc create configmap newrelic-config --from-file=newrelic.yml
```
4. Deploy
```
oc process -f https://raw.githubusercontent.com/DanPerovich/ocp-app-agent-example/master/init-pattern/template/template.yaml -p APPLICATION_NAME=init-example | oc create -f -
oc start-build example-app
```
5. Determine app URL to test in your browser
```
oc status
```
6. Verify data is flowing into New Relic UI


## Usefull oc commands
1. If a app gets messed up, it's easiest to just delete it and all of its components (pod, replication controller, service, deploymentconfig, buildconfig, build, container image, route:
```
oc delete all --selector app=<your app name>
```
