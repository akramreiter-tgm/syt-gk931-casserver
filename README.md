# syt-gk931-cas

`author: akramreiter`

# Protocol

### Setting up the server

Clone or download the [overlay template](https://github.com/apereo/cas-overlay-template).

Run `gradlew copyCasConfiguration` to copy the config files.

Setup a keystore and certificate with `gradlew createKeystore`.

This repository per default uses configuration from the directory `<working_partition>:/etc`. The quick workaround I used to resolve this was creating a symbolic link from `D:/etc` to the overlay template's `etc/` directory.

`gradlew run` starts the server.

The service is hosted on https://localhost:8443/cas/login.

Logging in with the default user `casuser` and the password `Mellon` should work on this page.

### Allow apps to access the service

To allow other apps to use the CAS login service, it needs to be explicitly allowed in the service registry.

In this case, it'll be done via the JSON service registry. The necessary package is already listed under dependencies in `build.gradle`, but commented out. Remove the `//`   to use it.

````
dependencies {
    // Other CAS dependencies/modules may be listed here...
    compile "org.apereo.cas:cas-server-support-json-service-registry:${casServerVersion}"
}
````

The path to the location of the services needs to be configured and `initFromJson` needs to be enabled in `/etc/cas/config/cas.properties`. In this example, the services are located at `/etc/cas/services`.

````properties
cas.serviceRegistry.initFromJson:true
cas.serviceRegistry.json.location:file:/etc/cas/services
````

Create a new file in `etc/cas/services`. The one I used is called `wildcard.json` to reflect that it's a wildcard that allows access universally. This is only for demo purposes and **shouldn't under any circumstances be used in a production environment**.

````json
{
  "@class" :            "org.apereo.cas.services.RegexRegisteredService",
  "serviceId" :         "^(https|http)://.*",
  "name" :              "Wildcard",
  "id" :                493,
  "evaluationOrder" :   99
}
````

The `id` shouldn't overlap with the ids of other services and `evaluationOrder` sets the order in which services are evaluated, as the name implies.

After restarting the server, it should show that 1 service is loaded

![](images\serviceloaded.PNG)

### Client

[This webapp](https://github.com/cas-projects/cas-sample-java-webapp) was used as the basis for the client. To connect to the local CAS server, the `web.xml` in `src/main/webapp/WEB-INF` had to be edited. In accordance to the instructions in its [readme](cas-sample-java-webapp-master/README.md), all instances of ` https://mmoayyed.unicon.net:8443 ` were replaced with `https://localhost:8443` since both the webapp and the server run on the same system.

The webapp needs a Java keystore with the password `changeit` in `ètc/`, which can be created with `keytool -genkey -keyalg RSA -alias localhost -keystore thekeystore -validity 360`. 

If there is no trust store, the webapp throws security errors when receiving a response from the CAS server. To prevent this, a trust store needs to be created from the server's certificate and moved to the webapp's `etc/jetty` directory.

````
keytool -export -alias cas -file cert -keystore thekeystore
keytool -importcert -file cert -keystore thetruststore -alias cas
````

Additionally, the trust store needs to be recognized, which was done via additional Jvm args in the `pom.xml` of the webapp.

````
<plugin>
                <groupId>org.eclipse.jetty</groupId>
                <artifactId>jetty-maven-plugin</artifactId>
                <version>9.3.6.v20151106</version>
                <configuration>
                    <jettyXml>${basedir}/etc/jetty/jetty.xml,${basedir}/etc/jetty/jetty-ssl.xml,${basedir}/etc/jetty/jetty-https.xml</jettyXml>
                    <systemProperties>
                        <systemProperty>
                            <name>org.eclipse.jetty.annotations.maxWait</name>
                            <value>300</value>
                        </systemProperty>
                    </systemProperties>
                    <webApp>
                        <contextPath>/sample</contextPath>
                    	<overrideDescriptor>${basedir}/etc/jetty/web.xml</overrideDescriptor>
                    </webApp>
                    <jvmArgs>-Djavax.net.ssl.trustStore=etc/jetty/thetruststore -Djavax.net.ssl.trustStorePassword=changeit -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true  -Djavax.net.debug=ssl:handshake -Xdebug -Xrunjdwp:transport=dt_socket,address=5002,server=y,suspend=n</jvmArgs>
                </configuration>
            </plugin>
````



Running the webapp is now possible via

````
mvn clean package jetty:run-forked
````

While logging in directly on the CAS server works fine, trying to do so gave me the following error: 

Server ![](images/servererr.png)

Client![](images/clienterr.png)

I spent roughly 7 hours working on this task, at least 2 of which were wasted trying to troubleshoot this single error. I wasn't able to find a solution and I have to stop here.

## Fragestellungen

#### Was bedeutet Single Sign On?

SSO bedeutet, dass durch das Einloggen an einem Server der Zugriff auf mehrere Services möglich ist. Dies wird durch einen Central Authentication Server (CAS) und Session-cookies ermöglicht.

####  Zeigen Sie in einem Diagramm die Ablaeufe von CAS Client, CAS Server, Service und User.

![](https://miro.medium.com/max/656/1*yCpUC4xPxtYV-LKbUv_TmQ.jpeg)

####  Was ist ein TGT und beschreiben Sie die Funktionsweise eine TGT?

Ein TGT ist effektiv eine Datei, die IP-Adresse des Clients, Gültigkeitsdauer und einen vom Ticket Granting Server (TGS) generierten Key. Der TGS hat 2 wichtige Funktionen:

- auf Ticket Requests von Clients reagieren und TGTs ausstellen
- Authentifizierung mittels TGTs ermöglichen und Service Tickets an den Client übermitteln

####  Welche Features/Merkmale bietet der Apereo CAS Server an? 

- Spring Boot Java Serverkomponente
- Pluggable Authentication (Einbindung von LDAP, MongoDb, etc für auth)
- Support für verschiedene Auth-Protokolle (Oauth2, CAS, SAML, etc)
- Multifactor Authentication (z.B. Google Authenticator)
- Delegated Authentication via external provider (Login via Facebook, Twitter, etc)
- Eingebauter Support für
  - Passwortmanagement
  - Benachrichtigungen
  - Terms of Use
  - Surrogate Authentication
- Registrieren von Client-Anwendungen
- Cross-platform support für Clients

####  Beschreiben Sie den CAS Server einer CAS Architektur?

Ist ein Server, der für die Authentifizierung für 1 oder mehrere Client Services zuständig ist. Hat 3 Hauptfunktionen für die Interaktion mit Usern und Client Services

- Stellt Tickets an User aus, die einen Client Web Service nutzen wollen, wenn sie sich einloggen (via Login-URL)
- Validiert Tickets, die Client Web Services erhalten (via Validation-URL)
- Invalidiert Tickets, wenn sich User ausloggen

#### Beschreiben Sie den CAS Client einer CAS Architektur?

Ist ein Webservice, das nur von vom CAS Server authentifizierten Usern genutzt werden kann. Nicht eingeloggte User werden an die Login-URL des CAS Server umgeleitet und müssen sich dort einloggen. Bei eingeloggten Usern wird das Ticket des Users vom CAS Server validiert bevor Zugriff auf den Client Service möglich ist.

####  Was ist die Aufgabe eines Tickets in einem TGT Service? 

Mit einem TGT können Service Tickets vom TGS angefordert werden, die den Zugriff auf gesicherte Services ermöglichen.

## Sources

- [Apereo CAS Wiki](https://apereo.github.io/cas/6.0.x/index.html)
- [Apereo CAS overlay template](https://github.com/apereo/cas-overlay-template)
- [Baeldung CAS SSO with Spring Security]( https://www.baeldung.com/spring-security-cas-sso )
- [CAS sample webapp](https://github.com/cas-projects/cas-sample-java-webapp)
- [Techopedia TGS](https://www.techopedia.com/definition/27186/ticket-granting-server-tgs)
- [CAS Infographic source](https://medium.com/@adiletmaratov/central-authentication-service-cas-implementation-using-django-microservices-70c4c50d5b6f)