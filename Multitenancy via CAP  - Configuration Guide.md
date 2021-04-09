# Multitenancy via CAP  - Configuration Guide

**Table of Contents:**
- [Multitenancy via CAP  - Configuration Guide](#multitenancy-via-cap----configuration-guide)
  - [Prerequisite](#prerequisite)
  - [Maven Dependencies](#maven-dependencies)
  - [Configuration for XSUAA](#configuration-for-xsuaa)
  - [Add the MTX Sidecar Application](#add-the-mtx-sidecar-application)
  - [Wiring It Up in mta.yaml](#wiring-it-up-in-mtayaml)
  - [Implement Handlers for the subscription callbacks](#implement-handlers-for-the-subscription-callbacks)
  - [Related References:](#related-references)

## Prerequisite

The Multi-target application has been setup in the single tenant way.

First of all, rename the mta.yaml(which is for single tenant as of now) into mta-signle-tenant.yaml, and deplicate and rename it into mta.yaml.

## Maven Dependencies
- Update the pom.xml in root.
  
  
  ```
      <!-- DEPENDENCIES VERSION -->
      <jdk.version>1.8</jdk.version>
      <cds.services.version>1.13.1</cds.services.version>
      <spring.boot.version>2.4.1</spring.boot.version>
      <cloud.sdk.version>3.38.0</cloud.sdk.version>
      <node.version>v12.16.1</node.version>
      
        ......

    <dependencyManagement>
        <!-- CDS SERVICES -->
        <dependency>
          <groupId>com.sap.cds</groupId>
          <artifactId>cds-services-bom</artifactId>
          <version>${cds.services.version}</version>
          <type>pom</type> 
          <scope>import</scope>
        </dependency>

        <!-- SPRING BOOT -->
        <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-dependencies</artifactId>
          <version>${spring.boot.version}</version>
          <type>pom</type>
          <scope>import</scope>
        </dependency>

        <!-- CLOUD SDK -->
        <dependency>
          <groupId>com.sap.cloud.sdk</groupId>
          <artifactId>sdk-bom</artifactId>
          <version>${cloud.sdk.version}</version>
          <type>pom</type>
          <scope>import</scope>
        </dependency>

        <!-- MANAGE Spring XSUAA Library, because of Spring 2.4 incompatible version in Cloud SDK BOM -->
        <dependency>
          <groupId>com.sap.cloud.security.xsuaa</groupId>
          <artifactId>xsuaa-spring-boot-starter</artifactId>
          <version>2.8.1</version>
        </dependency>
      </dependencies>
    </dependencyManagement>
  ```
- Update the pom.xml in srv module.
  
  Add the below dependency:
  ```
      <dependency>
        <groupId>com.sap.cds</groupId>
        <artifactId>cds-starter-cloudfoundry</artifactId>
      </dependency>
  ```


## Configuration for XSUAA

- Add the 2 scopes into xs-security-mt.json
  - mtcallback
  - mtdeployment
  ```
  {
    "tenant-mode": "shared",
    "description": "sus-sm-rdp-core application",
    "scopes": [
      {
        "name": "uaa.user",
        "description": "UAA"
      },
      {
        "name": "$XSAPPNAME.admin",
        "description": "admin"
      },
      {
        "name": "$XSAPPNAME.mtcallback",
        "description": "Multi Tenancy Callback Access",
        "grant-as-authority-to-apps": [
          "$XSAPPNAME(application,sap-provisioning,tenant-onboarding)"
        ]
      },
      {
        "name": "$XSAPPNAME.mtdeployment",
        "description": "Scope to trigger a re-deployment of the database artifacts"
      }
    ],
    "role-templates": [
      {
        "name": "Token_Exchange",
        "description": "UAA",
        "scope-references": [
          "uaa.user"
        ]
      },
      {
        "name": "admin",
        "description": "generated",
        "scope-references": [
          "$XSAPPNAME.admin"
        ]
      }
    ]
  }
  ```
> - "tenant-mode" should be set as _shared_.
> - The _mtcallback_ scope is required by the onboarding process. 
> - The _mtdeployment_ scope is required to redeploy database artifacts at runtime.


## Add the MTX Sidecar Application
- Create a new submodule in folder mtx-sidecar, and add the package.json in it
  ```
  {
      "name": "deploy",
      "dependencies": {
          "@sap/cds": "^4.2.4",
          "@sap/cds-mtx": "^1.1.5",
          "@sap/hdi-deploy": "^3.11.11",
          "@sap/instance-manager": "^2.2.0",
          "@sap/xssec": "^3.0.9",
          "@sap/hana-client": "^2.5.105",
          "express": "^4.17.1",
          "passport": "^0.4.1"
      },
      "scripts": {
          "start": "node server.js"
      }
  }
  ```
> - @sap/cds-mtx version should be higher than '1.1.5'.

- Create the server.js

  ```
  const app = require('express')();
  const cds = require('@sap/cds');
  const cp = require('child_process'); 

  const main = async () => {
      await cds.connect.to('db');
      const PORT = process.env.PORT || 4004;
      await cds.mtx.in(app);

      const provisioning = await cds.connect.to('ProvisioningService');
      provisioning.before(['UPDATE', 'DELETE', 'READ'], 'tenant', async (req) => {

          // check for the scope "mtcallback"
          if (!req.user.is('mtcallback')) {
              // Reject request
              const e = new Error('Forbidden');
              e.code = 403;
              return req.reject(e);
          }
      });
      app.listen(PORT);
  }

  main();


  ```
  > By default, this script implements authorization and checks for the scope mtcallback. If you use a custom scope name for requests issued by the SaaS Provisioning Service in your application security descriptor (security.json),

- Configure the build by means of two .cdsrc.json files.  
  Because the MTX sidecar will build the CDS model, you need to configure the build by means of two .cdsrc.json files
  + Add the build task for mtx-sidecar into .cdsrc.json file in the root folder of your project.
    ```
                {
                    "for": "mtx",
                    "src": ".",
                    "dest": "mtx-sidecar",
                    "options": {
                        "model": [
                            "db",
                            "srv",
                            "app"
                        ]
                    }
                },
      ......
    ```
    > This step is to specify from which location the CDS files should be collected. 
      
  + Add .cdsrc.json file into the mtx-sidecar directory
    ```
    {
        "hana": {
            "deploy-format": "hdbtable"
        },
        "build": {
            "tasks": [
                {
                    "for": "hana",
                    "src": "db",
                    "options": {
                        "model": [
                            "db",
                            "srv",
                            "app"
                        ]
                    }
                },
                {
                    "for": "java-cf",
                    "src": "srv",
                    "options": {
                        "model": [
                            "db",
                            "srv",
                            "app"
                        ]
                    }
                }
            ]
        },
        "odata": {
            "version": "v2"
        },
        "requires": {
            "db": {
                "kind": "hana",
                "multiTenant": true,
                "vcap": {
                    "label": "service-manager"
                }
            },
            "uaa": {
                "kind": "xsuaa"
            }
        }
    }

    ```
  > Due to the usage of mtx-sidecar approach, OData v2 and v4 could not be supported in parallel.
    >> [Open Point?]

- Add the mtx-sidecar module to your mta.yaml file:
  ```
  # ------------------- SIDECAR MODULE ---------------------
  - name: sus-sm-rdp-core-mtx-sidecar
  # ------------------- SIDECAR MODULE ---------------------
   type: nodejs
   path: mtx-sidecar
   parameters:
     memory: 256M
     disk-quota: 512M
   requires:
     - name: sus-sm-rdp-core-uaa
     - name: sus-sm-rdp-core-service-manager
   provides:
     - name: mtx-sidecar
       properties:
         url: ${default-url}

  ```
> 

## Wiring It Up in mta.yaml
 
    1. Add the gloabal paramters.
        ```
        domain: cfapps.sap.hana.ondemand.com
        appname: smrdp-core
        ```
       > Note: The parameters should be defined via *.mtaext if there will be difference between different landsape deployment. 
    2. Remove the db-deployer(sus-sm-rdp-core-db-deployer) and hdi-container (sus-sm-rdp-persist)
        > The mtx-sidecar will help to create the hdi container and the schema's artifacts instead for each subsribed account.
      
    3. Update xsuaa service.
       ```
        # ------------------------------------------------------------
        - name: sus-sm-rdp-core-uaa
        # ------------------------------------------------------------
          type: com.sap.xs.uaa
          parameters:
            path: ./xs-security-mt.json
            service: xsuaa
            service-plan: broker
            config: # override xsappname in cds-security.json, xsappname needs to be unique
              xsappname: sus-sm-rdp-core-${org}-${space}
       ```
        > _xsappname_ should be unique globally.   
        > _service-plan_ should be 'broker' type.
    4. Add 2 services - service-manager and saas-registry into mta.yaml.
       ```
        # ------------------------------------------------------------
        - name: sus-sm-rdp-core-service-manager
        # ------------------------------------------------------------
          type: org.cloudfoundry.managed-service
          parameters:
            service: service-manager
            service-plan: container

        # ------------------------------------------------------------
        - name: sus-sm-rdp-core-saas-registry
        # ------------------------------------------------------------
          type: org.cloudfoundry.managed-service
          parameters:
            service: saas-registry
            service-plan: application
            config:
              appName: sus-sm-rdp-core-${org}-${space}
              xsappname: sus-sm-rdp-core-${org}-${space}
              displayName: 'Responsible Design & Production - Core'
              description: 'This is the SaaS Application for Responsible Design & Production - Core.'
              category: 'Responsible Design & Production'
              appUrls:
                getDependencies: ~{srv-api/url}/mt/v1.0/subscriptions/dependencies
                onSubscription: ~{srv-api/url}/mt/v1.0/subscriptions/tenants/{tenantId}
          requires:
            - name: srv-api
       ```
       > A service-manager instance is required that the CAP Java SDK can create database containers per tenant at application runtime.  
       > A saas-registry service instance is required to make your application known to the SAP BTP Provisioning service and to register the endpoints that should be called when tenants are added or removed. 
    4. Add the module:mtx-sidecar as below:
       ```
        # ------------------- SIDECAR MODULE ---------------------
        - name: sus-sm-rdp-core-mtx-sidecar
        # ------------------- SIDECAR MODULE ---------------------
          type: nodejs
          path: mtx-sidecar
          parameters:
            memory: 256M
            disk-quota: 512M
          requires:
            - name: sus-sm-rdp-core-uaa
            - name: sus-sm-rdp-core-service-manager
          provides:
            - name: mtx-sidecar
              properties:
                url: ${default-url}        

       ``` 

    5. Update the module defination of srv module.  
      - Add dependency services: saas-registry, service-manager, appouter, mtx-sidecar  
      - Add properties: CDS_MULTITENANCY_CALLBACKURL
       ```
        # --------------------- SERVER MODULE ------------------------
        - name: sus-sm-rdp-core-srv
        # ------------------------------------------------------------
          type: java
          path: srv
          parameters:
            stable-host: ${org}-${space}-sus-sm-rdp-core-srv
            hosts:
              - ${stable-host}
          properties:
            SPRING_PROFILES_ACTIVE: cloud
            CDS_MULTITENANCY_CALLBACKURL: https://${stable-host}.${domain}
          build-parameters:
            [...]
          provides:
            - name: srv-api
              properties:
                url: https://${stable-host}.${domain}
          requires:
            # Resources extracted from CAP configuration
            - name: sus-sm-rdp-core-uaa
            - name: sus-sm-rdp-core-service-manager
            - name: sus-sm-rdp-core-saas-registry
            - name: sus-sm-rdp-core-portal-resources
            - name: mtx-sidecar
              properties:
                CDS_MULTITENANCY_SIDECAR_URL: ~{url}
            - name: app-url
              properties:
                CDS_MULTITENANCY_APPUI_URL: ~{url}
       ```
    6. Update the module defination of srv module.  
      - Add dependency services: saas-registry  
      - Add properties: TENANT_HOST_PATTERN as the parttern \<subsriber subaccount>-\<application name>-\<global account domain>
        ```
        # ------------------- APPROUTER MODULE ---------------------
        - name: sus-sm-rdp-core-approuter
        # ------------------------------------------------------------
          type: javascript.nodejs
          path: approuter
          parameters:
            disk-quota: 256M
            memory: 512M
          properties:
            SAP_JWT_TRUST_ACL: "[{\"clientid\":\"*\",\"identityzone\":\"sap-provisioning\"}]"
            TENANT_HOST_PATTERN: ^(.*)-${appname}.${domain}
            destinations:
              - name: dest_odata_srv
                url: "~{srv-api/url}"
                forwardAuthToken: true
          requires:
            - name: sus-sm-rdp-core-html5-repo-runtime
            - name: sus-sm-rdp-core-portal-resources
            - name: sus-sm-rdp-core-uaa
            - name: sus-sm-rdp-core-saas-registry
            - name: srv-api
          provides:
            - name: app-url
              properties:
                url: '${default-url}'
        ```
##  Implement Handlers for the subscription callbacks

  - Create new class SubscriptionHandler in base of com.sap.cds.services.handler.EventHandler. 
  - Add 3 event handlers 
    - **EVENT_SUBSCRIBE**  
      After the subsribtion is finished, the subscription URL should be return, otherwise the default URL will be return.
    - **EVENT_UNSUBSCRIBE**  
      After the unsubsribtion is finished, the tenant should be removed.   
    - **EVENT_GET_DEPENDENCIES**  
      Whin the subsribtion process, return the service dependencies of the application.   
      Here the portal servcie should be return.

  ```
  /**
  * Handler that implements subscription logic
  */
  @Component
  @Profile("cloud")
  @ServiceName(MtSubscriptionService.DEFAULT_NAME)
  class SubscriptionHandler implements EventHandler {

      @Value("${vcap.services.sus-sm-rdp-core-portal-resources.credentials.uaa.xsappname}")
      private String portalXsappname;

      @After(event = MtSubscriptionService.EVENT_SUBSCRIBE)
      public void afterSubscription(MtSubscribeEventContext context) {

          String subscriptionURL = "https://" + context.getSubscriptionPayload().subscribedSubdomain
                  + "-smrdp-core.cfapps.sap.hana.ondemand.com";
          context.setResult(subscriptionURL);

      }

      @On(event = MtSubscriptionService.EVENT_GET_DEPENDENCIES)
      public void onGetDependencies(MtGetDependenciesEventContext context) {
          ApplicationDependency dependency = new ApplicationDependency();
          dependency.xsappname = portalXsappname; // getServiceProperty("portal", "xsappname");
          context.setResult(Arrays.asList(dependency));
          // add the dependencies: portal...
          log.info(MtSubscriptionService.EVENT_SUBSCRIBE + "- Add the dependencies:" + dependency.xsappname);
          // context.setResult(Collections.emptyList());

      }

      @Before(event = MtSubscriptionService.EVENT_UNSUBSCRIBE)
      public void beforeUnsubscribe(MtUnsubscribeEventContext context) {
          // Trigger deletion of database container of off-boarded tenant
          context.setDelete(true);
      }

  }
  ```
  > - Subscription events are generated when a new tenant is added. By default, subscription creates a new database container for a newly subscribed tenant.   
  > - By default an EVENT_UNSUBSCRIBE is sent when a tenant is removed. By default, tenant-specific database containers aren’t deleted during removal. However, you can register a customer handler change this behavior.
  > - The event EVENT_GET_DEPENDENCIES fires when the SaaS Provisioning calls the getDependencies callback. Hence, if your application consumes any reuse services provided by SAP, you must implement the EVENT_GET_DEPENDENCIES to return the service dependencies of the application.





## Related References:
- [Developing Multitenant Applications in the Cloud Foundry Environment](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/5e8a2b74e4f2442b8257c850ed912f48.html).
- [CAP - Multitenancy](https://cap.cloud.sap/docs/java/multitenancy).
- [BTP - Expose Your App as a Content Provider](https://help.sap.com/viewer/ad4b9f0b14b0458cad9bd27bf435637d/Cloud/en-US/8a25fddb747f4ba992969049de96f836.html).
- [Troubleshooting](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/ae1d53e5fbe14383bfafe690f52711d7.html).
