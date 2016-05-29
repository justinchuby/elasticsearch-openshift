# elasticsearch-openshift
Elasticsearch ready to be deployed to openshift

## How To Set Up Elasticsearch on Openshift

From https://medium.com/@happymacaron/how-to-set-up-elasticsearch-on-openshift-405d0460c818

__With this repo, you can copy the files to your own Openshift repo and start from Step 3__

#### Version
Elasticsearch 2.3.3

#### Reference

Searching with ElasticSearch on OpenShift
https://blog.openshift.com/searching-with-elasticsearch-on-openshift/

How To Install and Configure Elasticsearch on Ubuntu 14.0 4
https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-elasticsearch-on-ubuntu-14-04#prerequisites

Security for Elasticsearch
https://www.elastic.co/downloads/shield

Import/Index a JSON file into Elasticsearch
http://stackoverflow.com/questions/15936616/import-index-a-json-file-into-elasticsearch


## Preparation

* An Openshift account

### Step 1 - Create A DIY Application on Openshift

A free Openshift account allows three applications. To create an application, log in to your Openshift app console at https://openshift.redhat.com/app/console/applications, and follow the steps.

1. Click on the “Add Application” button to start creating a new application.
2. Scroll to the bottom of the page and select the “Do-It-Yourself 0.1” cart.
3. Choose a name for your app at the “Public URL” section. This will be the URL of your application.
4. Hit “Create Application”

Now your application should be created. Don’t close the App Console page just yet, as we will need the git repository information in the next step.

### Step 2 - Download Elasticsearch and Configure it

Clone The Repo to Your Local Machine

You can either install Elasticsearch remotely (https://blog.openshift.com/searching-with-elasticsearch-on-openshift/), or installing it locally in a git repo and then push it to the Openshift server. Here, we are going to use the latter way.

Going back to your App Console, click on the application you’ve just created. Look for the “Source Code” section on the right and copy the URL starting with “ssh://”. This is the link to your Openshift git repo.

__Clone the Openshift repo to your local machine.__ If you are unfamiliar with git, here’s a quick-start guide you may refer to: http://rogerdudler.github.io/git-guide/

Now that you have cloned the repo, we can start installing elasticsearch.

#### Install Elasticsearch Locally

First, **download Elasticsearch** from its website. We need the .tar file. The link for version 2.3.3 is https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.3.3/elasticsearch-2.3.3.tar.gz

Then, go to your repo directory, and 

* unzip the file under the /diy folder in the repo.

* Make a soft link for the elasticsearch folder.

cd into the /diy directory, and create a soft link
```bash
ln -s elasticsearch-2.3.3 elasticsearch
```
In your repo, you will also find the “testrubyserver.rb” file. This is a simple server written in ruby to host the default page. We no longer need it for our purposes. So

* delete the “testrubyserver.rb” file, along with the “index.html” file.

After this, we need to configure elasticsearch a little bit.

* Open the config file at diy/elasticsearch/config/elasticsearch.yml
* Add the lines at the bottom of the file.
```yaml
cluster.name: elasticsearch-${OPENSHIFT_APP_UUID}
node.name: ${OPENSHIFT_GEAR_UUID}

path.data: ${OPENSHIFT_DATA_DIR}elasticsearch/data
path.work: ${OPENSHIFT_TMP_DIR}elasticsearch
path.logs: ${OPENSHIFT_DIY_LOG_DIR}elasticsearch_logs
network.host: ${OPENSHIFT_DIY_IP}
http.port: ${OPENSHIFT_DIY_PORT}
transport.tcp.port: 3306

discovery.zen.ping.multicast.enabled: false
```
We also need to change the Openshift config file so that the server starts your app automatically when it starts.

In your git repo, there is also a hidden folder that stores the config files for Openshift. We need to modify them for the elasticsearch server.

* Open the file “.openshift/action_hooks/start”. Modify it so that it looks like:
```bash
  #!/bin/bash
    # The logic to start up your application should be put in this
    # script. The application will work only if it binds to
    # $OPENSHIFT_DIY_IP:8080
    nohup $OPENSHIFT_REPO_DIR/diy/elasticsearch/bin/elasticsearch &gt; $OPENSHIFT_DIY_LOG_DIR/server.log 2>&1 &
```
* Also “.openshift/action_hooks/stop”
```bash
  #!/bin/bash
    source $OPENSHIFT_CARTRIDGE_SDK_BASH
    
    if [ -z "$(ps -ef | grep elasticsearch | grep -v grep)" ]
    then
        client_result "Application is already stopped"
    else
        kill `ps -ef | grep elasticsearch | grep -v grep | awk '{ print $2 }'` > /dev/null 2>&1
    fi
```

Finally, commit the changes.
```bash
git commit -a
```
Now Elasticsearch is ready to be deployed to the cloud.


### Step 3 - Deploy Elasticsearch to Openshift

You can deploy the app simply by pushing the git repo to Openshift. To do this, cd into your git repo, and
```bash
git push
```
You Elasticsearch server should be deployed successfully by now. In your browser, go to your app’s URL (something like your-app .rhcloud.com (http://rhcloud.com/)). If you see the JSON response like the following, the server is working. Congrats!
```json
{
  "name" : "00000000000000",
  "cluster_name" : "elasticsearch-00000000000000",
  "version" : {
    "number" : "2.3.3",
    "build_hash" : "218bdf10790eef486ff2c41a3df5cfa32dadcfde",
    "build_timestamp" : "2016-05-17T15:40:04Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.0"
  },
  "tagline" : "You Know, for Search"
}
```
### Step 4 - Secure Your Elasticsearch Server

We need to secure our Elasticsearch server because it doesn't have security built into its HTTP interface. In other words, anyone who knows your server’s address can access to your server and modify your data, which is something you don’t want.

Fortunately, the company who built Elasticsearch also developed Shield (https://www.elastic.co/products/shield), a plugin module that secures the Elasticsearch server.

We will need to ssh into our server to install Shield.

* Find the ssh link to your app from the App Console page:
* Open the App Console page
* Click on your app
* Click on  “Want to log in to your application?” on the left and copy the ssh command.

Now in your prompt, copy the ssh command and run it. You should see the welcome message when connected.
```
    *********************************************************************

    You are accessing a service that is for use only by authorized users.
    If you do not have authorization, discontinue use at once.
    Any use of the services is subject to the applicable terms of the
    agreement which can be found at:
    https://www.openshift.com/legal

    *********************************************************************

    Welcome to OpenShift shell

    This shell will assist you in managing OpenShift applications.

    !!! IMPORTANT !!! IMPORTANT !!! IMPORTANT !!!
    Shell access is quite powerful and it is possible for you to
    accidentally damage your application.  Proceed with care!
    If worse comes to worst, destroy your application with "rhc app delete"
    and recreate it
    !!! IMPORTANT !!! IMPORTANT !!! IMPORTANT !!!

    Type "help" for more info.
```

Go to the folder where elasticsearch lives.
```bash
cd $OPENSHIFT_REPO_DIR/diy/elasticsearch/
```
Run
```bash
bin/plugin install license
bin/plugin install shield
```
And restart elasticsearch
```bash
ctl_app stop
bin/elasticsearch
```
Add an admin user (whose name is es_admin)
```bash
bin/shield/esusers useradd es_admin -r admin
```
Now test it in another prompt. Replace “localhost” with your app’s address. 
```bash
curl -XGET 'http://localhost/'
```
You should receive a error message saying you do not have access to the server. Now try this. (Replace “localhost” with your app’s address. )
```bash
curl -u es_admin -XGET 'http://localhost/'
```
Enter the password you just set and you should see the message "You Know, for Search.”

### Optional - Accessing data anonymously

Although the server now is pretty secure, you might be thinking it’s too much of a trouble entering the password every time you search. It’s a good idea to give anonymous user read access to the data.

To do this, add the lines in your elasticsearch.yml file which resides in “diy/elasticsearch/config”.
```yaml
shield.authc:
  anonymous:
    username: anonymous
    roles: user
    authz_exception: true
```
Restart Elasticsearch. Run `curl -XGET 'http://your-app-address/'` in another address and you should see the message without having to enter the password.

If you are now at this step, congratulations! You have now set up the server successfully.

### Feeding data to the server

You are probably starting to wonder how you can feed data to the server. Elasticsearch provides a powerful RESTful api for you to interact with the server, and we will make use of this.

For now, please refer to the following pages for details.

https://www.elastic.co/guide/en/elasticsearch/reference/master/_batch_processing.html
http://stackoverflow.com/questions/15936616/import-index-a-json-file-into-elasticsearch


