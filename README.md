Jooq-Modelator
==============

[![GitHub license](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://raw.githubusercontent.com/ayedo/jooq-modelator/master/LICENSE)
![Build status](https://github.com/dim777/jooq-modelator/actions/workflows/check.yml/badge.svg)

## Overview

This gradle plugin generates Jooq metamodel classes from Flyway & Liquibase migration files. It does so by running them against a dockerized database, and then running the Jooq generator against that database.

It solves the following problems:

- No need to maintain and run local databases for builds
- You can run migrations against your very own RDMS, as opposed to an in memory H2 in compatibility mode like other plugins do
- The plugin is incremental end-to-end, and only runs if the migrations have changed, or the target metamodel folder has changed

## How it works

The plugin tries to pull the image with a provided tag from the docker store if necessary. It will then create a container of that image, or reuse an existing container. The plugin uses a key to tag the containers it has created. Then the container is started. 

A health check waits for the database inside the container to start. The health check is RDBMs and docker independent, as the plugin sends a SQL statement, and waits for its successful execution.

When the database is ready, the migrators are run against it.

After the migrations the jooq-generator is run.

## Requirements

You need to have Docker installed.

The plugin has been tested with Version 18.06.1-ce-mac73 (26764).

## Supported Technologies

Two migration engines are supported:

- Flyway (version '6.5.1')
- Liquibase (version '3.10.1')

__For Liquibase there are limitations:__

- You cannot choose the name of your database change log. __It has to be named 'databaseChangeLog'__. The file ending does not matter, and can be any of the supported file types.
- All migration files need be located within the configured migrations folders (see section 'Configuration'). This is required for incremental build support.

All databases which you can run in a Docker container, and for which a JDBC driver can be provided, are supported. The plugin has been successfully tested with Postgres 9.6, and MariaDB 10.2.

Due to backwards incompatible changes in the API, __no jooq generator version older than 3.11.0 is currently supported__.

## Installation

Add the following to your *build.gradle* plugin configuration block:
build.gradle
```groovy
plugins {
    id("jooq-modelator-plugin") version "1.0.0-SNAPSHOT"
}
```


```groovy
pluginManagement {
    repositories {
        maven {
            url 'https://nexus.i-neb.net/repository/maven-snapshots/'
        }
        maven {
            url 'https://nexus.i-neb.net/repository/maven-releases/'
        }
        gradlePluginPortal()
        ...
    }
    ...
}
```

build.gradle.kts
```groovy
plugins {
    id("jooq-modelator-plugin") version "1.0.1"
}
```
settings.gradle.kts
```groovy
pluginManagement {
    repositories {
        maven {
            url = uri("https://nexus.i-neb.net/repository/maven-snapshots/")
        }
        maven {
            url = uri("https://nexus.i-neb.net/repository/maven-releases/")
        }
        maven { url = uri("https://repo.spring.io/milestone") }
        gradlePluginPortal()
        ...
    }
    ...
}

```



## Configuration

To configure the plugin you need to add two things:

- A 'jooqModelator' plugin configuration extension (subsection "Plugin Configuration")
- A 'jooqModelatorRuntime' configuration in the dependencies for your database driver (subsection "Database Configuration") 

For example projects please see section "Example Projects".

### Plugin Configuration

Add the following to your build script:


    jooqModelator {
    
        // JOOQ RELATED CONFIGURATION
        
        // The version of the jooq-generator that should be used
        // The dependency is added, and loaded dynamically.
        // Only versions 3.11.0 and later are supported.
        jooqVersion = '3.12.0' // required, this is an example
        
        // Which edition of the jooq-generator to be used.
        // Possible values are: "OSS", "PRO", "PRO_JAVA_6", or "TRIAL".
        jooqEdition = 'OSS' // required, this is an example
    
        // The path to the XML jooq configuration file
        jooqConfigPath = '/var/folders/temp/jooqConfig.xml' // required, this is an example
    
        // The path to where the metamodel will be generated to.
        // Important: this needs to be kept in sync with the path configured in the jooqConfig.xml
        // the reason it needs to be configured here again is for incremental build support to work
        jooqOutputPath = '/var/folders/temp/ch/acme/metamodel' // required, this is an example

        // MIGRATION ENGINE RELATED CONFIGURATION
        
        // Which migration engine to use. 
        // Possible values are: 'FLYWAY', or 'LIQUIBASE'.
        migrationEngine = 'FLYWAY' // required, this is an example
            
        // a list of paths to the folders containing the migration files
        migrationsPaths = ['/var/folders/temp/migrations'] // required, this is an example
    
        // DOCKER RELATED CONFIGURATION
        
        // The tag of the image that will be pulled, and used to create the db container
        dockerTag = 'postgres:11.5' // required, this is an example
    
        // The environment variables to be passed to the docker container
        dockerEnv = ['POSTGRES_DB=postgres', 'POSTGRES_USER=postgres', 'POSTGRES_PASSWORD=secret'] // required, this is an example
    
        // The container port bindings to use on host and container respectively
        dockerHostPort = 5432 // required, this is an example
    
        dockerContainerPort = 5432 // required, this is an example
    
        // The plugin labels the containers it creates, and uses
        // the following labelkey as key to do so.
        // Should normally be left to the default
        labelKey = 'ch.ayedo.jooqmodelator' // Not required. This is the default
    
        // HEALTH CHECK RELATED CONFIGURATION
        
        // How long to wait in between failed retries. In milliseconds.
        delayMs = 500 // Not required. This is the default
    
        // How long to maximally wait for the database to react to the healthcheck. In milliseconds.
        maxDurationMs = 20000 // Not required. This is the default
    
        // The SQL statement to send to the database server as part of the health check.
        sql = 'SELECT 1' // Not required. This is the default
    }

### Database Configuration

You add your database drivers as follows:

    dependencies {
        // Add your JDBC drivers, and generator extensions here
        jooqModelatorRuntime('org.postgresql:postgresql:42.2.6')
    }

### Configuration Changes

When you change the `dockerTag`, `dockerEnv`, `dockerHostPort`, or `dockerContainerPort` a new container will be created. The old one is not deleted.

## Usage

The plugins adds a task named *generateJooqMetamodel* to your build.
Use it to generate the Jooq metamodel.

    ./gradlew generateJooqMetamodel

Please note: the first time you run the plugin it might take a long time to download missing docker images.

## Example Projects

### Flyway

Postgres: [click here](https://github.com/ayedo/jooq-modelator-examples/tree/flywayPostgres)

MariaDb: [click here](https://github.com/ayedo/jooq-modelator-examples/tree/flywayMariaDb)

### Liquibase

Postgres: [click here](https://github.com/ayedo/jooq-modelator-examples/tree/liquibasePostgres)

MariaDb: [click here](https://github.com/ayedo/jooq-modelator-examples/tree/liquibaseMariaDb)

