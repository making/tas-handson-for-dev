## Creating a template application with Spring Initializr


Create a Spring Boot template application with [Spring Initializr](https://start.spring.io) and deploy it immediately with `cf push`.
Create a `handson` directory as a working place.

```bash
mkdir handson
cd handson
```

Create a template application with the following command.

```bash
curl https://start.spring.io/starter.tgz \
    -d artifactId=hello-cf \
    -d baseDir=hello-cf \
    -d type=maven-project \
    -d dependencies=web,actuator,configuration-processor,prometheus \
    -d packageName=com.example \
    -d applicationName=HelloCfApplication | tar -xzvf -
```

Build the application with the following command:

```bash
cd hello-cf
./mvnw clean package -Dmaven.test.skip=true
```


## Deploy with `cf push`

Create `manifest.yml` under the `hello-cf` directory and describe the following contents.

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
```

> Note: In Hands-on environment, multiple people deploy an application with the same name. By default, the Host part of the application's URL will be the same as the name, which causes a URL collision. To avoid this, add `random-route: true` and add a random character string to the Host part.


Redeploy the application with `cf push`.

```bash
cf push
```

Access the app deployed with the following command.

```bash
# Get the Host part that contains a random string and set it in a variable
HOST=$(cf curl /v2/apps/$(cf app hello-cf --guid)/routes | jq -r ".resources[0].entity.host")

# Access Spring Boot Acutator Health Endpoint
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/health -w '\n'
```

You'll see the below output

```json
{"status":"UP"}
```

## Expose Spring Boot Actuator endpoints

[Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html) has many useful endpoints for operation. By default, only `/actuator/health` is exposed. Let's expose the `/actutor/info`, `/actuator/env` and `/actuator/prometheus` by setting the following environment variables in `manifest.yml`:

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
    # ⭐️⭐️⭐️
    MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: info,health,env,prometheus
    MANAGEMENT_INFO_JAVA_ENABLED: true
    MANAGEMENT_INFO_OS_ENABLED: true
```


Redeploy the application with cf push.

```bash
cf push
```

Access the info endpoint.

```
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq .
```
```json
{
  "java": {
    "version": "17.0.9",
    "vendor": {
      "name": "BellSoft"
    },
    "runtime": {
      "name": "OpenJDK Runtime Environment",
      "version": "17.0.9+11-LTS"
    },
    "jvm": {
      "name": "OpenJDK 64-Bit Server VM",
      "vendor": "BellSoft",
      "version": "17.0.9+11-LTS"
    }
  },
  "os": {
    "name": "Linux",
    "version": "6.5.0-15-generic",
    "arch": "amd64"
  }
}
```

Access the env endpoint.

```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/env | jq .
```

```json
{
  "activeProfiles": [
    "cloud"
  ],
  "propertySources": [
    {
      "name": "server.ports",
      "properties": {
        "local.server.port": {
          "value": "******"
        }
      }
    },
    {
      "name": "vcap",
      "properties": {
        "vcap.application.process_type": {
          "value": "******"
        },
        "vcap.application.application_uris": {
          "value": "******"
        },
        "vcap.application.space_name": {
          "value": "******"
        },
...
        "HOME": {
          "value": "******",
          "origin": "System Environment Property \"HOME\""
        },
        "MALLOC_ARENA_MAX": {
          "value": "******",
          "origin": "System Environment Property \"MALLOC_ARENA_MAX\""
        }
      }
    }
  ]
}
```

> Note: The values inside the response of the env endpoint are masked by default.

```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/prometheus 
```

Access the prometheus endpoint.

```
# HELP jvm_threads_live_threads The current number of live threads including both daemon and non-daemon threads
# TYPE jvm_threads_live_threads gauge
jvm_threads_live_threads 21.0
# HELP executor_active_threads The approximate number of threads that are actively executing tasks
# TYPE executor_active_threads gauge
executor_active_threads{name="applicationTaskExecutor",} 0.0
# HELP process_files_open_files The open file descriptor count
# TYPE process_files_open_files gauge
process_files_open_files 56.0
# HELP executor_completed_tasks_total The approximate total number of tasks that have completed execution
# TYPE executor_completed_tasks_total counter
executor_completed_tasks_total{name="applicationTaskExecutor",} 0.0
# HELP jvm_memory_used_bytes The amount of used memory
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{area="heap",id="Tenured Gen",} 1.6711832E7
jvm_memory_used_bytes{area="nonheap",id="CodeHeap 'profiled nmethods'",} 9542656.0
jvm_memory_used_bytes{area="heap",id="Eden Space",} 1.0675856E7
jvm_memory_used_bytes{area="nonheap",id="Metaspace",} 3.6345472E7
jvm_memory_used_bytes{area="nonheap",id="CodeHeap 'non-nmethods'",} 1265920.0
jvm_memory_used_bytes{area="heap",id="Survivor Space",} 1355856.0
jvm_memory_used_bytes{area="nonheap",id="Compressed Class Space",} 4847896.0
jvm_memory_used_bytes{area="nonheap",id="CodeHeap 'non-profiled nmethods'",} 2567424.0
# HELP disk_free_bytes Usable space for path
# TYPE disk_free_bytes gauge
disk_free_bytes{path="/home/vcap/app/.",} 8.8434688E8
# HELP jvm_threads_daemon_threads The current number of live daemon threads
# TYPE jvm_threads_daemon_threads gauge
jvm_threads_daemon_threads 17.0
...
```

## Utilization of Info endpoint

The Info endpoint outputs properties or starting with `info.` or `INFO_` in JSON format. It is convenient to include the version of the running application.

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
    # ⭐️⭐️⭐️
    MANAGEMENT_INFO_ENV_ENABLED: true
    INFO_VERSION: 0.0.1
    INFO_MESSAGE: Hello World!
```

Redeploy the application with `cf push`.

```bash
cf push
```


Access the info endpoint.

```bash
curl -s https://${HOST}.apps.dhaka.cf-app.com/actuator/info | jq .
```

```json
{
  "message": "Hello World!",
  "version": "0.0.1",
  "java": {
    "version": "17.0.9",
    "vendor": {
      "name": "BellSoft"
    },
    "runtime": {
      "name": "OpenJDK Runtime Environment",
      "version": "17.0.9+11-LTS"
    },
    "jvm": {
      "name": "OpenJDK 64-Bit Server VM",
      "vendor": "BellSoft",
      "version": "17.0.9+11-LTS"
    }
  },
  "os": {
    "name": "Linux",
    "version": "6.5.0-15-generic",
    "arch": "amd64"
  }
}
```

In the JSON of the response, you will see that it contains the keys `message` and `version`.