# Spring Batch Demo for Spring Cloud Data Flow

The projects here contain Spring Batch based applications used to demo [Composed Task](https://dataflow.spring.io/docs/batch-developer-guides/batch/data-flow-composed-task/)

## Requirements
- wget
- Java 1.8+
- Maven
- Database tool to query MySQL to verify results.(Make sure to expose MySQL port locally if using docker)
- Python 3+ (Only to run local webserver to make it easier for SCDF to load our apps)

## Quickstart

Install Docker for your desktop (engine version 18.09.2 or higher)

Download docker-compose file
```
wget https://raw.githubusercontent.com/spring-cloud/spring-cloud-dataflow/2.1.0.RELEASE/spring-cloud-dataflow-server/docker-compose.yml
```
If you do not have wget installed, you can open the link above in a web browser and save it to your machine.

### Expose MySQL port to locahost

Expose MySQL port so we can use it to verify results.

- Open the docker-compose.yml file that we downloaded in a text editor

REMOVE the following lines
```yaml
expose:
      - "3036"

```

ADD the following lines to the `mysql` section

```yaml
ports:
        - "33061:3306"
```


Start it up!
From the directory where the docker-compose.yml is saved, run:

```bash
export DATAFLOW_VERSION=2.1.0.RELEASE
export SKIPPER_VERSION=2.0.2.RELEASE
docker-compose up

```

Connect your favorite MySQL viewer to port 33061 on localhost. username:root password:rootpw

###

## Modules

- **core:** Contains common code used by the rest of the applications
- **file-ingest:** Spring batch application that reads first name and last name from a given csv file as `filepath` parameter and write to the database table called `persons`. The default behavior of the common [PersonItemProcessor](batch/core/src/main/java/io/spring/cloud/dataflow/batch/processor/PersonItemProcessor.java) is to not do anything to the payload
- **db-transform:** Spring batch application that reads first name and last name from `persons` table and apply an optional transformation via the [PersonItemProcessor](batch/core/src/main/java/io/spring/cloud/dataflow/batch/processor/PersonItemProcessor.java). The options supported are "NONE","UPPERCASE","LOWERCASE","BACKWARDS"


## Build

### Download the code


```bash
git clone https://github.com/Pivotal-Field-Engineering/spring-cloud-dataflow-samples.git
```

Build all the modules.

```bash
cd spring-cloud-dataflow-samples/batch
mvn clean package

```


## How to Run
The following instructions assume you are running it locally using docker. To make it easy to register the application with SCDF locally, copy the file-ingest and db-transform jar files to a common directory and run a web server in the directory so SCDF.

From the batch directoy run

```bash
mkdir tasks
cp file-ingest/target/ingest-1.0.0.BUILD-SNAPSHOT.jar ./tasks
cp cp db-transform/target/dbtransform-1.0.0.BUILD-SNAPSHOT.jar ./tasks
cd tasks
python -m SimpleHTTPServer 8000
```

### SCDF Shell
We will use the SCDF shell to install our applications. To run:

```bash
docker exec -it dataflow-server java -jar shell.jar
```

### Simple File Ingest

#### Register the file-ingest app

```bash
app register --name Demo-ImportFileApp --type task --uri http://host.docker.internal:8000/ingest-1.0.0.BUILD-SNAPSHOT.jar
```

#### Verify
```bash
app info --name Demo-ImportFileApp --type task
```

#### Create task
```bash
task create ImportTask --definition "ImportFile: Demo-ImportFileApp --filepath=classpath:data.csv"
```

#### Run the task
```bash
task launch ImportTask --arguments "--increment-instance-enabled=true"
```

### Database Transform

#### Register the db-transform app
```bash
app register --name Demo-DbTransformApp --type task --uri http://host.docker.internal:8000/db-transform-1.0.0.BUILD-SNAPSHOT.jar
```

#### Verify
```bash
app info --name Demo-DbTransformApp --type task
```

#### Create task
```bash
task create DbUppercaseTask --definition "Uppercase: Demo-DbTransformApp --action=UPPERCASE"
```

#### Run the task
```bash
task launch DbUppercaseTask --arguments "--increment-instance-enabled=true"
```

### Composed Task that demos Distributed Saga Pattern

#### Create the Composed Task