# Datasource via Subsystem/Fraction

This example builds upon the JAX-RS/ShrinkWrap example to also
deploy a JDBC datasource through subsystem configuration.

Deploying a JDBC driver through this method requires that
the driver be a part of the JBoss Modules module tree. 

Just as WildFly-Swarm can provide portions of the module tree
through various jars, so can your own application.

By creating `src/main/resources/modules/com/h2database/h2/main/module.xml`,
this new module is made available during the boot of the app-server.

The `module.xml` references the `h2.jar` through Maven coordinates:

    <resources>
        <artifact name="com.h2database:h2:${version.h2}"/>
    </resources>

The `${version.h2}` is simply a property defined in your pom.xml,
and normal Maven resource-filtering replaces it, to avoid having
to hard-code it.

Please raise any issues found with this example on the main project:
https://github.com/wildfly-swarm/wildfly-swarm/issues

## Project `pomx.xml`

The project is a normal maven project with `jar` packaging, not `war`.

    <packaging>jar</packaging>

The project adds a `<plugin>` to configure `wildfly-swarm-plugin` to
create the runnable `.jar`.  Additional configuration parameters are
added to instruct the plugin to include the `com.h2database.h2` module
from the WildFly distribution.  This allows access to the H2 driver
jar.

    <plugin>
      <groupId>org.wildfly.swarm</groupId>
      <artifactId>wildfly-swarm-plugin</artifactId>
      <version>${version.wildfly-swarm}</version>
      <configuration>
        <mainClass>org.wildfly.swarm.examples.ds.subsystem.Main</mainClass>
      </configuration>
      <executions>
        <execution>
          <goals>
            <goal>package</goal>
          </goals>
        </execution>
      </executions>
    </plugin>

To define the needed parts of WildFly Swarm, a few dependencies are added

    <dependency>
        <groupId>org.wildfly.swarm</groupId>
        <artifactId>wildfly-swarm-jaxrs</artifactId>
        <version>${version.wildfly-swarm}</version>
    </dependency>
    <dependency>
        <groupId>org.wildfly.swarm</groupId>
        <artifactId>wildfly-swarm-datasources</artifactId>
        <version>${version.wildfly-swarm}</version>
    </dependency>

The `wildfly-swarm-jaxrs` dependency allows usage of ShrinkWrap APIs within the `main()` in addition
to providing the JAX-RS APIs.  The `wildfly-swarm-datasources` dependency provides configuration
classes for adding the JDBC driver and datasources to the container.

## Project `main()`

Since this project deploys JAX-RS resources without a `.war` being construction, it
provides its own `main()` method (specified above via the `maven-jar-plugin`) to
configure the container and deploy the resources programatically.

    package org.wildfly.swarm.examples.ds.subsystem;

    import org.wildfly.swarm.container.Container;
    import org.wildfly.swarm.datasources.Datasource;
    import org.wildfly.swarm.datasources.DatasourcesFraction;
    import org.wildfly.swarm.datasources.Driver;
    import org.wildfly.swarm.jaxrs.JAXRSDeployment;
    
    public class Main {

        public static void main(String[] args) throws Exception {
            Container container = new Container();

            container.subsystem(new DatasourcesFraction()
                            .driver(new Driver("h2")
                                    .xaDatasourceClassName("org.h2.jdbcx.JdbcDataSource")
                                    .module("com.h2database.h2"))
                            .datasource(new Datasource("ExampleDS")
                                    .driver("h2")
                                    .connectionURL("jdbc:h2:mem:test;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE")
                                    .authentication("sa", "sa"))
            );


            container.start();
    
            JAXRSDeployment appDeployment = new JAXRSDeployment( container );
            appDeployment.addResource(MyResource.class);

            container.deploy(appDeployment);
        }
    }


This method constructs a new default Container, which automatically
initializes all fractions (or subsystems) that are available.  The datasources
fraction has no particular default configuration, so by providing a
specific configuration we enable a driver and a datasource.

JNDI names are bound automatically.

The empty container is started.

A `JAXRSDeployment` is constructed, and the JAX-RS resource class is
added to it and then deployed to the container.

The resource looks up the Datasource through JNDI at run-time:

    @Path( "/" )
    public class MyResource {

        @GET
        @Produces( "text/plain" )
        public String get() throws NamingException, SQLException {
            Context ctx = new InitialContext();
            DataSource ds = (DataSource) ctx.lookup( "jboss/datasources/ExampleDS" );
            Connection conn = ds.getConnection();
            try {
                return "Howdy using connection: " + conn;
            } finally {
                conn.close();
            }
        }
    }


## Run

You can run it many ways:

* mvn package && java -jar ./target/wildfly-swarm-example-datasource-subsystem-1.0.0.Beta1-SNAPSHOT-swarm.jar
* mvn wildfly-swarm:run
* In your IDE run the `org.wildfly.swarm.examples.ds.subsystem.Main` class

## Use

    http://localhost:8080/
