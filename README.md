Apache Fineract

REQUIREMENTS
============
* `Java >= 17` (Azul Zulu JVM is tested by our CI on GitHub Actions)
* MariaDB `10.9`

You can run the required version of the database server in a container, instead of having to install it, like this:

    docker run --name mariadb-10.9 -p 3306:3306 -e MARIADB_ROOT_PASSWORD=mysql -d mariadb:10.9

and stop and destroy it like this:

    docker rm -f mariadb-10.9

<br>Beware that this database container database keeps its state inside the container and not on the host filesystem.  It is lost when you destroy (rm) this container.  This is typically fine for development.  See [Caveats: Where to Store Data on the database container documentation](https://hub.docker.com/_/mariadb) re. how to make it persistent instead of ephemeral.<br>

Tomcat v9 is only required if you wish to deploy the Fineract WAR to a separate external servlet container.  Note that you do not require to install Tomcat to develop Fineract, or to run it in production if you use the self-contained JAR, which transparently embeds a servlet container using Spring Boot.  (Until FINERACT-730, Tomcat 7/8 were also supported, but now Tomcat 9 is required.)

<br>IMPORTANT: If you use MySQL or MariaDB
============

Recently (after release `1.7.0`), we introduced improved date time handling in Fineract. Date time is from now on stored in UTC and we are enforcing UTC timezone even on the JDBC driver, e. g. for MySQL:

```
serverTimezone=UTC&useLegacyDatetimeCode=false&sessionVariables=time_zone=‘-00:00’
```

__DO__: If you do use MySQL as your Fineract database then the following configuration is highly recommended:

* Run the application in UTC (the default command line in our Docker image has the necessary parameters already set)
* Run the MySQL database server in UTC (if you use managed services like AWS RDS then this should be the default anyway, but it would be good to double-check)

__DON'T__: In case the Fineract instance and the MySQL server are __not__ running in UTC then the following could happen:

* MySQL is saving date time values differently from PostgreSQL
* Example scenario: if the Fineract instance runs in timezone: GMT+2, and the local date time is 2022-08-11 17:15 ...
* ... then __PostgreSQL saves__ the LocalDateTime as is: __2022-08-11 17:15__
* ... and __MySQL saves__ the LocalDateTime in UTC: __2022-08-11 15:15__
* ... but when we __read__ the date time from PostgreSQL __or__ from MySQL, then both systems give us the same values: __2022-08-11 17:15 GMT+2__

If a previously used Fineract instance didn't run in UTC (backward compatibility), then all prior dates will be read wrongly by MySQL/MariaDB. This can cause issues when you run the database migration scripts.

__RECOMMENDATION__: you need to shift all dates in your database by the timezone offset that your Fineract instance used.

<br>INSTRUCTIONS: How to run for local development
============

Run the following commands:
1. `./gradlew createDB -PdbName=fineract_tenants`
1. `./gradlew createDB -PdbName=fineract_default`
1. `./gradlew bootRun`


INSTRUCTIONS: How to execute Integration Tests -- RUN IN INTELLIJ ?
============
> Note that if this is the first time to access MySQL DB, then you may need to reset your password.

Run the following commands:
1. `./gradlew createDB -PdbName=fineract_tenants`
1. `./gradlew createDB -PdbName=fineract_default`
1. `./gradlew clean test`


INSTRUCTIONS: How to run using Docker and docker-compose
===================================================

It is possible to do a 'one-touch' installation of Fineract using containers (AKA "Docker").
Fineract now packs the mifos community-app web UI in it's docker deploy.
You can now run and test fineract with a GUI directly from the combined docker builds.
This includes the database running in a container.

As Prerequisites, you must have `docker` and `docker-compose` installed on your machine; see
[Docker Install](https://docs.docker.com/install/) and
[Docker Compose Install](https://docs.docker.com/compose/install/).

Alternatively, you can also use [Podman](https://github.com/containers/libpod)
(e.g. via `dnf install podman-docker`), and [Podman Compose](https://github.com/containers/podman-compose/)
(e.g. via `pip3 install podman-compose`) instead of Docker.

Now to run a new Fineract instance you can simply:

1. `git clone https://github.com/apache/fineract.git ; cd fineract`
1. for windows, use `git clone https://github.com/apache/fineract.git --config core.autocrlf=input ; cd fineract`
1. `./gradlew :fineract-provider:jibDockerBuild -x test`
1. `docker-compose up -d`
1. fineract (back-end) is running at https://localhost:8443/fineract-provider/
1. wait for https://localhost:8443/fineract-provider/actuator/health to return `{"status":"UP"}`
1. you must go to https://localhost:8443 and remember to accept the self-signed SSL certificate of the API once in your browser, otherwise  you get a message that is rather misleading from the UI.
1. community-app (UI) is running at http://localhost:9090/?baseApiUrl=https://localhost:8443/fineract-provider&tenantIdentifier=default
1. login using default _username_ `mifos` and _password_ `password`

https://hub.docker.com/r/apache/fineract has a pre-built container image of this project, built continuously.

You must specify the MySQL tenants database JDBC URL by passing it to the `fineract` container via environment
variables; please consult the [`docker-compose.yml`](docker-compose.yml) for exact details how to specify those.
_(Note that in previous versions, the `mysqlserver` environment variable used at `docker build` time instead of at
`docker run` time did something similar; this has changed in [FINERACT-773](https://issues.apache.org/jira/browse/FINERACT-773)),
and the `mysqlserver` environment variable is now no longer supported.)_

<br>TOMCAT CONFIGURATION
====================

Please refer to the `application.properties` and the official Spring Boot documentation (https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html) on how to do performance tuning for Tomcat. Note: you can set now the acceptable form POST size (default is 2MB) via environment variable `FINERACT_SERVER_TOMCAT_MAX_HTTP_FORM_POST_SIZE`.

Checkstyle and Spotless
============

This project enforces its code conventions using [checkstyle.xml](config/checkstyle/checkstyle.xml) through Checkstyle and [fineract-formatting-preferences.xml](config/fineract-formatting-preferences.xml) through Spotless. They are configured to run automatically during the normal Gradle build, and fail if there are any violations detected. You can run the following command to automatically fix spotless violations:

    `./gradlew spotlessApply`

Since some checks are present in both Checkstyle and Spotless, the same command can help you fix some of the Checkstyle violations (but not all, other Checkstyle violations need to fixed manually).

You can also check for Spotless violations (only; but normally don't have to, because the regular build full already includes this anyway):

    `./gradlew spotlessCheck`

We recommend that you configure your favourite Java IDE to match those conventions. For Eclipse, you can go to
Window > Java > Code Style and import our [config/fineractdev-formatter.xml](config/fineractdev-formatter.xml) under formatter section and [config/fineractdev-cleanup.xml](config/fineractdev-cleanup.xml) under Clean up section. The same fineractdev-formatter.xml configuration file (that can be used in Eclipse IDE) is also used by Spotless to both check for violations and autoformat code on the CLI.
You could also use Checkstyle directly in your IDE (but you don't neccesarily have to, it may just be more convenient for you).  For Eclipse, use https://checkstyle.org/eclipse-cs/ and load our checkstyle.xml into it, for IntelliJ you can use [CheckStyle-IDEA](https://plugins.jetbrains.com/plugin/1065-checkstyle-idea).


License
============

This project is licensed under Apache License Version 2.0. See <https://github.com/apache/incubator-fineract/blob/develop/LICENSE.md> for reference.

The Connector/J JDBC Driver client library from MariaDB.org, which is licensed under the LGPL,
is used in development when running integration tests that use the Liquibase library.  That JDBC
driver is however not included in and distributed with the Fineract product and is not
required to use the product.
If you are developer and object to using the LGPL licensed Connector/J JDBC driver,
simply do not run the integration tests that use the Liquibase library and/or use another JDBC driver.
As discussed in [LEGAL-462](https://issues.apache.org/jira/browse/LEGAL-462), this project therefore
complies with the [Apache Software Foundation third-party license policy](https://www.apache.org/legal/resolved.html).


<br><br>APACHE FINERACT PLATFORM API
============

The API for Fineract is documented in [apiLive.htm](fineract-provider/src/main/resources/static/api-docs/apiLive.htm), and the [apiLive.htm can be viewed on Fineract.dev](https://fineract.apache.org/legacy-docs/apiLive.htm "API Documentation").  If you have your own Fineract instance running, you can find this documentation under [/fineract-provider/api-docs/apiLive.htm](https://localhost:8443/fineract-provider/api-docs/apiLive.htm).

The Swagger documentation (work in progress; see [FINERACT-733](https://issues.apache.org/jira/browse/FINERACT-733)) can be accessed under [/fineract-provider/swagger-ui/index.html](https://localhost:8443/fineract-provider/swagger-ui/index.html) and [live Swagger UI here on Fineract.dev](https://demo.fineract.dev/fineract-provider/swagger-ui/index.html).

Apache Fineract supports client code generation using [Swagger Codegen](https://github.com/swagger-api/swagger-codegen) based on the [OpenAPI Specification](https://swagger.io/specification/).  For more instructions on how to generate the client code, check [docs/developers/swagger/client.md](docs/developers/swagger/client.md).


<br>ONLINE DEMOS
============

* [fineract.dev](https://www.fineract.dev) always runs the latest version of this code
* [demo.mifos.io](https://demo.mifos.io) A demo account is provided for users to experience the functionality of the Community App.  Users can use "mifos" for USERNAME and "password" for PASSWORD (without quotation marks).
* [Swagger-UI Demo video](https://www.youtube.com/watch?v=FlVd-0YAo6c) This is a demo video for Swagger-UI documentation, more information [here](https://github.com/apache/fineract#swagger-ui-documentation).


<br>DEVELOPERS
============
Please see <https://cwiki.apache.org/confluence/display/FINERACT/Contributor%27s+Zone> for the developers wiki page.

Please refer to <https://cwiki.apache.org/confluence/display/FINERACT/Fineract+101> for the first-time contribution to this project.

Please see <https://cwiki.apache.org/confluence/display/FINERACT/How-to+articles> for technical details to get started.

Please visit [our JIRA Dashboard](https://issues.apache.org/jira/secure/Dashboard.jspa?selectPageId=12335824) to find issues to work on, see what others are working on, or open new issues.


<br>GORVENANCE AND POLICIES
=======================

[Becoming a Committer](https://cwiki.apache.org/confluence/display/FINERACT/Becoming+a+Committer)
documents the process through which you can become a committer in this project.


<br>ERROR HANDLING GUIDELINES
------------------
* When catching exceptions, either rethrow them, or log them.  Either way, include the root cause by using `catch (SomeException e)` and then either `throw AnotherException("..details..", e)` or `LOG.error("...context...", e)`.
* Completely empty catch blocks are VERY suspicous!  Are you sure that you want to just "swallow" an exception?  Really, 100% totally absolutely sure?? ;-) Such "normal exceptions which just happen sometimes but are actually not really errors" are almost always a bad idea, can be a performance issue, and typically are an indication of another problem - e.g. the use of a wrong API which throws an Exception for an expected condition, when really you would want to use another API that instead returns something empty or optional.
* In tests, you'll typically never catch exceptions, but just propagate them, with `@Test void testXYZ() throws SomeException, AnotherException`..., so that the test fails if the exception happens.  Unless you actually really want to test for the occurence of a problem - in that case, use [JUnit's Assert.assertThrows()](https://github.com/junit-team/junit4/wiki/Exception-testing) (but not `@Test(expected = SomeException.class)`).
* Never catch `NullPointerException` & Co.

More Information
============
More details of the project can be found at <https://cwiki.apache.org/confluence/display/FINERACT>.
