Tanzu Application Service can run not only resident applications (Long Running Processes) such as web applications, but also short-lived tasks. 
Execute batch processing using the [Task](https://docs.cloudfoundry.org/devguide/using-tasks.html) function. Deploy simple batch processing using [Spring Batch](https://spring.io/projects/spring-batch).


## Execute a task

Let's run a simple command with the `cf run-task` command.

```
cf run-task hello-cf -c "ls -la"
```


## Create a Spring Batch application

```
cd ..

curl https://start.spring.io/starter.tgz \
    -d artifactId=billing-job \
    -d baseDir=billing-job \
    -d type=maven-project \
    -d packageName=com.example \
    -d dependencies=batch,postgresql,configuration-processor \
    -d applicationName=BillingJobApplication | tar -xzvf -

cd billing-job
```

```java
package com.example;

public record Usage(Long id, String firstName, String lastName, Long dataUsage, Long minutes) {
}
```

```java
package com.example;

import java.math.BigDecimal;
import java.math.RoundingMode;

public record Bill(Long id, String firstName, String lastName, Long dataUsage, Long minutes, BigDecimal billAmount) {


	public static BigDecimal calcBillAmount(Long dataUsage, Long minutes) {
		// dataUsage * 0.001 + usageMinutes * 0.01
		return BigDecimal.valueOf(dataUsage).multiply(new BigDecimal("0.001"))
				.add(BigDecimal.valueOf(minutes).multiply(new BigDecimal("0.01")))
				.setScale(2, RoundingMode.FLOOR);
	}

	public static Bill fromUsage(Usage usage) {
		BigDecimal billAmount = calcBillAmount(usage.dataUsage(), usage.minutes());
		return new Bill(usage.id(), usage.firstName(), usage.lastName(), usage.dataUsage(), usage.minutes(), billAmount);
	}

}
```

```java
package com.example;

import javax.sql.DataSource;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.batch.item.database.builder.JdbcBatchItemWriterBuilder;
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.builder.FlatFileItemReaderBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.Resource;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class JobConfig {
	@Bean
	@StepScope
	public FlatFileItemReader<Usage> usageItemReader(@Value("#{jobParameters['usageInfoFile']}") Resource usageInfoFile) {
		return new FlatFileItemReaderBuilder<Usage>()
				.name("UsageItemReader")
				.resource(usageInfoFile)
				.delimited()
				.names("id", "firstName", "lastName", "minutes", "dataUsage")
				.fieldSetMapper(fs -> new Usage(fs.readLong("id"),
						fs.readString("firstName"),
						fs.readString("lastName"),
						fs.readLong("minutes"),
						fs.readLong("dataUsage")))
				.linesToSkip(1)
				.build();
	}

	@Bean
	public ItemProcessor<Usage, Bill> usageToBillItemProcessor() {
		return new ItemProcessor<Usage, Bill>() {
			@Override
			public Bill process(Usage usage) throws Exception {
				return Bill.fromUsage(usage);
			}
		};
	}

	@Bean
	public ItemWriter<Bill> billItemWriter(DataSource dataSource) {
		return new JdbcBatchItemWriterBuilder<Bill>()
				.beanMapped()
				.dataSource(dataSource)
				.sql("INSERT INTO BILL_STATEMENTS (id, first_name, last_name, minutes, data_usage,bill_amount) VALUES (:id, :firstName, :lastName, :minutes, :dataUsage, :billAmount)")
				.build();
	}

	@Bean
	public Step billingStep(JobRepository jobRepository, PlatformTransactionManager transactionManager, ItemReader<Usage> itemReader, ItemProcessor<Usage, Bill> itemProcessor, ItemWriter<Bill> itemWriter) {
		return new StepBuilder("BillingStep", jobRepository)
				.<Usage, Bill>chunk(1000, transactionManager)
				.reader(itemReader)
				.processor(itemProcessor)
				.writer(itemWriter)
				.build();
	}

	@Bean
	public Job billingJob(JobRepository jobRepository, Step billingStep) {
		return new JobBuilder("BillingJob", jobRepository)
				.incrementer(new RunIdIncrementer())
				.start(billingStep)
				.build();
	}
}
```

```
./mvnw clean package -Dmaven.test.skip=true
```

```
cf create-service postgres on-demand-postgres-small billing-db
```

```yaml
applications:
- name: billing-job
  no-route: true
  instances: 0
  path: target/billing-job-0.0.1-SNAPSHOT.jar
  buildpacks: 
  - java_buildpack_offline
  services:
  - billing-db
  env:
    JBP_CONFIG_OPEN_JDK_JRE: '{jre: {version: 17.+}}'
```

```
cf push
```


```
cf logs billing-job 
```

```
cf run-task billing-job -m 128m -c ".java-buildpack/open_jdk_jre/bin/java org.springframework.boot.loader.launch.JarLauncher usageInfoFile=classpath:usageinfo.csv"
```


```
   2024-04-10T18:10:46.04+0900 [APP/TASK/530028ad/0] OUT .   ____          _            __ _ _
   2024-04-10T18:10:46.04+0900 [APP/TASK/530028ad/0] OUT /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
   2024-04-10T18:10:46.04+0900 [APP/TASK/530028ad/0] OUT ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
   2024-04-10T18:10:46.04+0900 [APP/TASK/530028ad/0] OUT \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
   2024-04-10T18:10:46.04+0900 [APP/TASK/530028ad/0] OUT '  |____| .__|_| |_|_| |_\__, | / / / /
   2024-04-10T18:10:46.04+0900 [APP/TASK/530028ad/0] OUT =========|_|==============|___/=/_/_/_/
   2024-04-10T18:10:46.04+0900 [APP/TASK/530028ad/0] OUT :: Spring Boot ::                (v3.2.4)
   2024-04-10T18:10:46.18+0900 [APP/TASK/530028ad/0] OUT 2024-04-10T09:10:46.185Z  INFO 7 --- [demo] [           main] com.example.BillingJobApplication        : Starting BillingJobApplication v0.0.1-SNAPSHOT using Java 17.0.10 with PID 7 (/home/vcap/app/BOOT-INF/classes started by vcap in /home/vcap/app)
   2024-04-10T18:10:46.19+0900 [APP/TASK/530028ad/0] OUT 2024-04-10T09:10:46.190Z  INFO 7 --- [demo] [           main] com.example.BillingJobApplication        : The following 1 profile is active: "cloud"
    ....
   2024-04-10T18:10:49.50+0900 [APP/TASK/530028ad/0] OUT 2024-04-10T09:10:49.500Z  INFO 7 --- [demo] [           main] com.example.BillingJobApplication        : Started BillingJobApplication in 4.261 seconds (process running for 4.93)
   2024-04-10T18:10:49.67+0900 [APP/TASK/530028ad/0] OUT 2024-04-10T09:10:49.670Z  INFO 7 --- [demo] [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=BillingJob]] launched with the following parameters: [{'run.id':'{value=1, type=class java.lang.Long, identifying=true}','usageInfoFile':'{value=classpath:usageinfo.csv, type=class java.lang.String, identifying=true}'}]
   2024-04-10T18:10:49.73+0900 [APP/TASK/530028ad/0] OUT 2024-04-10T09:10:49.737Z  INFO 7 --- [demo] [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [BillingStep]
   2024-04-10T18:10:49.87+0900 [APP/TASK/530028ad/0] OUT 2024-04-10T09:10:49.871Z DEBUG 7 --- [demo] [           main] o.s.batch.core.step.tasklet.TaskletStep  : Applying contribution: [StepContribution: read=5, written=5, filtered=0, readSkips=0, writeSkips=0, processSkips=0, exitStatus=EXECUTING]
   2024-04-10T18:10:49.87+0900 [APP/TASK/530028ad/0] OUT 2024-04-10T09:10:49.876Z DEBUG 7 --- [demo] [           main] o.s.batch.core.step.tasklet.TaskletStep  : Saving step execution before commit: StepExecution: id=1, version=1, name=BillingStep, status=STARTED, exitStatus=EXECUTING, readCount=5, filterCount=0, writeCount=5 readSkipCount=0, writeSkipCount=0, processSkipCount=0, commitCount=1, rollbackCount=0, exitDescription=
   2024-04-10T18:10:49.89+0900 [APP/TASK/530028ad/0] OUT 2024-04-10T09:10:49.890Z  INFO 7 --- [demo] [           main] o.s.batch.core.step.AbstractStep         : Step: [BillingStep] executed in 151ms
   2024-04-10T18:10:49.91+0900 [APP/TASK/530028ad/0] OUT 2024-04-10T09:10:49.916Z  INFO 7 --- [demo] [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=BillingJob]] completed with the following parameters: [{'run.id':'{value=1, type=class java.lang.Long, identifying=true}','usageInfoFile':'{value=classpath:usageinfo.csv, type=class java.lang.String, identifying=true}'}] and the following status: [COMPLETED] in 213ms
```

You can see from the log that 5 items of data were processed.

> Note: If `Exit status 137 (out of memory)` occurs, please change `-m 128m` to `-m 256m` and re-run.


```
cf run-task billing-job -m 128m -c ".java-buildpack/open_jdk_jre/bin/java org.springframework.boot.loader.launch.JarLauncher usageInfoFile=https://github.com/making/fakedata/raw/master/usageinfo/usageinfo-10000-en.csv"
```


```
   2024-04-10T19:32:29.58+0900 [APP/TASK/9d230a5e/0] OUT .   ____          _            __ _ _
   2024-04-10T19:32:29.58+0900 [APP/TASK/9d230a5e/0] OUT /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
   2024-04-10T19:32:29.58+0900 [APP/TASK/9d230a5e/0] OUT ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
   2024-04-10T19:32:29.58+0900 [APP/TASK/9d230a5e/0] OUT \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
   2024-04-10T19:32:29.58+0900 [APP/TASK/9d230a5e/0] OUT '  |____| .__|_| |_|_| |_\__, | / / / /
   2024-04-10T19:32:29.58+0900 [APP/TASK/9d230a5e/0] OUT =========|_|==============|___/=/_/_/_/
   2024-04-10T19:32:29.58+0900 [APP/TASK/9d230a5e/0] OUT :: Spring Boot ::                (v3.2.4)
   2024-04-10T19:32:29.67+0900 [APP/TASK/9d230a5e/0] OUT 2024-04-10T10:32:29.676Z  INFO 6 --- [demo] [           main] com.example.BillingJobApplication        : Starting BillingJobApplication v0.0.1-SNAPSHOT using Java 17.0.10 with PID 6 (/home/vcap/app/BOOT-INF/classes started by vcap in /home/vcap/app)
   2024-04-10T19:32:29.68+0900 [APP/TASK/9d230a5e/0] OUT 2024-04-10T10:32:29.680Z  INFO 6 --- [demo] [           main] com.example.BillingJobApplication        : The following 1 profile is active: "cloud"
   ...
   2024-04-10T19:32:32.17+0900 [APP/TASK/9d230a5e/0] OUT 2024-04-10T10:32:32.171Z  INFO 6 --- [demo] [           main] com.example.BillingJobApplication        : Started BillingJobApplication in 3.171 seconds (process running for 3.754)
   2024-04-10T19:32:32.33+0900 [APP/TASK/9d230a5e/0] OUT 2024-04-10T10:32:32.337Z  INFO 6 --- [demo] [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=BillingJob]] launched with the following parameters: [{'run.id':'{value=3, type=class java.lang.Long, identifying=true}','usageInfoFile':'{value=https://github.com/making/fakedata/raw/master/usageinfo/usageinfo-10000-en.csv, type=class java.lang.String, identifying=true}'}]
   2024-04-10T19:32:32.42+0900 [APP/TASK/9d230a5e/0] OUT 2024-04-10T10:32:32.425Z  INFO 6 --- [demo] [           main] o.s.batch.core.job.SimpleStepHandler     : Executing step: [BillingStep]
   2024-04-10T19:32:34.12+0900 [APP/TASK/9d230a5e/0] OUT 2024-04-10T10:32:34.121Z DEBUG 6 --- [demo] [           main] o.s.batch.core.step.tasklet.TaskletStep  : Applying contribution: [StepContribution: read=1000, written=1000, filtered=0, readSkips=0, writeSkips=0, processSkips=0, exitStatus=EXECUTING]
   2024-04-10T19:32:34.12+0900 [APP/TASK/9d230a5e/0] OUT 2024-04-10T10:32:34.126Z DEBUG 6 --- [demo] [           main] o.s.batch.core.step.tasklet.TaskletStep  : Saving step execution before commit: StepExecution: id=3, version=1, name=BillingStep, status=STARTED, exitStatus=EXECUTING, readCount=1000, filterCount=0, writeCount=1000 readSkipCount=0, writeSkipCount=0, processSkipCount=0, commitCount=1, rollbackCount=0, exitDescription=
   ...
   2024-04-10T19:32:34.96+0900 [APP/TASK/9d230a5e/0] OUT 2024-04-10T10:32:34.959Z DEBUG 6 --- [demo] [           main] o.s.batch.core.step.tasklet.TaskletStep  : Applying contribution: [StepContribution: read=0, written=0, filtered=0, readSkips=0, writeSkips=0, processSkips=0, exitStatus=EXECUTING]
   2024-04-10T19:32:34.96+0900 [APP/TASK/9d230a5e/0] OUT 2024-04-10T10:32:34.961Z DEBUG 6 --- [demo] [           main] o.s.batch.core.step.tasklet.TaskletStep  : Saving step execution before commit: StepExecution: id=3, version=11, name=BillingStep, status=STARTED, exitStatus=EXECUTING, readCount=10000, filterCount=0, writeCount=10000 readSkipCount=0, writeSkipCount=0, processSkipCount=0, commitCount=11, rollbackCount=0, exitDescription=
   2024-04-10T19:32:34.97+0900 [APP/TASK/9d230a5e/0] OUT 2024-04-10T10:32:34.972Z  INFO 6 --- [demo] [           main] o.s.batch.core.step.AbstractStep         : Step: [BillingStep] executed in 2s545ms
   2024-04-10T19:32:34.99+0900 [APP/TASK/9d230a5e/0] OUT 2024-04-10T10:32:34.995Z  INFO 6 --- [demo] [           main] o.s.b.c.l.support.SimpleJobLauncher      : Job: [SimpleJob: [name=BillingJob]] completed with the following parameters: [{'run.id':'{value=3, type=class java.lang.Long, identifying=true}','usageInfoFile':'{value=https://github.com/making/fakedata/raw/master/usageinfo/usageinfo-10000-en.csv, type=class java.lang.String, identifying=true}'}] and the following status: [COMPLETED] in 2s629ms
```

```
$ cf tasks billing-job 
Getting tasks for app billing-job in org handson-22297 / space demo as tmaki...

id   name       state       start time                      command
2    9d230a5e   SUCCEEDED   Wed, 10 Apr 2024 10:27:06 UTC   .java-buildpack/open_jdk_jre/bin/java org.springframework.boot.loader.launch.JarLauncher usageInfoFile=https://github.com/making/fakedata/raw/master/usageinfo/usageinfo-10000-en.csv
1    530028ad   SUCCEEDED   Wed, 10 Apr 2024 09:10:39 UTC   .java-buildpack/open_jdk_jre/bin/java org.springframework.boot.loader.launch.JarLauncher usageInfoFile=classpath:usageinfo.csv
```