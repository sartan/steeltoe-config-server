# Steeltoe app + Config Server

This repo uses code from [Steeltoe Samples](https://github.com/SteeltoeOSS/Samples) to demonstrate deploying your own
Config Server to Cloud Foundry and connecting it to your Steeltoe application. Additionally, it documents examples
of fetching configuration Config Server.

## Deploy your app
```
cd SimpleCloudFoundry
cf push --hostname <unique-app-hostname>
```

## Deploy our own Config Server

```
cd configserver
./mvnw package -Dmaven.test.skip=true
cf push config-server -p target/configserver-0.0.1-SNAPSHOT.jar --hostname <unique-config-server-hostname>
```

## Point app at Config Server

http://steeltoe.io/docs/steeltoe-configuration/#2-2-2-configure-settings

> Only two settings are really necessary. `spring:application:name` configures the "application name" to be sample, and
`spring:cloud:config:uri` the address of the config server.


The application name is already configured in the `appsettings.json`, so we only need to override the config server
uri. When translated into environment variables, `spring:cloud:config:uri` turns into `spring__cloud__config__uri`. Note
the double underscores (__) which denote hierarchy. So to point a Steeltoe application to our own Config Server, we need
to set the environment in Cloud Foundry and restart the application.

```
cf set-env foo spring__cloud__config__uri https://<unique-config-server-hostname>.cfapps.io
cf restart foo
``` 

## Querying Config Server

Configuration can be queried from Config Server via combination of `application`, `profile`, and `label`.

- **label** - in the case of Config Server backed by git, this is a branch, tag, or commit sha
- **application** - name of application requesting the configuration
- **profile** - a grouping of configuration, which can be used to provide different configuration based on
environment such as "dev", "test", "qa", "production"

```no-highlight
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.yml
/{label}/{application}-{profile}.properties
```

Configuration can be accessed in the [YAML](https://en.wikipedia.org/wiki/YAML) or
[properties](https://en.wikipedia.org/wiki/.properties) format, as well as via a JSON payload that returns
configuration along with some metadata (i.e. matched application name, property source, etc).

### Examples

The examples below use Config Server deployed with a hostname of "my-config-server". Adjust the urls to match your own
deployment.

- Configuration in JSON format for app named `myapp`, with profile `cloud`, on master branch of the config repository:

    http://my-config-server.cfapps.io/myapp/cloud/master

- Configuration in YML format for app named `foo` with a profile of `cloud`, on a specific commit in the config
repository:

    http://my-config-server.cfapps.io/0f00ee87e4918ef124109980c1c3715462ec15db/foo-cloud.yml

- Configuration in YML format for app named `myapp` with a profile of `foobar`, living on the `master` branch of
the config repository:

    http://my-config-server.cfapps.io/master/myapp-foobar.yml

- Configuration in properties format for app named `myapp` with a profile `docker`, living on the `master`
branch of the config repository:

    http://my-config-server.cfapps.io/master/myapp-docker.properties

    Note how the values come from the
    "[docker](https://github.com/SteeltoeOSS/config-repo/blob/master/application.yml#L42-L55)" profile section in
    the `application.yml`. Also notice that `info.description`, `info.url`, and `eureka.client.serviceUrl.defaultZone`
    properties come from the top section of
    [application.yml](https://github.com/SteeltoeOSS/config-repo/blob/master/application.yml#L1-L7),
    which applies to *all* applications with *any* profile.

- Configuration in YML format for an app named `foo` with a profile of `development`:

    http://my-config-server.cfapps.io/foo-development.yml

    Look at the below url to better understand where the properties come from:

    http://my-config-server.cfapps.io/foo/development/master

    Notice how properties are aggregated from multiple sources:
    - [foo.properties](https://github.com/SteeltoeOSS/config-repo/blob/master/foo.properties) - applies to all
    `foo` apps
    - [foo-development.properties](https://github.com/SteeltoeOSS/config-repo/blob/master/foo-development.properties) -
    applies to `foo` apps with `development` profile
    - Top portion of [application.yml](https://github.com/SteeltoeOSS/config-repo/blob/master/application.yml#L1-L7) -
    applies to all applications regardless of name or profile
