<img src="docs/images/banner.png"/>

# score-compose

`score-compose` is a reference implementation of the [Score specification](https://github.com/score-spec/spec) for [Docker compose](https://docs.docker.com/compose/), primarily used for local development. It supports most aspects of the Score specification and provides a powerful resource provisioning system for supplying and customising the dynamic configuration of attached services such as databases, queues, storage, and other network or storage APIs.

## Feature support

`score-compose` supports as many Score features as possible, however there are certain parts that don't fit well in a local Docker case and are not supported:

| Feature                                                             | Support | Impact                                                                                                                                                                                                                                                                                                                                              |
|---------------------------------------------------------------------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `containers.*.resources.limits` / `containers.*.resources.requests` | none    | **Limits will be validated but ignored.** While the compose specification has some support for this, it is requires particular Docker versions that cannot be relied on. *This should have no impact on Workload execution*.                                                                                                                        |
| `containers.*.livenessProbe` / `containers.*.readinessProbe`        | none    | **Probes will be validated but ignored.** The Score specification only details K8s-like HTTP probes, but the compose specification only supports direct command execution. We cannot convert between the two reliably. *This should have no impact on Workload execution*. Tracked in [#86](https://github.com/score-spec/score-compose/issues/86). |
| `containers.*.volumes[*].path`                                      | none    | **Volume mounts with `path` will be rejected.** Docker compose doesn't support sub-path mounts for Docker volumes.                                                                                                                                                                                                                                  |

## Resource support

`score-compose` comes with out-of-the-box support for:

| Type          | Class   | Params                 | Output                                                                                                                                                          |
| ------------- | ------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| environment   | default | (none)                 | `${KEY}`                                                                                                                                                        |
| service-port  | default | `workload`, `port`     | `hostname`, `port`                                                                                                                                              |
| volume        | default | (none)                 | `source`                                                                                                                                                        |
| redis         | default | (none)                 | `host`, `port`, `username`, `password`                                                                                                                          |
| postgres      | default | (none)                 | `host`, `port`, `name` (aka `database`), `username`, `password`                                                                                                 |
| mysql         | default | (none)                 | `host`, `port`, `name` (aka `database`), `username`, `password`                                                                                                 |
| s3            | default | (none)                 | `endpoint`, `access_key_id`, `secret_key`, `bucket`, with `region=""`, `aws_access_key_id=<access_key_id>`, and `aws_secret_key=<secret_key>` for compatibility |
| dns           | default | (none)                 | `host`                                                                                                                                                          |
| route         | default | `host`, `path`, `port` |                                                                                                                                                                 |
| amqp          | default | (none)                 | `host`, `port`, `vhost`, `username`, `password`                                                                                                                 |
| mongodb       | default | (none)                 | `host`, `port`, `username`, `password`, `connection`                                                                                                            |
| kafka-topic   | default | (none)                 | `host`, `port`, `name`, `num_partitions`                                                                                                                        |
| elasticsearch | default | (none)                 | `host`, `port`, `username`, `password`                                                                                                                          |

These can be found in the default provisioners file. You are encouraged to write your own provisioners and add them to the `.score-compose` directory (with the `.provisioners.yaml` extension) or contribute them upstream to the [default.provisioners.yaml](internal/command/default.provisioners.yaml) file.

## Installation

To install `score-compose`, follow the instructions as described in our [installation guide](https://docs.score.dev/docs/score-implementation/score-compose/#installation). You will also need a recent version of Docker and the Compose plugin installed. Read more [here](https://docs.docker.com/compose/install/).

## Get started

**NOTE**: the following examples and guides relate to `score-compose >= 0.11.0`, check your version using `score-compose --version` and re-install if you're behind!

See the [examples](./examples) for more examples of using Score and provisioning resources:

- [01-hello](examples/01-hello) - a basic example of a Score Workload
- [02-environment](examples/02-environment) - an example of environment variables and the `type: environment` resource
- [03-files](examples/03-files) - mounting local files into the running Workload
- [04-multiple-workloads](examples/04-multiple-workloads) - examples of multiple containers and workloads together
- [05-volume-mounts](examples/05-volume-mounts) - an example of an "empty-dir" volume resource with `type: volume`
- [06-resource-provisioning](examples/06-resource-provisioning) - detailed example and information about resource provisioning and the operation of the `template://` and `cmd://` provisioners
- [07-overrides](examples/07-overrides) - details of how to override aspects of the input Score file and output Docker compose files
- [08-service-port-resource](examples/08-service-port-resource) - an example of using the `service-port` resource type to link between workloads
- [09-dns-and-route](examples/09-dns-and-route) - an example of using the `dns` and `route` resources to route http requests
- [10-amqp-rabbitmq](examples/10-amqp-rabbitmq) - an example the default `amqp` resource provisioner
- [11-mongodb-document-database](examples/11-mongodb-document-database) - an example the default `mongodb` resource provisioner
- [12-mysql-database](examples/12-mysql-database) - an example of the default `mysql` resource provisioner
- [13-kafka-topic](examples/13-kafka-topic) - an example of the default `kafka-topic` resource provisioner
- [14-elasticsearch](examples/14-elasticsearch) - an example of the default `elasticsearch` resource provisioner

If you're getting started, you can use `score-compose init` to create a basic `score.yaml` file in the current directory along with a `.score-compose/` working directory.

```
$ score-compose init --help
The init subcommand will prepare the current directory for working with score-compose and prepare any local
files or configuration needed to be successful.

A directory named .score-compose will be created if it doesn't exist. This file stores local state and generally should
not be checked into source control. Add it to your .gitignore file if you use Git as version control.

The project name will be used as a Docker compose project name when the final compose files are written. This name
acts as a namespace when multiple score files and containers are used.

Usage:
  score-compose init [flags]

Flags:
  -f, --file string      The score file to initialize (default "./score.yaml")
  -h, --help             help for init
  -p, --project string   Set the name of the docker compose project (defaults to the current directory name)

Global Flags:
      --quiet           Mute any logging output
  -v, --verbose count   Increase log verbosity and detail by specifying this flag one or more times
```

Once you have a `score.yaml` file created, modify it by following [this guide](https://docs.score.dev/docs/get-started/score-compose-hello-world/), and use `score-compose generate` to convert it into a Docker compose manifest:

```
The generate command will convert Score files in the current Score compose project into a combined Docker compose
manifest. All resources and links between Workloads will be resolved and provisioned as required.

By default this command looks for score.yaml in the current directory, but can take explicit file names as positional
arguments.

"score-compose init" MUST be run first. An error will be thrown if the project directory is not present.

Usage:
  score-compose generate [flags]

Examples:

  # Specify Score files
  score-compose generate score.yaml *.score.yaml

  # Regenerate without adding new score files
  score-compose generate

  # Provide overrides when one score file is provided
  score-compose generate score.yaml --override-file=./overrides.score.yaml --override-property=metadata.key=value

Flags:
      --build stringArray               An optional build context to use for the given container --build=container=./dir or --build=container={"context":"./dir"}
      --env-file string                 Location to store a skeleton .env file for compose - this will override existing content
  -h, --help                            help for generate
      --image string                    An optional container image to use for any container with image == '.'
  -o, --output string                   The output file to write the composed compose file to (default "compose.yaml")
      --override-property stringArray   An optional set of path=key overrides to set or remove
      --overrides-file string           An optional file of Score overrides to merge in

Global Flags:
      --quiet           Mute any logging output
  -v, --verbose count   Increase log verbosity and detail by specifying this flag one or more times
```

**NOTE**: The `score-compose run` command still exists but is hidden and should be considered deprecated as it does not support resource provisioning.

## Testing

Run the tests using `go test -v ./... -race`. If you do not have `docker` CLI installed locally or want the tests to run
faster, consider setting `NO_DOCKER=true` to skip any `docker compose` based validation during testing.

## Get in touch

Connect with us through the [Score Slack channel](https://join.slack.com/t/scorecommunity/shared_invite/zt-2a0x563j7-i1vZOK2Yg2o4TwCM1irIuA) or contact us via email at team@score.dev.

We host regular community meetings to discuss updates, share ideas, and collaborate. Here are the details:

| Community call | Info |
|:-----------|:------------|
| Meeting Link | Join via [Google Meet](https://meet.google.com/znt-usdc-hzs) or call +49 40 8081618260 (Pin: 599 887 196)
| Meeting Agenda & Notes | Add to our agenda or review minutes [here](https://github.com/score-spec/spec/discussions/categories/community-meetings)
| Meeting Time | 1:00-2:00pm UTC, every first Thursday of the month

If you can't attend at the scheduled time but would like to discuss something, please reach out. We’re happy to arrange an ad-hoc meeting that fits your schedule.

### Contribution Guidelines and Governance

Our general contributor guidelines can be found in [CONTRIBUTING.md](CONTRIBUTING.md). Please note that some repositories may have additional guidelines. For more information on our governance model, please refer to [GOVERNANCE.md](https://github.com/score-spec/spec/blob/main/GOVERNANCE.md).

### Documentation

You can find our documentation at [docs.score.dev](https://docs.score.dev/docs).

### Roadmap

See [Roadmap](https://github.com/score-spec/spec/blob/main/roadmap.md). You can [submit an idea](https://github.com/score-spec/spec/blob/main/roadmap.md#get-involved) anytime.

### License

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

### Code of conduct

[![Contributor Covenant](https://img.shields.io/badge/Contributor%20Covenant-2.1-4baaaa.svg)](CODE_OF_CONDUCT.md)
