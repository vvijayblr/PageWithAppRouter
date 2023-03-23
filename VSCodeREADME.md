https://blogs.sap.com/2022/06/17/sap-tech-bytes-serve-web-page-with-an-approuter-cloud-foundry-basics-2/

The Web Page
First, we need an index.html file (our web page). I created a very simple UI5 app that displays a MeessagePage and, more interestingly, fetches information about the logged in user from the /user-api/ endpoint, which the approuter provides. We will get into that next.

The Approuter
We will use an approuter, which is essentially a node.js based application (npm package), to authenticate users and route them to our web page. The approuter expects an xs-app.json file describing its routing behaviour. We create this file and put it next to your index.html:

The xs-app.json describes the routing behavior of the application. We defined the welcomeFile as well as the authenticationMethod. The routes array is the most interesting part:

The first route is linked to the sap-approuter-userapi service, which the approuter provides since version 9 (learn more about this service API in this blog post). Defining the route means that whenever we call the URL of our approuter and attach a string that matches the regex pattern of the source (“^/user-api(.*)”), the approuter will forward all request to the defined service. The sap-approuter-userapi service provides information about the logged in user, which we use to greet the user of our web page (see the title of the MessagePage).
The second route points to a localDir (local directory) and in combination with the welcomeFile, forwards all requests (regex pattern “^(.*)$”) that didn’t hit the first route to our index.html. To put it into simpler words, the approuter serves the web application. We also configured the authenticationType to be xsuaa, which will make sense once we bind our app to the Authorization & Trust Management service (short “xsuaa”) on Cloud Foundry. This means all users of the approuter will need to login with this service to get to our web page.

The Package.json
Our approuter is a node.js based application, which means we need a package.json file describing the application and its dependencies (only one in this case). We create the following file on root level of the project (next to all other files):

By the way: The approach we are following here with our approuter is also referred to as a “standalone approuter”. You can read about the differences between a standalone and managed approuters in this blog post.

Deployment Configuration - manifest.yaml
To be able to deploy our approuter to Cloud Foundry (cf push), we need a manifest.yaml file as our deployment descriptor. We create this file on root level of the project:

This file is very similar to the one we used in the previous blog post, but two things are different:

We changed the buildpack from staticfile-buildpack to nodejs-buildpack, as we are now pushing a different kind of application.
We added a service instance called my-xsuaa to the deployment descriptor, which makes sure our app will be bound to this service instance during the deployment. This means that before we can deploy our application, we have to create this service instance in our Cloud Foundry space manually, which we will do next.

Authorization & Trust Management service - xs-security.json
Before we can create an instance of the Authorization & Trust Management service in our Cloud Foundry space, which will later be bound to our approuter, we first have to provide a minimal configuration file for this instance. We create the following xs-security.json file on root level of the project – beware that you might have to change the region (here “us10-001”) of the OAuth redirect uri in case you are planning to deploy to another region):

After logging in to our Cloud Foundry space (with the command cf login), we can run the following command to create the instance of the Authorization & Trust Management service in our space.

cf create-service xsuaa application my-xsuaa -c xs-security.json

Notice that the name of the instance (my-xsuaa) matches the configuration in our manifest.yaml file (necessary for the binding).

Deployment
Next, we can deploy our application to the SAP BTP, Cloud Foundry environment. We should already be logged in to our space. We can execute the cf push command to start the deployment process. The process may take a minute or two. Once it’s done, we can see the URL of the application in the terminal output:

C:\VijayFolder\CloudFoundry\PageWithAppRouter>cf push
Pushing app my-web-page to org 3484523btrial / space dev as velayutham.vijayakumar@gmail.com...
Applying manifest file C:\VijayFolder\CloudFoundry\PageWithAppRouter\manifest.yaml...

Updating with these attributes...
  ---
  applications:
+ - name: my-web-page
+   random-route: true
+   buildpack: https://github.com/cloudfoundry/nodejs-buildpack
+   services:
+   - my-xsuaa
Manifest applied
Packaging files to upload...
Uploading files...
 3.33 KiB / 3.33 KiB [===============================================================================] 100.00% 1s 

Waiting for API to complete processing files...

Staging app and tracing logs...
   -----> Download go 1.19
   -----> Running go build supply
   /tmp/buildpackdownloads/ddcdba8d520546385414ac76bc084798 ~
   ~
   -----> Nodejs Buildpack version 1.8.7
   -----> Installing binaries
   engines.node (package.json): unspecified
   engines.npm (package.json): unspecified (use default)
   **WARNING** Node version not specified in package.json or .nvmrc. See: http://docs.cloudfoundry.org/buildpacks/node/node-tips.html
   -----> Installing node 18.15.0
   Download [https://buildpacks.cloudfoundry.org/dependencies/node/node_18.15.0_linux_x64_cflinuxfs3_8ca46b56.tgz]   Using default npm version: 9.5.0
   -----> Installing yarn 1.22.19
   Download [https://buildpacks.cloudfoundry.org/dependencies/yarn/yarn_1.22.19_linux_noarch_any-stack_32d0e82e.tgz]
   Installed yarn 1.22.19
   -----> Creating runtime environment
   PRO TIP: It is recommended to vendor the application's Node.js dependencies
   Visit http://docs.cloudfoundry.org/buildpacks/node/index.html#vendoring
   NODE_ENV=production
   NODE_HOME=/tmp/contents1518821041/deps/0/node
   NODE_MODULES_CACHE=true
   NODE_VERBOSE=false
   NPM_CONFIG_LOGLEVEL=error
   NPM_CONFIG_PRODUCTION=true
   -----> Building dependencies
   Installing node modules (package.json)
   added 250 packages, and audited 251 packages in 6s
   1 package is looking for funding
   run `npm fund` for details
   19 vulnerabilities (6 moderate, 11 high, 2 critical)
   To address issues that do not require attention, run:
   npm audit fix
   To address all issues (including breaking changes), run:
   npm audit fix --force
   Run `npm audit` for details.
   npm notice
   npm notice New minor version of npm available! 9.5.0 -> 9.6.2
   npm notice Changelog: <https://github.com/npm/cli/releases/tag/v9.6.2>
   npm notice Run `npm install -g npm@9.6.2` to update!
   npm notice
   **WARNING** Unmet dependencies don't fail npm install but may cause runtime issues
   See: https://github.com/npm/npm/issues/7494
   -----> Download go 1.19
   -----> Running go build finalize
   /tmp/buildpackdownloads/ddcdba8d520546385414ac76bc084798 ~
   ~
   Contrast Security no credentials found. Will not write environment files.
   inside Sealights hook
   service is not configured to run with Sealights
   Exit status 0
   Uploading droplet, build artifacts cache...
   Uploading build artifacts cache...
   Uploading droplet...
   Uploaded build artifacts cache (52.1M)
   Uploaded droplet (51.9M)
   Uploading complete
   Cell 66777274-56d5-4ae6-80f0-ea61129612e1 stopping instance f923ad33-e7f9-4a9a-9de3-7c7bcad3223a
   Cell 66777274-56d5-4ae6-80f0-ea61129612e1 destroying container for instance f923ad33-e7f9-4a9a-9de3-7c7bcad3223a

Waiting for app my-web-page to start...

Instances starting...
Instances starting...
Instances starting...

name:                my-web-page
requested state:     started    
isolation segment:   trial      
routes:              my-web-page-reflective-grysbok-fp.cfapps.us10-001.hana.ondemand.com
last uploaded:       Wed 22 Mar 18:25:18 IST 2023
stack:               cflinuxfs3
buildpacks:
isolation segment:   trial
        name                                               version   detect output   buildpack name
        https://github.com/cloudfoundry/nodejs-buildpack   1.8.7     nodejs          nodejs

type:            web
sidecars:
instances:       1/1
memory usage:    1024M
start command:   npm start
     state     since                  cpu    memory    disk      logging      details
#0   running   2023-03-22T12:55:34Z   0.0%   0 of 1G   0 of 1G   0/s of 0/s


When opening the URL, you will notice that you are being redirected to a login page. You can login with the same credentials you use to login to your SAP BTP account. You might get logged in with Single-Sign-On, in which case the login is hardly noticeable, but you can be sure you are logged in if you see your name being displayed in the UI5 app:

And that’s it. We have deployed an approuter, which is authenticating users and serving a web page to the SAP BTP, Cloud Foundry environment.

