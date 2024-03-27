To automatically provision instances of databases that applications can use, you can utilize a Service Broker. 
In this hands-on, we will use [VMware Postgres for VMware Tanzu Application Service](https://network.tanzu.vmware.com/products/vmware-postgres-for-tas/).
This allows you to create a dedicated database using only the `cf` command.

You can check whether this service is available using the `cf marketplace` command.

```
cf marketplace
```

If the output contains lines similar to the following, VMware Postgres for VMware Tanzu Application Service is available.
The plan name (here `on-demand-postgres-small `) can be different.

```
...
postgres                    on-demand-postgres-small                                Postgres service to provide on-demand dedicated instances configured as database.                                                                                                                                             postgres-odb
...
```

## Update source code


First, update the source code to access PostgreSQL. Add the following three `<dependency>` within `<dependencies>` of `pom.xml` under the `hello-cf` directory.

`pom.xml`

```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
      <groupId>org.flywaydb</groupId>
      <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
      <scope>runtime</scope>
    </dependency>
```

`src/main/java/com/example/VehicleController.java`

```java
package com.example;

import java.util.List;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.jdbc.core.simple.JdbcClient;
import org.springframework.jdbc.support.GeneratedKeyHolder;
import org.springframework.jdbc.support.KeyHolder;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class VehicleController {

	private final JdbcClient jdbcClient;

	public VehicleController(JdbcClient jdbcClient) {
		this.jdbcClient = jdbcClient;
	}

	@GetMapping(path = "/vehicles")
	public ResponseEntity<?> getVehicles() {
		List<Vehicle> vehicles = this.jdbcClient.sql("""
						SELECT id, name FROM vehicle ORDER BY id
						""")
				.query(Vehicle.class)
				.list();
		return ResponseEntity.ok(vehicles);
	}

	@PostMapping(path = "/vehicles")
	public ResponseEntity<?> postVehicles(@RequestBody Vehicle vehicle) {
		KeyHolder keyHolder = new GeneratedKeyHolder();
		this.jdbcClient.sql("""
						INSERT INTO vehicle(name) VALUES (?)
						""")
				.param(vehicle.name())
				.update(keyHolder, "id");
		Vehicle created = new Vehicle(keyHolder.getKey().intValue(), vehicle.name());
		return ResponseEntity.status(HttpStatus.CREATED).body(created);
	}

	@DeleteMapping(path = "/vehicles/{id}")
	public ResponseEntity<?> deleteVehicle(@PathVariable("id") Integer id) {
		this.jdbcClient.sql("""
						DELETE FROM vehicle WHERE id = ?
						""").param(id)
				.update();
		return ResponseEntity.noContent().build();
	}

	record Vehicle(Integer id, String name) {

	}
}
```

`src/main/resources/db/migration/V1__init.sql`

```sql
CREATE TABLE vehicle
(
    id   SERIAL PRIMARY KEY,
    name VARCHAR(16)
);

INSERT INTO vehicle(name)
VALUES ('Avalon');
INSERT INTO vehicle(name)
VALUES ('Corolla');
INSERT INTO vehicle(name)
VALUES ('Crown');
INSERT INTO vehicle(name)
VALUES ('Levin');
INSERT INTO vehicle(name)
VALUES ('Yaris');
INSERT INTO vehicle(name)
VALUES ('Vios');
INSERT INTO vehicle(name)
VALUES ('Glanza');
INSERT INTO vehicle(name)
VALUES ('Aygo');
```


`application.properties`

```properties
spring.datasource.driver-class-name=org.postgresql.Driver
# Local用のダミー設定
spring.datasource.url=jdbc:postgresql://localhost:5432/car
spring.datasource.username=${USER}
spring.datasource.password=

```


```
./mvnw clean package -Dmaven.test.skip=true
```

## Create a service instance


```
cf create-service postgres on-demand-postgres-small vehicle-db
```
```
Creating service instance vehicle-db in org handson-22297 / space demo as tmaki...

Create in progress. Use 'cf services' or 'cf service vehicle-db' to check operation status.
OK
```

```
cf service vehicle-db
```

```
Showing info of service vehicle-db in org handson-22297 / space demo as tmaki...

name:            vehicle-db
guid:            0ca24400-500b-4227-a964-9ec278327fc2
type:            managed
broker:          postgres-odb
offering:        postgres
plan:            on-demand-postgres-small
tags:            
offering tags:   postgres, pivotal, on-demand
description:     Postgres service to provide on-demand dedicated instances configured as database.
documentation:   
dashboard url:   

Showing status of last operation:
   status:    create succeeded
   message:   Instance provisioning completed
   started:   2024-03-15T09:27:20Z
   updated:   2024-03-15T09:27:20Z

...
```

## Deploy the application

```yaml
applications:
- name: hello-cf
  random-route: true
  instances: 1 # ⭐️
  memory: 768m
  path: target/hello-cf-0.0.1-SNAPSHOT.jar
  buildpacks: 
  - java_buildpack_offline
  services:
  - hello
  - vehicle-db # ⭐️
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{jre: {version: 17.+}}'
    MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: info,health,env,prometheus
    MANAGEMENT_INFO_JAVA_ENABLED: true
    MANAGEMENT_INFO_OS_ENABLED: true
    MANAGEMENT_INFO_ENV_ENABLED: true
    INFO_VERSION: 0.0.5 # ⭐️
    INFO_MESSAGE: Hello World!
    API_KEY: ${vcap.services.hello.credentials.api-key}
```

```
cf push --strategy rolling
```

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/74ae2665-18a1-457c-a038-588334d1dfd1">


```bash
HOST=$(cf curl /v2/apps/$(cf app hello-cf --guid)/routes | jq -r ".resources[0].entity.host")

curl -s https://${HOST}.apps.dhaka.cf-app.com/vehicles | jq .
```

```json
[
  {
    "id": 1,
    "name": "Avalon"
  },
  {
    "id": 2,
    "name": "Corolla"
  },
  {
    "id": 3,
    "name": "Crown"
  },
  {
    "id": 4,
    "name": "Levin"
  },
  {
    "id": 5,
    "name": "Yaris"
  },
  {
    "id": 6,
    "name": "Vios"
  },
  {
    "id": 7,
    "name": "Glanza"
  },
  {
    "id": 8,
    "name": "Aygo"
  }
]
```

```
curl -s https://${HOST}.apps.dhaka.cf-app.com/vehicles -d "{\"name\": \"Lexus\"}" -H "Content-Type: application/json" | jq .
```
```json
{
  "id": 9,
  "name": "Lexus"
}
```

```
curl -s https://${HOST}.apps.dhaka.cf-app.com/vehicles/1 -XDELETE
```

```
curl -s https://${HOST}.apps.dhaka.cf-app.com/vehicles | jq .
```
```json
[
  {
    "id": 2,
    "name": "Corolla"
  },
  {
    "id": 3,
    "name": "Crown"
  },
  {
    "id": 4,
    "name": "Levin"
  },
  {
    "id": 5,
    "name": "Yaris"
  },
  {
    "id": 6,
    "name": "Vios"
  },
  {
    "id": 7,
    "name": "Glanza"
  },
  {
    "id": 8,
    "name": "Aygo"
  },
  {
    "id": 9,
    "name": "Lexus"
  }
]
```

```
$ cf env hello-cf
Getting env variables for app hello-cf in org handson-22297 / space demo as tmaki...
System-Provided:
VCAP_SERVICES: {
  "credhub": [
    ...
  ],
  "postgres": [
    {
      "binding_guid": "298392bb-363f-46d4-9f6a-2752cdd745f1",
      "binding_name": null,
      "credentials": {
        "db": "postgres",
        "hosts": [
          "10.0.8.71"
        ],
        "jdbcUrl": "jdbc:postgresql://10.0.8.71:5432/postgres?user=pgadmin&password=205AHMsxJ943u71p68wL",
        "password": "205AHMsxJ943u71p68wL",
        "port": 5432,
        "service_gateway_access_port": 0,
        "service_gateway_enabled": false,
        "uri": "postgresql://pgadmin:205AHMsxJ943u71p68wL@10.0.8.71:5432/postgres",
        "user": "pgadmin"
      },
      "instance_guid": "0ca24400-500b-4227-a964-9ec278327fc2",
      "instance_name": "vehicle-db",
      "label": "postgres",
      "name": "vehicle-db",
      "plan": "on-demand-postgres-small",
      "provider": null,
      "syslog_drain_url": null,
      "tags": [
        "postgres",
        "pivotal",
        "on-demand"
      ],
      "volume_mounts": []
    }
  ]
}

...
```

```
   Downloaded app package (22.1M)
   -----> Java Buildpack v4.66.0 (offline) | https://github.com/cloudfoundry/java-buildpack#9e8f9bec
   ...
   -----> Downloading Java Cf Env 3.1.5 from https://java-buildpack.cloudfoundry.org/java-cfenv/java-cfenv-3.1.5.jar (found in cache)
   Exit status 0
```


```
cf logs hello-cf --recent 
```

```
   2024-03-15T18:34:10.42+0900 [APP/PROC/WEB/1] OUT 2024-03-15T09:34:10.422Z  INFO 25 --- [           main] s.b.CfDataSourceEnvironmentPostProcessor : Setting spring.datasource properties from bound service [vehicle-db]
   2024-03-15T18:34:10.42+0900 [APP/PROC/WEB/1] OUT 2024-03-15T09:34:10.422Z  INFO 25 --- [           main] i.p.c.s.boot.CfEnvironmentPostProcessor  : Setting null properties from bound service [hello] using io.pivotal.cfenv.spring.boot.CredHubCfEnvProcessor
```

`postgresql://pgadmin:205AHMsxJ943u71p68wL@10.0.8.71:5432/postgres`


```
cf ssh hello-cf -L 5432:10.0.8.71:5432
```

```
psql postgresql://pgadmin:205AHMsxJ943u71p68wL@127.0.0.1:5432/postgres
```
