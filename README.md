# cos-processors

Example Repository for the Container Build and Catalog for Processors

## Adding support for Camel Components to the Processor Container Image

The container image is built as a Maven project. We rely on JIB to build and push the container image.

The components supported in the Camel DSL for Processors must be added as dependencies to the `cos-processor-camel/pom.xml`
file. These components are from the Camel Quarkus project. Each time a component is added, we must ensure we have a 
productized version of it for our downstream builds.

So for example:

To allow the user to make use of the `kafka` component in their Camel DSL, we ensure that we have the `camel-quarkus-kafka`
dependency listed in the [pom.xml](cos-processor-camel/pom.xml).

Once this is done, the user will be able to write DSL similar to:

```yaml
      from:
        uri: "kafka:rblake-source"
        steps:
          - unmarshal:
              json: { }
          - log: "receiving event from the Kafka Topic"
          - marshal:
              json: { }
          - to:
              uri: "kafka:rblake-sink"
```

## Build and Push of the Container Image

The build and push of the container image is parameterized via command line arguments to Maven. We'll likely need to
alter Container Registry, Container Registry credentials and image tag for different builds e.g upstream vs downstream

So the following command will likely be run each time we merge to main to build + push the image:

```shell
 mvn clean package -Dquarkus.container-image.push=true -Dquarkus.container-image.username=$CONTAINER_REGISTRY_USER
  -Dquarkus.container-image.password=$CONTAINER_REGISTRY_PASSWORD -Dquarkus.container-image.group=rhoas -Dquarkus.container-image.tag=$GIT_BRANCH-$GIT_COMMIT_SHA
```

// TODO - Combine the build process with the downstream, productized build 

## Processor Catalog

The Control Plane needs to know which container image should be used when a user requests a Processor. It includes the container image in the payload from the Agent API to the data plane. 

To try and align with the Connector catalog, the proposal is to have a simplified catalog for Processors with a single entry for now.

This catalog would be regenerated each time a change is merged to the `main` branch of this repository to:

- Reference the most up-to-date Processor container image
- Increment the revision of the Processor catalog definition

### Distributing the Processor Catalog

The proposal is that the Processor Catalog is distributed via a Container Image. The Container image will be generated as part of the build process and pushed to the container registry. To use the catalog:

- Pull the required processor catalog version from the registry
- Copy or mount the files from the Container image into the required location
  - This can be achieved via an `InitContainer` in the case of Kubernetes for example  
