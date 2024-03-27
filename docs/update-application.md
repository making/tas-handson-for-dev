Learn how to update applications. When redeploying an application using the `cf push` command, it follows the steps of stopping the old version and then starting the new version. During this time, the application cannot be accessed (404 error). To update applications without causing downtime, Tanzu Application Service offers the following methods:

* Blue-Green Update
* Rolling Update

## Blue-Green Deployment

Blue-Green Deployment is a method that allows both the old and new versions of an application to coexist, routing requests to both versions. After verifying the new version operates correctly, routing to the old version is removed. This approach is characterized by the ability to gradually transition to the new version while receiving feedback, and the ease of reverting to the old version if there are issues with the new one. In Blue-Green Deployment, the old and new versions are treated as separate, independent applications. With Tanzu Application Service, the ratio of requests to the old and new versions is proportional to the number of instances of each. Co-locating the old and new versions typically requires additional resources.

https://docs.cloudfoundry.org/devguide/deploy-apps/blue-green.html

Consider the currently running `hello-cf` app as Blue.

You can check the version of the Blue version using the Spring Boot Actuator Info endpoint:

```bash
HOST=$(cf curl /v2/apps/$(cf app hello-cf --guid)/routes | jq -r ".resources[0].entity.host")

curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```

```
0.0.2
```

Next, deploy the new version (Green). Here, change only the `INFO_VERSION` environment variable without modifying the application's source code. Modify your `manifest.yml` as follows:

```yaml
applications:
- name: hello-cf
  random-route: true
  instances: 1
  memory: 768m
  path: target/hello-cf-0.0.1-SNAPSHOT.jar
  buildpacks:
  - java_buildpack_offline
  services:
  - hello
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{jre: {version: 17.+}}'
    MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: info,health,env,prometheus
    MANAGEMENT_INFO_JAVA_ENABLED: true
    MANAGEMENT_INFO_OS_ENABLED: true
    MANAGEMENT_INFO_ENV_ENABLED: true
    INFO_VERSION: 0.0.3 # ⭐️
    INFO_MESSAGE: Hello World!
    API_KEY: ${vcap.services.hello.credentials.api-key}
```

Use this `manifest.yml` to push the application with a different name:

```
cf push hello-cf-green
```

Use the `cf apps` command to list the applications and verify that both applications are running:

```bash
cf apps
```

```
Getting apps in org handson-22297 / space demo as tmaki...

name             requested state   processes           routes
hello-cf         started           web:1/1, task:0/0   hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com
hello-cf-green   started           web:1/1, task:0/0   hello-cf-green-palm-capybara-yi.apps.dhaka.cf-app.com
```

At this stage, you're in the following state:

Retrieve the random Host part assigned to the hello-cf-green app with the following command:

```bash
HOST_NEW=$(cf curl /v2/apps/$(cf app hello-cf-green --guid)/routes | jq -r ".resources[0].entity.host")
```

Check the version of Green using the Spring Boot Actuator Info endpoint:

```bash
curl -s https://${HOST_NEW}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.3
```

Also, confirm that there is no change in the version of Blue at this stage:

```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.2
```

Currently, Blue and Green exist as completely separate applications. The URL for Green is not known to end-users, allowing for safe testing. Next, map the route for Blue, i.e., the URL used by end-users, to Green as well. This way, end-users will be routed to either the Blue or Green application.

Use the following `cf map-route` command to map the same route to the `hello-cf-green` app as `hello-cf`:

```bash
cf map-route hello-cf-green apps.dhaka.cf-app.com -n ${HOST}
```
```
Mapping route hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com to app hello-cf-green in org handson-22297 / space demo as tmaki...
OK
```

Use the `cf apps` command to list the applications and verify that two routes are mapped to the `hello-cf-green` app:

```bash
cf apps
```
```
Getting apps in org handson-22297 / space demo as tmaki...

name             requested state   processes           routes
hello-cf         started           web:1/1, task:0/0   hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com
hello-cf-green   started           web:1/1, task:0/0   hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com, hello-cf-green-palm-capybara-yi.apps.dhaka.cf-app.com
```

Access the URL originally mapped to Blue several times. The versions will return in a Round-Robin fashion:

```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.2
```
```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.3
```
```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.2
```
```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.3
```

Next, remove the route to Blue, ensuring end-users are only routed to Green.

Execute the following `cf unmap-route` command to unmap the original route from the `hello-cf` app:

```bash
cf unmap-route hello-cf apps.dhaka.cf-app.com -n ${HOST}
```
```
Removing route hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com from app hello-cf in org handson-22297 / space demo as tmaki...
OK
```

Use the `cf apps` command to list the applications and verify that no routes are mapped to the hello-cf app:

```bash
cf apps
```
```
Getting apps in org handson-22297 / space demo as tmaki...

name             requested state   processes           routes
hello-cf         started           web:1/1, task:0/0   
hello-cf-green   started           web:1/1, task:0/0   hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com, hello-cf-green-palm-capybara-yi.apps.dhaka.cf-app.com
```

Access the URL originally mapped to Blue several times. All responses will be the new version:

```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.3
```
```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.3
```
```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.3
```
```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.3
```

Once you've confirmed that 100% of requests are being routed to Green without issues, you can stop or delete Blue. If you wish for a quick rollback, opt to stop. Also, remove the new route mapped to Green as it's no longer needed.

Execute the following commands:

```
# Delete previous Blue if it exists
cf delete -f hello-cf-blue

# For quick rollback, rename hello-cf to hello-cf-blue
cf rename hello-cf hello-cf-blue

# Promote hello-cf-green to hello-cf
cf rename hello-cf-green hello-cf

# Delete (or Stop) hello-cf-blue
cf delete -f hello-cf-blue

# Unmap the new route mapped to Green
cf unmap-route hello-cf apps.dhaka.cf-app.com -n ${HOST_NEW}

# Delete the unmapped route
cf delete-route apps.dhaka.cf-app.com -n ${HOST_NEW} -f
```

Use the `cf apps` command to list the applications and verify that the original route is mapped to the hello-cf app. The display should look the same as before deploying Green:


```bash
cf apps
```
```
Getting apps in org handson-22297 / space demo as tmaki...

name       requested state   processes           routes
hello-cf   started           web:1/1, task:0/0   hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com
```

> Note: The above steps follow the documentation, but deploying using the following steps requires fewer steps:
> 
> 1. Rename the old app, adding a suffix `-venerable`.
>    ```
>    cf rename hello-cf hello-cf-venerable
>    ```
> 2. Deploy the new app using the old app's original name.
>    ```
>    cf push
>    ```
> 3. Map the original route to the new app (only if `random-route: true`).
>    ```
>    cf map-route hello-cf apps.dhaka.cf-app.com -n ${HOST}
>    ```
> 4. Delete or stop the old app.
>    ```
>    cf delete -f hello-cf-venerable
>    ```

## Rolling Update

Cloud Foundry supports updating applications through Rolling Updates. Unlike Blue-Green Updates, where a new application is deployed and routing is switched, this method updates the same application from the old version to the new one. If there are multiple instances, they are updated one at a time, so you only need one extra resource.

First, use the `cf scale` command to scale out the `hello-cf` app to 2 instances.

```bash
cf scale hello-cf -i 2
```

Next, deploy the new version. Here, you won't change the application's source code, only the environment variable INFO_VERSION. Please change the `manifest.yml` as follows.

```yaml
applications:
- name: hello-cf
  random-route: true
  instances: 2 # ⭐️
  memory: 768m
  path: target/hello-cf-0.0.1-SNAPSHOT.jar
  buildpacks:
  - java_buildpack_offline
  services:
  - hello
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{jre: {version: 17.+}}'
    MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: info,health,env,prometheus
    MANAGEMENT_INFO_JAVA_ENABLED: true
    MANAGEMENT_INFO_OS_ENABLED: true
    MANAGEMENT_INFO_ENV_ENABLED: true
    INFO_VERSION: 0.0.4 # ⭐️
    INFO_MESSAGE: Hello World!
    API_KEY: ${vcap.services.hello.credentials.api-key}
```

To perform a Rolling Update, add the `--strategy rolling` option to the `cf push` command.

```
cf push --strategy rolling
```

Open another terminal and run the following command to watch the update process.

```
watch cf app hello-cf

# Or

while true; do cf app hello-cf; sleep 1; done
```

Access the Info endpoint to check the version of the application running.

```
HOST=$(cf curl /v2/apps/$(cf app hello-cf --guid)/routes | jq -r ".resources[0].entity.host")

curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.4
```

The results of the `cf apps` command will change as follows during a Rolling Update:

1. Two instances of the old version are running.
  ```
  ...
  type:           web
  sidecars:       
  instances:      2/2
  memory usage:   768M
       state     since                  cpu     memory           disk           logging        details
  #0   running   2024-03-15T04:38:42Z   0.7%    139.9M of 768M   180.7M of 1G   0/s of 16K/s   
  #1   running   2024-03-15T04:47:58Z   11.7%   142.4M of 768M   180.7M of 1G   0/s of 16K/s   
  ...
  ```
1. While the two instances of the old version remain running, a new version instance starts to boot up.
  ```
  ...
  type:           web
  sidecars:       
  instances:      2/2
  memory usage:   768M
       state     since                  cpu    memory           disk           logging        details
  #0   running   2024-03-15T04:38:43Z   3.7%   153.5M of 768M   180.7M of 1G   0/s of 16K/s   
  #1   running   2024-03-15T04:47:58Z   6.8%   148.8M of 768M   180.7M of 1G   0/s of 16K/s   
  
  type:           web
  sidecars:       
  instances:      0/1
  memory usage:   768M
       state      since                  cpu    memory      disk      logging      details
  #0   starting   2024-03-15T04:49:16Z   0.0%   0 of 768M   0 of 1G   0/s of 0/s   
  ...
  ```
1. One instance of the new version becomes running, and another instance of the new version starts to boot up. One of the old version instances is removed.
  ```
  ...
  type:           web
  sidecars:       
  instances:      1/1
  memory usage:   768M
       state     since                  cpu    memory           disk           logging        details
  #0   running   2024-03-15T04:38:42Z   3.7%   153.5M of 768M   180.7M of 1G   0/s of 16K/s   
  
  type:           web
  sidecars:       
  instances:      1/2
  memory usage:   768M
       state      since                  cpu      memory           disk           logging           details
  #0   running    2024-03-15T04:49:31Z   118.0%   161.9M of 768M   180.7M of 1G   165B/s of 16K/s   
  #1   starting   2024-03-15T04:49:36Z   0.0%     0 of 0           0 of 0         0/s of 0/s        
  ...
  ```
1. Both instances of the new version are running. The old version instances are gone.
  ```
  ...
  type:           web
  sidecars:       
  instances:      2/2
  memory usage:   768M
       state     since                  cpu     memory           disk           logging          details
  #0   running   2024-03-15T04:49:32Z   4.5%    154M of 768M     180.7M of 1G   0/s of 16K/s     
  #1   running   2024-03-15T04:50:06Z   96.5%   109.8M of 768M   180.7M of 1G   81B/s of 16K/s   
  ...
  ```

You can also perform a Rolling Update for restarting the application with the following command.

```
cf restart --strategy rolling hello-cf
```

If you just want to apply changes in `manifest.yml`, you don't need to execute `cf push`. The following commands are faster and sufficient.

```
cf apply-manifest
cf restart --strategy rolling hello-cf
```

## Revisions

The history of updates made to the same application is managed as "Revisions." You can check the list of revisions for a specific app using the `cf revisions` command.

```
cf revisions hello-cf
```
```
This command is in EXPERIMENTAL stage and may change without notice

Getting revisions for app hello-cf in org handson-22297 / space demo as tmaki...

revision      description                                                 deployable   revision guid                          created at
2(deployed)   New droplet deployed. New environment variables deployed.   true         74783ff6-fc57-47e6-b98b-2a33ac4580b1   2024-03-15T04:49:14Z
1             Initial revision.                                           true         a52b646d-e360-4178-b14f-44795c91bafa   2024-03-15T04:38:24Z
```

You can also view the list of revisions from the Apps Manager.

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/dccaf9d8-82b0-4c28-b54f-1893728520c8">

Now, let's change the value of the environment variable `INFO_VERSION` to `0.4.0+r`.
```yaml
applications:
- name: hello-cf
  random-route: true
  instances: 2
  memory: 768m
  path: target/hello-cf-0.0.1-SNAPSHOT.jar
  buildpacks:
  - java_buildpack_offline
  services:
  - hello
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{jre: {version: 17.+}}'
    MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: info,health,env,prometheus
    MANAGEMENT_INFO_JAVA_ENABLED: true
    MANAGEMENT_INFO_OS_ENABLED: true
    MANAGEMENT_INFO_ENV_ENABLED: true
    INFO_VERSION: 0.0.4+r # ⭐️
    INFO_MESSAGE: Hello World!
    API_KEY: ${vcap.services.hello.credentials.api-key}
```

And perform a `cf push`. 

```
cf push --strategy rolling
```

After a while, the deployment will complete, and the value of the `version` key in the Info endpoint response will change to `0.4.0+r`.

```
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.4+r
```

Check the list of revisions again to confirm that a new revision has been added.

```
cf revisions hello-cf
```
```
This command is in EXPERIMENTAL stage and may change without notice

Getting revisions for app hello-cf in org handson-22297 / space demo as tmaki...

revision      description                                                 deployable   revision guid                          created at
3(deployed)   New droplet deployed. New environment variables deployed.   true         7719c408-6c32-4c0e-8e45-08ace58b2722   2024-03-15T06:01:26Z
2             New droplet deployed. New environment variables deployed.   true         74783ff6-fc57-47e6-b98b-2a33ac4580b1   2024-03-15T04:49:14Z
1             Initial revision.   
```

On the Apps Manager, you can view the list of environment variables for each revision.

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/b0c1421a-f340-46c8-871e-6c0a250d8df1">

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/7aaad849-32ba-485e-a52c-6832ec6f78f2">

You can also roll back to a specific revision. Let's try rolling back to revision version 2 with the following command.

```
cf rollback hello-cf --version 2 -f
```

Rollbacks can also be performed from the Apps Manager with the "REDEPLOY" button.

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/4f8f0ebd-2955-4577-996c-8228a98bc670">

After a while, the rollback will complete, and the value of the `version` key in the Info endpoint response will return to `0.4.0`.

```
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq -r .version
```
```
0.0.4
```


The rollback operation is also recorded as a new revision.



```
cf revisions hello-cf
```
```
This command is in EXPERIMENTAL stage and may change without notice

Getting revisions for app hello-cf in org handson-22297 / space demo as tmaki...

revision      description                                                                            deployable   revision guid                          created at
4(deployed)   New droplet deployed. New environment variables deployed. Rolled back to revision 2.   true         9b290c78-d18c-4936-b6d8-36e68ff228c2   2024-03-15T06:17:30Z
3             New droplet deployed. New environment variables deployed.                              true         7719c408-6c32-4c0e-8e45-08ace58b2722   2024-03-15T06:01:26Z
2             New droplet deployed. New environment variables deployed.                              true         74783ff6-fc57-47e6-b98b-2a33ac4580b1   2024-03-15T04:49:14Z
1             Initial revision.                                                                      true         a52b646d-e360-4178-b14f-44795c91bafa   2024-03-15T04:38:24Z
```



