# build action
This action performs build related operations for javascript/typescript projects. The following can be performed by this action:

- Install npm dependencies (using `npm install --production`)
- Runs tests (using `npm run ci:test`)
- Runs compilation/build (using `npm run ci:build`)
- Builds docker container using `docker-compose`
- Tags docker container using naming conventions
- Pushes docker container to specified repository (e.g. AWS ECR or DockerHub)
- Creates a GitHub [deployment](https://docs.github.com/en/rest/guides/delivering-deployments) for the built docker image

## quickstart
This action relies on the following default conventions to work. While you can specify your own overrides, the following default conventions are expected by this action: 
- `package.json` file with the following `scripts` defined:
  - `ci:test` - should run tests, linter, code coverage, etc.
  - `ci:build` - should perform any compilation (e.g. `tsc` output)
- `docker-compose.yml` file with the following setup:
  - defined service (e.g. `myservice`) that includes `build` with a `BUILD_VERSION` argument
- `Dockerfile` with `BUILD_VERSION` arg defined

You can then use this action with the [push event](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#push) like this:

```yaml
- name: Build app
  id: build
  uses: modiohealth/action-builder@next
  with:
    basePath: /path/to/app
    dockerServiceName: myservice
    dockerRepository: ${{ secrets.DOCKER_REPO }}
    githubToken: ${{ secrets.GITHUB_TOKEN }}
```

The following outputs are available (via `${{steps.<STEP_ID>.outputs.*}}`):
- `dockerTag`: the docker image that was built (if `pushEnabled` set to `true`)
- `deploymentIds`: GitHub deployment ids (if `deployEnabled` set to `true`, comma separated)

## core configuration
You can customize this action with the following parameters.

| Required | Parameter | Description | Default Value |
|----------|-----------|-------------|---------------|
|x| `basePath` | Base path to the application code | `'.'` |
|x| `githubToken` | GitHub token |  `null` |

### test configuration
Runs any tests that are part of `npm run ci:test`. An exit code of `0` means all tests passed. Examples of tests that would be run here:
- Linting (`tslint`, `eslint`)
- Jest
- Webdriver

| Parameter | Description | Default Value |
|-----------|-------------|---------------|
| `testEnabled` | Run tests | `true` |
| `testAction` | Default npm script to run CI tests | `ci:test` |

### build configuration
Building is split between 2 steps:
- Running `npm run ci:build` to generate any project specific artifacts required by docker (e.g. `tsc`). This must be defined and return a `0` exit code to be considered successful
- Running `docker-compose build <dockerServiceName>`. This should return a `0` exit code

| Parameter | Description | Default Value |
|-----------|-------------|---------------|
| `buildEnabled` | Run build  | `true` |
| `buildAction` | Default build action | `ci:build` |
| `dockerServiceName` | Docker compose service name to be built | `null` |

### push configuration
Once the associated service container is built it can be pushed to the defined `dockerRepository`. The docker tag used will be autogenerated and available as the output `dockerTag`.

| Parameter | Description | Default Value |
|-----------|-------------|---------------|
| `pushEnabled` | Upload docker container to repository | `true` |
| `dockerRepository` | Docker  | Docker repository to push built container to | `null` |

### deploy configuration
This creates a GitHub deployment using the [deployment api](https://docs.github.com/en/rest/reference/repos#deployments). Note that all deployments must include a `deploymentGroupId` parameter (allows multiple deployment actions to respond to the same deployment id).

| Required | Parameter | Description | Default Value |
|----------|-----------|-------------|---------------|
| `deployEnabled` | Perform GitHub deployment for the built container | `false` |
| `deployApps` | Apps to deploy with this build (comma separated) | `null` |


## conventions
This action follows these conventions.

### npm
The following `npm` scripts should be defined for any projects using this action:
- `ci:test`: should run any tests needed for a project (linting, unit tests, etc.)
- `ci:build`: should build any artifacts needed by `docker-compose build <service_name>` and return exit code `0`

### docker
This action uses docker-compose to build corresponding services. Doing this allows developers to define development environments that are a combination of the service being developed plus any dependent services.

_EXAMPLE:_ You are developing an API service that relies on a MySQL database and Redis cache. Wen developing your service you'll have both a `docker-compose.yml` and `Dockerfile` to build an API container as well as any dependent services that you'll need for local development.


`docker-compose.yml`:
```yaml
version: "3"
services:
  db:
    image: mysql:5.7
    command: --default-authentication-plugin=mysql_native_password
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: dummy
      MYSQL_USER: admin
      MYSQL_PASSWORD: admin
    ports:
      - "6000:3306"
  redis:
    image: redis:alpine
    ports:
      - '6001:6379'
  api:
    image: api:latest
    build:
      context: .
      args:
        - BUILD_VERSION=local
    depends_on:
      - db
      - redis
    ports:
      - "3000:3000"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - DATABASE_HOST=db
      - DATABASE_PORT=3306
      - PORT=3000
```

`Dockerfile`:
```
FROM node:12 as build
ENV LDFLAGS="-static-libgcc -static-libstdc++"
WORKDIR /app
COPY package* ./
RUN npm install --production
COPY dist .

FROM node:12-alpine
WORKDIR /app
COPY --from=build /app .
ARG BUILD_VERSION
ENV PORT=3000
EXPOSE 3000
ENTRYPOINT ["node", "main.js"]
```

In the above sample files you'll see that this example project defines 3 "services": `db`, `redis`, and `api`. Only `api` needs to be built and deployed (denoted by the presence of `build.context` param). The `api` service also defines an additional build argument called `BUILD_VERSION`. This argument is automatically set to the correct value by this action.

To use the above definitions by this action you'd set the following parameters for this action:
- `dockerServiceName` = `api`

_IMPORTANT_: this action is designed to build and test a single service. If you define multiple services you'll simply need to use this service for each corresponding service that you want to build.

### docker tagging
Docker images are tagged based on branch name and commit SHA. Every build from this action results in 2 tags:
- Explicit Tag: this is useful when using in a non-local environment (e.g. deploying to kubernetes) where a tag like `latest` isn't as meaningful as `master-1234567`. This tag is computed as `<branch_name>-<git commit sha, first 7 characters` (always lowercased)
- Shortcut Tag: this is useful when working locally where you don't want to update the image tag every time there is a new image built (you'll need to run `docker pull` or equivalent though). The following naming scheme is used depending on the branch:
  - `latest`: this comes from the `master` branch
  - `next`: this comes from the `develop` branch
  - `<branch_name>`: this comes from all other branches (always lowercased). For example if you have a branch called `feat-1234` then the tag will be `feat-1234`

## contributing
The default version of this action is currently hardcoded to `v1`. If action parameter names are changed, removed, or required new parameters are added then the action version will need to be updated to prevent any dependencies from breaking. Basically, be mindful if making changes to `action.yml`.

See `.github/workflows` for build and publishing instructions.

_NOTE_: GitHub actions does not currently support private actions to be shared across projects. No sensitive information should be hardcoded in this action.

Some useful documetation for creating actions:
- [`action.yml` documentation](https://help.github.com/en/articles/metadata-syntax-for-github-actions)
- [GitHub webhook event types](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/events-that-trigger-workflows#webhook-events)
- [TypeScript definitions for webhooks](https://github.com/octokit/webhooks.js)
