# API Gateway Basicauth function using IDCS
This function provides verification of username and password against IDCS at runtime and allows only authorized users to access  API gateway deployment.

The implementation conforms to the guidelines in the OCI Documentation at https://docs.cloud.oracle.com/en-us/iaas/Content/APIGateway/Tasks/apigatewayusingauthorizerfunction.htm.

As you make your way through this tutorial, look out for this icon ![user input icon](./images/userinput.png).
Whenever you see it, it's time for you to perform an action.


## Prerequisites

[Create users in IDCS](https://docs.oracle.com/en/cloud/paas/identity-cloud/uaids/create-user-accounts.html)

Before you deploy this sample function, make sure you have run step A, B and C of the [Oracle Functions Quick Start Guide for Cloud Shell](https://www.oracle.com/webfolder/technetwork/tutorials/infographics/oci_functions_cloudshell_quickview/functions_quickview_top/functions_quickview/index.html)
* A - Set up your tenancy
* B - Create application
* C - Set up your Cloud Shell dev environment

## List Applications
Assuming your have successfully completed the prerequisites, you should see your
application in the list of applications.
```
fn ls apps
```

## Deploy a function that implements an API
We need another function that will be a target for API Gateway. We suggest [oci-display-httprequest-info-python](../oci-display-httprequest-info-python).
In Cloud Shell, run the *fn deploy* command to build the function and its dependencies as a Docker image,
push the image to OCIR, and deploy the function to Oracle Functions in your application.

![user input icon](./images/userinput.png)
```
cd ../oci-display-httprequest-info-python
fn -v deploy --app <app-name>
```

## Create or Update your Dynamic Group for API Gateway
In order to invoke functions, your API Gateway must be part of a dynamic group.

When specifying the *Matching Rules*, we suggest matching all functions in a compartment with:
```
ALL {resource.type = 'ApiGateway', resource.compartment.id = 'ocid1.compartment.oc1..aaaaaxxxxx'}
```


## Create or Update IAM Policies for API Gateway
Create a new policy that allows the API Gateway dynamic group to invoke functions. We will grant `use` access to `functions-family` in the compartment.

![user input icon](./images/userinput.png)

Your policy should look something like this:
```
Allow dynamic-group <dynamic-group-name> to use functions-family in compartment <compartment-name>
```

For more information on how to create policies, check the [documentation](https://docs.cloud.oracle.com/iaas/Content/Identity/Concepts/policysyntax.htm).


## Configure Identity Cloud Service (IDCS)
Login to IDCS admin console and create, add an Application and select "Confidential Application".
![IDCS-appcreate0](./images/IDCS-appcreate0.png)

Enter a name for your IDCS Application, for example "myAPI".

![IDCS-appcreate1](./images/IDCS-appcreate1.png)

For "Allowed Grant Types", select "Resource Owner". Click *Next*.

![IDCS-appcreate2](./images/IDCS-appcreate2.png)

For Primary Audience, enter anything "display-httprequest-info" for example.
For Scopes, click *Add*. In the dialog box, for field "Scope", enter anything "display-httprequest-info" for example, click *Add*.

![IDCS-appcreate3](./images/IDCS-appcreate3.png)

Click *Next*.

![IDCS-appcreate4](./images/IDCS-appcreate4.png)

Click *Finish*.

![IDCS-appcreate5](./images/IDCS-appcreate5.png)

Now that the application is added, note the *Client ID* and *Client Secret*.

![IDCS-appcreate6](./images/IDCS-appcreate6.png)

Click *Close*.

Click on Configurations tab  under Client Information section click on add scope and select the *application name* from the dropdown. Note the scope value.

![IDCS-appcreate7](./images/IDCS-appcreate7.png)
![IDCS-appcreate8](./images/IDCS-appcreate8.png)

Click *Activate* and click *Ok* in the dialog.

Note the *IDCS URL*, this is the URL you see in your browser URL bar, copy the IDCS url ( For example: https://idcs-xxxxxxxxxxx.identity.oraclecloud.com/ ), client-id, client-secret and scope these values are provided to the Basicauth function.



## Review and customize the function
Review the following files in the current folder:
- [pom.xml](./pom.xml) specifies all the dependencies for your function
- [func.yaml](./func.yaml) that contains metadata about your function and declares properties
- [src/main/java/com/example/fn/BasicAuth.java](./src/main/java/com/example/fn/BasicAuth.java) which contains the Java code

The name of your function *basicauth* is specified in [func.yaml](./func.yaml).

set the following variable in "src/main/java/com/example/utils/ResourceServerConfig.java" to the values  noted while configuring IDCS.
```
public static final String CLIENT_ID = "xxxxxxxxxxx";
public static final String CLIENT_SECRET = "xxxxxxxxx";
public static final String IDCS_URL = "https://idcs-xxxxxxxx.identity.oraclecloud.com";

//INFORMATION ABOUT THE TARGET APPLICATION
public static final String SCOPE_AUD = "display-httprequest-infodisplay-httprequest-info";
```


## Deploy the basicauth function
In Cloud Shell, run the *fn deploy* command to build the function and its dependencies as a Docker image,
push the image to OCIR, and deploy the function to Oracle Functions in your application.

![user input icon](./images/userinput.png)
```
fn -v deploy --app <app-name>
```
## Invoke the basicauth function in cloud shell
In Cloud Shell, run *fn invoke* command to invoke the deployed function, returns active status as true if the token is valid or else returns false.

![user input icon](./images/userinput.png)
```
echo -n '{"type":"TOKEN", "token":"Basic aW5jaGFyYS5zaGFtYW5uYUBvcmFj....."}' | fn invoke <app-name> <func-name>
```

## Create the API Gateway
The functions is meant to be invoked through API Gateway.

![user input icon](./images/userinput.png)

On the OCI console, navigate to *Developer Services* > *API Gateway*. Click on *Create Gateway*. Provide a name, set the type to "Public", select a compartment, a VCN, a public subnet, and click *Create*.

![APIGW create](./images/apigw-create.png)

Once created, click on your gateway. Under *Resources*, select *Deployments* and click *Create Deployment*.

* Provide a name, a path prefix ("/basicauth" for example).
* Under *API Request Policies* Add Authentication
    * Authentication Type: *Custom*
    * Choose the application and the basicauth function
    * For "Authentication token", select *Header*
    * For the "Header Name", enter "Autorization"

Click *Save Changes* when you are finished
![APIGW deployment create](./images/apigw-deployment-create.png)

Click *Next*. Provide a name to the route ("/hello" for example), select methods eg: "GET", select *HTTP-URL* for your back-end.

![APIGW deployment create](./images/apigw-deployment-create-route.png)

Click *Next* and finally, click *Save Changes*.

Note the endpoint of your API Gateway deployment.

![APIGW deployment endpoint](./images/apigw-deployment-endpoint.png)


## Invoke the Deployment endpoint
The function validates if the user information is valid.

![user input icon](./images/userinput.png)

Use the curl command to make the HTTP request
```
 curl -i -u "<username>:<password>" https://d6xxxxxxxxk64.apigateway.us-ashburn-1.oci.customer-oci.com/basicauth/hello
```
If the user is valid gateway will make a call to backend with HTTP200 else
The gateway will reject the request with an HTTP401.



# basicauth
