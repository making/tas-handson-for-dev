Let's look at some ways to set properties on your app.

## Setting environment variables in applications

First, create a property (`api.key`) that can be changed with environment variables in the application, and make the application use that property.
Create the following new files.

`src/main/java/com/example/ApiProperties.java`
```java
package com.example;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "api")
public class ApiProperties {

    private String key = "SECRET";

    public String getKey() {
        return key;
    }

    public void setKey(String key) {
        this.key = key;
    }
}
```

`src/main/java/com/example/HelloController.java`

```java
package com.example;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    private final ApiProperties props;

    public HelloController(ApiProperties props) {
        this.props = props;
    }

    @GetMapping(path = "/")
    public ResponseEntity<?> hello(@RequestHeader(name = "X-Api-Key", required = false) String apiKey) {
        if (props.getKey().equals(apiKey)) {
            return ResponseEntity.ok("Hello");
        } else {
            return ResponseEntity.status(HttpStatus.FORBIDDEN).body("Forbidden");
        }
    }
}
```


First, hard-code the environment variable `API_KEY` in `manifest.ym`l as follows.

```yaml
applications:
- name: hello-cf
  random-route: true
  instances: 1
  memory: 768m
  path: target/hello-cf-0.0.1-SNAPSHOT.jar
  buildpacks: 
  - java_buildpack_offline
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{jre: {version: 17.+}}'
    MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: info,health,env,prometheus
    MANAGEMENT_INFO_JAVA_ENABLED: true
    MANAGEMENT_INFO_OS_ENABLED: true
    MANAGEMENT_INFO_ENV_ENABLED: true
    INFO_VERSION: 0.0.2 # ⭐️
    INFO_MESSAGE: Hello World!
    # ⭐️⭐️⭐️
    API_KEY: opensesami
```

Build the application and `cf push` again.


```bash
./mvnw clean package -Dmaven.test.skip=true
cf push
```

Access the app. `Hello` is returned only when HTTP request header `X-Api-Key` is `opensesami`, and `Forbidden` is returned otherwise.


```bash
HOST=$(cf curl /v2/apps/$(cf app hello-cf --guid)/routes | jq -r ".resources[0].entity.host")

curl -sv https://${HOST}.apps.dhaka.cf-app.com
```

```
> GET / HTTP/2
> Host: hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com
> User-Agent: curl/8.4.0
> Accept: */*
> 
< HTTP/2 403 
< content-type: text/plain;charset=UTF-8
< date: Wed, 06 Mar 2024 12:46:47 GMT
< x-vcap-request-id: 61878b33-4a18-41c1-576e-7a967140f58c
< content-length: 9
< 
Forbidden
```

```bash
curl -sv -H "X-Api-Key: opensesami" https://${HOST}.apps.dhaka.cf-app.com
```

```
> GET / HTTP/2
> Host: hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com
> User-Agent: curl/8.4.0
> Accept: */*
> X-Api-Key: opensesami
> 
< HTTP/2 200 
< content-type: text/plain;charset=UTF-8
< date: Wed, 06 Mar 2024 12:47:10 GMT
< x-vcap-request-id: 382e4bcc-c5c5-4b95-68c1-a35a6415a25a
< content-length: 5
< 
Hello
```

## Setting environment variables via User Provided Service

`manifest.yml` is usually managed by git. Hardcoding the API Key directly into a file managed by git is not secure. 
Save it on the Platform side instead of writing it in the file at hand.
First, use [User Provided Service](https://docs.vmware.com/en/VMware-Tanzu-Application-Service/5.0/tas-for-vms/services-user-provided.html). Create a `hello` service instance with the following command.

```bash
cf create-user-provided-service hello -p '{"api-key":"OPENSESAMI"}'
```

Edit `manifest.yml` to bind the `hell`o service instance to the `hello-cf` app. Also, set the value of the environment variable `API_KEY` to get `api-key` from the `hello` service instance.

```yaml
applications:
- name: hello-cf
  random-route: true
  instances: 1
  memory: 768m
  path: target/hello-cf-0.0.1-SNAPSHOT.jar
  buildpacks: 
  - java_buildpack_offline
  # ⭐️⭐️⭐️
  services:
  - hello
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{jre: {version: 17.+}}'
    MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: info,health,env,prometheus
    MANAGEMENT_INFO_JAVA_ENABLED: true
    MANAGEMENT_INFO_OS_ENABLED: true
    MANAGEMENT_INFO_ENV_ENABLED: true
    INFO_VERSION: 0.0.2 
    INFO_MESSAGE: Hello World!
    # ⭐️⭐️⭐️
    API_KEY: ${vcap.services.hello.credentials.api-key}
```

> Note: It is a feature of Spring Boot that you can get the value from the service instance in the format of `${vcap.services.<service instance name>.credentials.<key name>}`. When using other languages or frameworks, it is necessary to get the JSON string from the environment variable `VCAP_SERVICES` and parse it to get the value. Click [here](https://ik.am/entries/529) for a usage example.
> <br>
> Alternatively, you can reset the JSON string from the environment variable VCAP_SERVICES to a flat environment variable by using the Pancake Buildpack.<br>
> https://github.com/starkandwayne/pancake-buildpack


Redeploy with `cf push`.

```bash
cf push
```


This time, `Hello` is returned only when HTTP request header `X-Api-Key` is `OPENSESAMI`, and `Forbidden` is returned otherwise.

```bash
curl -sv -H "X-Api-Key: opensesami" https://${HOST}.apps.dhaka.cf-app.com
```
```
> GET / HTTP/2
> Host: hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com
> User-Agent: curl/8.4.0
> Accept: */*
> X-Api-Key: opensesami
> 
< HTTP/2 403 
< content-type: text/plain;charset=UTF-8
< date: Wed, 06 Mar 2024 12:58:08 GMT
< x-vcap-request-id: 8e2c250b-4241-4d01-6e77-c197f57fcdd5
< content-length: 9
< 
Forbidden
```

```bash
curl -sv -H "X-Api-Key: OPENSESAMI" https://${HOST}.apps.dhaka.cf-app.com
```
```
> GET / HTTP/2
> Host: hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com
> User-Agent: curl/8.4.0
> Accept: */*
> X-Api-Key: OPENSESAMI
> 
< HTTP/2 200 
< content-type: text/plain;charset=UTF-8
< date: Wed, 06 Mar 2024 12:58:15 GMT
< x-vcap-request-id: ca1d0aa8-2019-4753-5b01-17041dfbd693
< content-length: 5
< 
Hello
```

By using User Provided Service, it is not necessary to hard code API Key in `manifest.yml`, but the information stored in User Provided Service can be displayed, so it is not a secure location to store confidential information. The location is not secure. You can see the settings with the `cf env` command as follows: We will take advantage of the more secure CredHub Service Broker in the next section.

```bash
cf env hello-cf
```

```
Getting env variables for app hello-cf in org handson-22297 / space demo as tmaki...
System-Provided:
VCAP_SERVICES: {
  "user-provided": [
    {
      "binding_guid": "824f8b90-0a7b-417b-9631-b757ad386c5e",
      "binding_name": null,
      "credentials": {
        "api-key": "OPENSESAMI" ⭐️⭐⭐
      },
      "instance_guid": "d2fd30ce-3c6a-4180-a31d-0cb6cfa31d2a",
      "instance_name": "hello",
      "label": "user-provided",
      "name": "hello",
      "syslog_drain_url": null,
      "tags": [],
      "volume_mounts": []
    }
  ]
}


VCAP_APPLICATION: {
  "application_id": "050f7c76-46da-45ce-96d0-05fcc7bd7e6a",
  "application_name": "hello-cf",
  "application_uris": [
    "hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com"
  ],
  "cf_api": "https://api.sys.dhaka.cf-app.com",
  "limits": {
    "fds": 16384
  },
  "name": "hello-cf",
  "organization_id": "375a40e6-6813-4f60-b0ed-2ad47e60a258",
  "organization_name": "handson-22297",
  "space_id": "04bdfef4-6b76-4a40-a6c1-b7f1967170ab",
  "space_name": "demo",
  "uris": [
    "hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com"
  ],
  "users": null
}

...
```

## Store sensitive information for apps using CredHub Service Broker

CredHub is a good place to store sensitive information. By using CredHub Service Broker instead of User Provided Service, confidential information can be registered in CredHub by `cf` command, and it can be handled like User Provided Service.

First, use the following command to unbind the `hello` service instance created earlier from the `hello-cf` application and delete the `hello` service instance.

```bash
cf unbind-service hello-cf hello
cf delete-service -f hello
```

Make sure the Credhub Service is present in the Marketplace with the following command.

```bash
cf marketplace
```
```
Getting all service offerings from marketplace in org handson-22297 / space demo as tmaki...

offering                    plans                                                   description                                                                                                                                                                                                                   broker
app-autoscaler              standard                                                Scales bound applications in response to load                                                                                                                                                                                 app-autoscaler
️credhub                     default                                                 Stores configuration parameters securely in CredHub                                                                                                                                                                           credhub-broker ⭐️⭐️⭐
smb                         Existing                                                Existing SMB shares (see: https://code.cloudfoundry.org/smb-volume-release/)                                                                                                                                                  smbbroker
...

TIP: Use 'cf marketplace -e SERVICE_OFFERING' to view descriptions of individual plans of a given service offering.
```


Check the CredHub Service Plan with the following command:

```bash
cf marketplace -e credhub
```

```
Getting service plan information for service offering credhub in org handson-22297 / space demo as tmaki...

broker: credhub-broker
   plan      description                                           free or paid   costs
   default   Stores configuration parameters securely in CredHub   free
```

Create a `hello` service instance with the following command.

```bash
cf create-service credhub default hello -c '{"api-key": "OpenSesami"}'
```

Rebind & restart or push again with the following command.

```bash
cf bind-service hello-cf hello
cf restart hello-cf

# OR

cf push
```


Make sure you can access it as you would for the User Provided Service.

```bash
curl -sv -H "X-Api-Key: OPENSESAMI" https://${HOST}.apps.dhaka.cf-app.com
```

```
> GET / HTTP/2
> Host: hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com
> User-Agent: curl/8.4.0
> Accept: */*
> X-Api-Key: OPENSESAMI
> 
< HTTP/2 403 
< content-type: text/plain;charset=UTF-8
< date: Wed, 06 Mar 2024 13:12:32 GMT
< x-vcap-request-id: e3552584-a31e-48a2-6d2d-93e07cc9fe8c
< content-length: 9
< 
Forbidden
```


```bash
curl -sv -H "X-Api-Key: OpenSesami" https://${HOST}.apps.dhaka.cf-app.com
```

```
> GET / HTTP/2
> Host: hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com
> User-Agent: curl/8.4.0
> Accept: */*
> X-Api-Key: OpenSesami
> 
< HTTP/2 200 
< content-type: text/plain;charset=UTF-8
< date: Wed, 06 Mar 2024 13:12:44 GMT
< x-vcap-request-id: 0acf6a85-e078-4b7c-42e5-5a1b2d444201
< content-length: 5
< 
Hello
```


Make sure that the API Key value is not displayed when you execute cf env.

```bash
cf env hello-cf
```
```
Getting env variables for app hello-cf in org handson-22297 / space demo as tmaki...
System-Provided:
VCAP_SERVICES: {
  "credhub": [
    {
      "binding_guid": "8f0d8440-f569-4f62-bc88-ee31d7cd6f20",
      "binding_name": null,
      "credentials": {
        "credhub-ref": "/credhub-service-broker/credhub/771a3387-4d43-40c7-bc68-77fdf78598d0/credentials" ⭐️⭐⭐
      },
      "instance_guid": "771a3387-4d43-40c7-bc68-77fdf78598d0",
      "instance_name": "hello",
      "label": "credhub",
      "name": "hello",
      "plan": "default",
      "provider": null,
      "syslog_drain_url": null,
      "tags": [
        "credhub"
      ],
      "volume_mounts": []
    }
  ]
}


VCAP_APPLICATION: {
  "application_id": "050f7c76-46da-45ce-96d0-05fcc7bd7e6a",
  "application_name": "hello-cf",
  "application_uris": [
    "hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com"
  ],
  "cf_api": "https://api.sys.dhaka.cf-app.com",
  "limits": {
    "fds": 16384
  },
  "name": "hello-cf",
  "organization_id": "375a40e6-6813-4f60-b0ed-2ad47e60a258",
  "organization_name": "handson-22297",
  "space_id": "04bdfef4-6b76-4a40-a6c1-b7f1967170ab",
  "space_name": "demo",
  "uris": [
    "hello-cf-quiet-dugong-mi.apps.dhaka.cf-app.com"
  ],
  "users": null
}
...
```