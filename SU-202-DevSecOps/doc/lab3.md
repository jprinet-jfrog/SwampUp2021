# Lab 3

## Context

- You understood that late security validation is probably not the best way to go
- Shift left is your new motto

## Objective

![](images/lab3.png)

- Build the Docker image in a CI like environment using JFrog CLI
- Upload the build information to Artifactory
- Scan your build
- [Try to] Promote / Download the Docker image

## Configure Gradle build

```bash
jfrog rt gradle-config --server-id-resolve="${CLI_INSTANCE_ID}" --repo-resolve="${GRADLE_REPO_DEV}" --server-id-deploy="${CLI_INSTANCE_ID}" --repo-deploy="${GRADLE_REPO_DEV}" --use-wrapper=false --uses-plugin=true --deploy-ivy-desc=false
```

## Index resources

- index **devsecops-**** build

## Build the gradle project

```bash
printf "Building ${PROJECT_VERSION_LAB3}\nwith struts ${STRUTS_VERSION_UNSAFE} (unsafe)\n" 
```

```bash
jfrog rt gradle clean artifactoryPublish \
            -b build.gradle \
            --build-name "${CLI_GRADLE_BUILD_NAME}" \
            --build-number 1 \
            -PprojectVersion="${PROJECT_VERSION_LAB3}" \
            -PartifactoryUrl="${ARTIFACTORY_URL}" \
            -PartifactoryGradleRepo="${GRADLE_REPO_DEV}" \
            -PartifactoryUser="${ARTIFACTORY_LOGIN}" \
            -PartifactoryApiKey="${ARTIFACTORY_API_KEY}" \
            -PstrutsVersion="${STRUTS_VERSION_UNSAFE}"
```

## Publish Gradle Build info

```bash
jfrog rt build-publish --server-id="${CLI_INSTANCE_ID}" "${CLI_GRADLE_BUILD_NAME}" 1
```

## Create Xray policy

Create **fail-build-on-high-severity** security policy:
- trigger a violation on **high severity** security issue discovered
- **Fail Build** as action

## Create Xray watch

Create **devsecops-docker-build-watch** watch:
- add **devsecops-\*/*** build (pattern) as resource
- add **fail-build-on-high-severity** as policy

## Log into Docker registry

```bash
docker login -u "${ARTIFACTORY_LOGIN}" -p "${ARTIFACTORY_API_KEY}" "${DOCKER_REGISTRY_DEV}"
```

## Build Docker image

```bash
printf "Building ${IMAGE_ABSOLUTE_NAME_DEV_LAB3}\nwith base image ${BASE_IMAGE_UNSAFE} (unsafe)\n" 
```

```bash
docker build -t "${IMAGE_ABSOLUTE_NAME_DEV_LAB3}" --build-arg "BASE_IMAGE=${BASE_IMAGE_UNSAFE}" .
```

## Push Docker image to Artifactory

```bash
jfrog rt docker-push --server-id="${CLI_INSTANCE_ID}" --skip-login --build-name="${CLI_DOCKER_BUILD_NAME}" --build-number=1 --module="${CLI_DOCKER_BUILD_NAME}" "${IMAGE_ABSOLUTE_NAME_DEV_LAB3}" "${DOCKER_REPO_DEV}"
```

## Publish Docker Build info

```bash
jfrog rt build-publish --server-id="${CLI_INSTANCE_ID}" "${CLI_DOCKER_BUILD_NAME}" 1
```

## Scan Gradle Build

```bash
jfrog rt build-scan --server-id="${CLI_INSTANCE_ID}" "${CLI_GRADLE_BUILD_NAME}" 1
```

## Scan Docker Build

```bash
jfrog rt build-scan --server-id="${CLI_INSTANCE_ID}" "${CLI_DOCKER_BUILD_NAME}" 1
```

3 is the return code when a violation is found, it is a non success return code, you can fail your build based on that.
This is taken care of transparently when using one native Artifactory plugin for your CI server.

## Conclusion

Some vulnerabilities were detected when scanning the build, build pipeline is stopped.