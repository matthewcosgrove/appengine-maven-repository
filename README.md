# appengine-maven-repository

Private Maven repositories hosted on Google App-Engine, backed by Google Cloud Storage, supporting HTTP Basic authentication and minimalistic user access control deployed in less than 5 minutes.

   * [Why ?](#why-)
   * [Installation](#installation)
      * [Prerequisites](#prerequisites)
      * [Configuration](#configuration)
      * [Deployment](#deployment)
   * [Artifacts](#artifacts)
   * [Limitations](#limitations)
   * [License](#license)
   
# Why ?

Private Maven repositories shouldn't cost you [an arm and a leg](https://bintray.com/account/pricing), nor requires you to become a [Linux Sys-Admin](https://inthecheesefactory.com/blog/how-to-setup-private-maven-repository/en) to setup, and should ideally be **zero maintenance** and **cost nothing**.

Thanks to Google App-Engine's [free quotas](https://cloud.google.com/appengine/docs/quotas), you'll benefits (for free):

* 5GB of storage
* 1GB of daily incoming bandwidth
* 1GB of daily outgoing bandwidth
* 20,000+ storage ops per day

Moreover, no credit card is required to benefit of those free quotas!

# Installation

## Prerequisites

First of all, you'll need to go to your [Google Cloud console](https://console.cloud.google.com) and create a new project: 

![](http://i.imgur.com/iSt98wWl.png)

As soon as your project is created, a default [Google Cloud storage bucket](https://console.cloud.google.com/storage/browser) has been automatically created for you which provides the first 5GB of storage for free.

## Configuration

Clone (or [download](https://github.com/renaudcerrato/appengine-maven-repository/archive/master.zip)) the source code:

```bash
$ git clone https://github.com/renaudcerrato/appengine-maven-repository.git
```

Edit [`WEB-INF/appengine-web.xml`](src/main/webapp/WEB-INF/appengine-web.xml#L3), and replace the default application ID with your own:

```xml
<?xml version="1.0" encoding="utf-8"?>
<appengine-web-app xmlns="http://appengine.google.com/ns/1.0">
    <application>my-maven-repo</application>
    ...
```

Finally, update [`WEB-INF/users.txt`](src/main/webapp/WEB-INF/users.txt) to declare users, passwords and permissions:

```ini
# That file declares your users - using basic authentication.
# Minimalistic access control is provided through the following permissions: write, read, or list.
# Syntax is:
# <username>:<password>:<permission>

admin:l33t:write
john:j123:read
donald:coolpw:read
guest:guest:list
```
> The `list` permission allows to list the content of your repository (when pointing your browser to your repository URL), but prohibits downloads. The `write` permission implies `read`, which itself implies `list`.


## Deployment

Once you're ready to go live, just push the application to Google App-Engine:

```bash
$ cd appengine-maven-repository
$ ./gradlew appengineUpdate
```

Be aware that the very first time the commands above will run, a browser page will be launched asking you to authorize the Gradle App-Engine plugin to access your Google Cloud account. Just copy the returned authorization code, paste it into your console and press [Enter].

And voilà! Your private Maven repository can be accessed at the following address:

`https://<yourappid>.appspot.com`

# Artifacts

To fetch and/or deploy Maven artifacts to your repository: some additional build tool configuration is required. 

> Ensure you do NOT commit credentials with your code. With Gradle, you can achieve this by amending the following example using the approach specified [here](http://stackoverflow.com/a/12751665/752167) of moving your creds to `~/.gradle/gradle.properties` and only referring to the variable names within your build.

### Gradle

An example deploying artifacts using the maven plugin for Gradle:

```gradle
apply plugin: 'java'
apply plugin: 'maven'

...

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: "https://<yourappid>.appspot.com") {
                authentication(userName: "admin", password: "password")
            }
            pom.version = "1.0-SNAPSHOT"
            pom.artifactId = "test"
            pom.groupId = "com.example"
        }
    }
}
```

Using the above, deploying artifacts to your repository is as simple as:

```bash
$ ./gradlew upload
```

Accessing password protected Maven repositories using Gradle only requires you to specify the `credentials` closure:

```gradle
repositories {
    ...
    maven {
        credentials {
            username 'user'
            password 'password'
        }
        url "https://<yourappid>.appspot.com"
    }
}

```

### Maven

Add the standard `distributionManagement` configuration in your `pom.xml` for dealing with remote Maven repositories as described [here](http://maven.apache.org/plugins/maven-deploy-plugin/usage.html) as well as the `server` configuration in `~/.m2/settings.xml` with the following exception. It is necessary to configure Basic Auth in the `httpHeaders` section within your `server` config which your new private repo requires for Maven to be able to connect to it

```maven
<server>
    <id>my-snapshots-id</id>
    <username>guest</username>
	<password>guest</password>
    <configuration>
        <httpHeaders>
            <property>
                <name>Authorization</name>
                <!-- Base64-encoded "guest:guest" can be encoded online at https://www.base64encode.org/ -->
                <value>Basic Z3Vlc3Q6Z3Vlc3Q=</value>
            </property>
        </httpHeaders>
    </configuration>
</server>
```
where you should use the credentials you set up for write access to the repo. You will need to do this for both snapshots and releases, if that is your intention. 

> If you do not add the additional server config above you will see a build failure when you try to `mvn deploy` with this message  `org.apache.maven.wagon.providers.http.httpclient.impl.auth.HttpAuthenticator handleAuthChallenge
WARNING: Malformed challenge: Authentication challenge is empty`

Now deploying artifacts to your repository is as simple as:

```bash
$ mvn deploy
```

# Limitations

Google App-Engine HTTP requests are limited to 32MB - and thus, any artifacts above that limit can't be hosted.

# License

```
Copyright 2016 Cerrato Renaud

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
