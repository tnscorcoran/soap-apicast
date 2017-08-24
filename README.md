# Integrating SOAP based Web Services into Red Hat 3scale API Management

## Blog
See ***********         LINK TO BLOG         *************** for context and high level details     https://docs.google.com/document/d/1X4omcsZZ3gzMQYEDfPENBmuwaOrMvUePcSMDacTBSTA/edit#

## Overview
Aside from some minor configuration on the 3scale API Manager web interface, the solution mainly involves a small customization to one of the *Lua* scripting source files used by the gateway, *configuration.lua*. For precise details on the customization, use a *diff* tool on the file in this repo with the [one we customized](https://github.com/3scale/apicast/blob/master/apicast/src/configuration.lua) 
We create a Docker image using the standard APICast (aka 3scale Gateway) image and just override this lua file. * Note as discussed in the ***********          LINK TO BLOG         ***************, we use a HTTP Header called SOAPAction. This is because, by convention, this header identifies the SOAP Operation. Should you wish to use another, fork this repo and modify [this line](https://github.com/tnscorcoran/soap-apicast/blob/master/configuration.lua#L200) *

## Instructions
To implement, you simply follow the *3scale API Manager configuration* section below. Then, depending on your desired implementation, follow either the *Raw Docker* or *Openshift* gateway configuration section that follows.  

### 3scale API Manager configuration
On the 3scale API Manager we configure the SOAP endpoint the same way we configure a REST endpoint.

**_Mapping SOAP Endpoint URL path to 3scale method:_**
![Mapping](https://raw.githubusercontent.com/tnscorcoran/soap-apicast/master/_images/1-Mapping.png)
In my case, I use a fictitious Geo-Location SOAP Service - identified by the path /geo-endpoint. I map this to a logical 3scale method geo-service. This will cause all SOAP requests to this endpoint to be authorized and reported to 3scale under this method.

Additionally, in order to get the fine-grained, operation-level access control and traffic visibility, we define 3scale metrics for each operation. Navigate to API -> Your SOAP API -> Definition. Create a Method for each endpoint - in my case just one geo service. Additionally create a Metric for each SOAP operation your Service exposes. In my case I’ve configured possible operations city, country etc.

**_API Definition with method and multiple metrics representing operations:_**
![API Definition](https://raw.githubusercontent.com/tnscorcoran/soap-apicast/master/_images/2-method-metric-definition.png)

At this point, in your 3scale Application Plans (access policies), you could now control access with rate limits to each of these metrics and the method. Of course, to see any numbers, you'll need to configure your gateway and make Web Service calls as detailed below.

In your Analytics section, you now get overall traffic visibility at Method level and at individual metric or operation level.

**_Method level visibility showing overall traffic to SOAP Service endpoint:_**
![Method level visibility](https://raw.githubusercontent.com/tnscorcoran/soap-apicast/master/_images/3-method-level-analytics.png)


**_Metric level visibility showing traffic to individual SOAP operations:_**
![Metric level visibility](https://raw.githubusercontent.com/tnscorcoran/soap-apicast/master/_images/4-metric-operation-level-analytics.png)


### Raw Docker gateway configuration
There are some minor modifications to the instructions given in the [APIcast on Docker](https://support.3scale.net/docs/deployment-options/apicast-docker) documentation. Log on to your RHEL box and make the following commands (or your equivalents):

- sudo su
- systemctl start docker
- git clone **_< this repo or your fork >_** 
- cd soap-apicast 
- docker run --name apicast --rm -p 8080:8080 -v $(pwd)/configuration.lua:/opt/app-root/src/src/configuration.lua -e THREESCALE_PORTAL_ENDPOINT=https://**_<3scale access token>_**@**_<3scale domain>_** -e APICAST_LOG_LEVEL=debug registry.access.redhat.com/3scale-amp20/apicast-gateway

This will take a few seconds to start as it pulls down the image and your configuration from your API Manager, be it On-Prem or from the 3scale SaaS.

Test your Service with your equivalent of the following:

curl -X POST --header "SoapAction: **_<the 3scale system name you configured for this operation>_**" --header "Content-Type: application/soap+xml" --header "Accept: application/soap+xml" --header "user-key: **_< your API key >_**" -d '**_< your SOAP request XML >_**' http://**_< your gateway host >_**:8080/**_< your SOAP endpoint >_** --verbose

Check your 3scale Analytics. Both the operation metric and endpoint method will have incremented.

### Openshift gateway configuration
Some minor changes are needed to the instructions given in the [APIcast on Openshift](https://support.3scale.net/docs/deployment-options/apicast-openshift) documentation. Log on to your RHEL box and make the following commands (or your equivalents):

- sudo su
- systemctl start docker
- oc cluster up --public-hostname=**_< your Openshift domain >_** --routing-suffix=**_< your Openshift IP address >_**.xip.io --version=latest 
- oc login
- oc new-project soap-apicast
- oc new-build https://github.com/tnscorcoran/soap-apicast --strategy=docker
- oc get is/soap-apicast (slot in what this command gives for IMAGE_NAME two lines below)
- oc secret new-basicauth apicast-configuration-url-secret --password=https://**_<3scale access token>_**@**_<3scale domain>_**
- oc new-app -f https://raw.githubusercontent.com/3scale/apicast/v3.0.0/openshift/apicast-template.yml -p IMAGE_NAME=**_< your IMAGE_NAME >_** -p LOG_LEVEL=debug
- oc expose service/apicast (it will give you the route something like: http://apicast-soap-apicast.**_< your Openshift domain >_**)

**_After a couple of minutes 2 Gateway Pods should successfully deploy as follows:_**
![Gateway Deployed](https://raw.githubusercontent.com/tnscorcoran/soap-apicast/master/_images/5-openshift-soap-apicast-overview.png)

Test your Service with your equivalent of the following:

curl -X POST --header "SoapAction: **_<the 3scale system name you configured for this operation>_**" --header "Content-Type: application/soap+xml" --header "Accept: application/soap+xml" --header "user-key: **_< your API key >_**" -d '**_< your SOAP request XML >_**' http://**_< your route exposed by command 'oc expose service' above >_**:8080/**_< your SOAP endpoint >_** --verbose

Check your 3scale Analytics. Both the operation metric and endpoint method will have incremented.
