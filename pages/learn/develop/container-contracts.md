---
title: Container Contracts
excerpt: A guide to using container contracts to enforce device compatibility
---

# Container contracts

__Note__ Container contracts are available for devices running Supervisor versions >= 10.6.0. All prior versions will not enforce any contracts.

Container contracts are used to ensure the compatibility of a device to run an application service. For example, container contracts may be used to ensure that the host operating system is running a specific version of balenaOS or Supervisor such that if an application is dependent on a specific version, it only gets deployed to those devices that can meet these requirements to fulfill the contract. Currently, you may define contract requirements based on the versions of [balenaOS](/reference/OS/overview/2.x/) and [Supervisor](/reference/supervisor/supervisor-api/).

Each service container can define the contracts that it enforces. If a container's contract requirements are not met, the release is not deployed to the device. You only need to define a contract for service containers that you wish to enforce a contract.

## Contract specification

Container contracts are defined as a `contract.yml` file and must be placed in the application structure alongside the corresponding service Dockerfile. For example, the following [multi-container application](/learn/develop/multicontainer/) defines two services named `first-service` and `second-service`, each with a corresponding `contract.yml` file.

```shell
.
├── docker-compose.yml
├── first-service
│   ├── contract.yml
│   └── Dockerfile.template
└── second-service
    ├── contract.yml
    └── Dockerfile.template
```    

`contract.yml` is a YAML file containing `type`, `slug`, `name`, and `requires` keys, which define the requirements of the contract. The following example requires the device to be running a version of balenaOS greater or equal than `2.40.0` for the contract to pass.

```yaml
type: 'sw.container'
slug: 'enforce-os-gt-240'
name: 'Enforce hostOS Version'
requires:
  - type: 'sw.os'
    version: '>=2.40.0'
```

* `type` should be set to `sw.container` for container contracts.
* `slug` should be a unique identifier to identify the contract.
* `name` should be a descriptive name of the contract.
* `requires` should be a list of requirements to be enforced by the contract defined by `type` and `version` keys. Currently, `type` may be either `sw.os` for the hostOS version or `sw.supervisor` for the device Supervisor version. 

__Note__ If `name` or `type` are not supplied, the build will fail. If `slug` is not provided, the build will complete, but the contract will fail validation on deployment. If `requires` is not provided, the contract will have no effect and the release will be deployed.

Multiple contract requirements may be added to a single contract. The following example requires that the hostOS is greater than or equal to 2.44.0, and the Supervisor version is greater than 10.

```yaml
type: 'sw.container'
slug: 'enforce-supervisor-and-os'
name: 'Enforce hostOS and supervisor'
requires:
  - type: 'sw.supervisor'
    version: '>10'
  - type: 'sw.os'
    version: '>=2.44.0'  
```

__Note__ Types in the contract requirements are not validated, so passing values for `type` other than `sw.os` or `sw.supervisor` will result in the contract passing.

When deploying a release, if the contract requirements are not met, the release will fail, and all devices in the application will remain on their current release. This default behavior can be overridden by including an [optional label](#optional-contracts) in the application docker-compose.yml file. If there are any existing running services, they will not be destroyed and continue running the previous release.

Should a contract fail, the logs will contain detail about the failing contract:

```shell
Could not move to new release: Some releases were rejected due to having unmet requirements:
Contracts: Services with unmet requirements: first-service
```

__Note__ For application fleets that contain a mixture of hostOS and Supervisor versions, contracts may be used to ensure only compatible devices run specific services as releases will only be deployed to those meeting the contract requirements.

## Optional Contracts

By default, when a service contract fails, none of the services are deployed to the device. However, in a multi-container application, it is possible to ignore those services that fail the contract requirements with the other services being deployed as normal. To do so, we make use of [Supervisor labels](/learn/develop/multicontainer/#labels) to indicate which services should be considered optional.

In the `docker-compose.yml` file of the application, add the `io.balena.features.optional=1` to the labels list for each service you wish to mark as optional. In the following example, even if the `first-service` service contract fails, the `second-service` service will still be deployed (assuming it doesn’t have any failing contracts of its own).

```Dockerfile
version: '2'
services:
  first-service:
    build: ./first-service
    labels: 
      - io.balena.features.optional=1
  second-service:
    build: ./second-service
```

__Note__ When using optional contracts, if there are any existing running services of the failed contract, they will be killed and not continue running the old version.
