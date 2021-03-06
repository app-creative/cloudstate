# Building native images in Cloudstate

Cloudstate uses the GraalVM native image compiler to compile native images, to ensure that when a Cloudstate proxy starts due to autoscaling, it is immediately ready to handle load at full capacity, with no classloading or JIT overheads.

GraalVM native image builds can be very slow, taking significant resources on a machine (greater than 16GB), so to address this, Cloudstate supports the use of build agents running on GCP.

## Using GCP build agents

To use a GCP build agent, you need to have a GCP account and project, with `gcloud` logged in. It's recommended that you set your default project using `gcloud config set project <some-project>`, and your default availability zone to the zone that is geographically closest to you, using `gcloud config set compute/zone <us-east1>`, however if you prefer not to do this you can also select these by using the `GCP_AGENT_PROJECT` and `GCP_AGENT_ZONE` environment variables.

Once you have `gcloud` configured, start sbt with the environment variable `GCP_AGENT_ENABLED` set to `true`, for example:

```bash
GCP_AGENT_ENABLED=true sbt
```

When you do this, sbt will start the GCP agent as needed, and shut it down when you exit sbt. The instance name will default to `$USER-gcp-build-agent`, this can be overridden using the `GCP_INSTANCE_NAME` environment variable. If for some reason sbt doesn't shutdown gracefully, you may need to kill the instance yourself, using `gcloud compute instances delete <instance-name>`. By default, preemptible instances will be used, this means Google will automatically shut down the instance after 24 hours if you don't.

Since instances are started by sbt, and then shutdown when you exit sbt, it is strongly advised that you use an sbt shell session, rather than running individual commands. That is, instead of running:

```bash
GCP_AGENT_ENABLED=true sbt proxy-core/docker:publish
GCP_AGENT_ENABLED=true sbt proxy-postgres/docker:publish
```

Run:

```bash
GCP_AGENT_ENABLED=true sbt
```

Then once in the sbt shell, run:

```bash
proxy-core/docker:publish
proxy-postgres/docker:publish
```

This way, the same instance will be reused, rather than having to pay the cost of starting up and shutting down the instance each time. Note also that it's not just the cost of starting up and shutting down the instance that you're paying, on the first build, all docker base images will need to be fetched, none of your existing layers will be cached, and the Docker context won't be in the cache either. Subsequent runs within the same sbt session will have a significant performance improvement as these caches will all be warmed.

## Building the images

In our build, we have multiple different images that we want to build from the same project - both native and non native images, along with different configurations. To support this, we need to reconfigure sbt for each image we want to build. To support this, we have a number of command helpers that dynamically configure sbt, then run a selected command. These are:

* `dockerBuildInMemory`
* `dockerBuildNativeInMemory`
* `dockerBuildCassandra`
* `dockerBuildNativeCassandra`
* `dockerBuildNoStore`
* `dockerBuildNativeNoStore`
* `dockerBuildPostgres`
* `dockerBuildNativePostgres`
* `dockerBuildAllNonNative`
* `dockerBuildAllNative`

### Just building an image

To just build a native image, run the selected command above, followed by `publishLocal`, eg:

```
dockerBuildNativeInMemory publishLocal
```

Contrary to the name of the command, this will build the image remotely if you have that enabled - the reason for this is that the remote build agent support doesn't redefine any of the sbt docker plugin commands, it simply configures the sbt docker plugin to run against the remote docker daemon. So `publishLocal` usually means just build an image in the docker daemon, but don't push it, and since our docker daemon happens to be running remotely through ssh port forwarding, it's done remotely.

Just building the image isn't very useful if you're using the remote agent, since then your image is in the remote agents docker image cache, which is not a very useful place for it to be.

### Building and pushing an image

If you wish to push an image to a public repository somewhere, for example to test it in the cloud, then instead use `publish`. Note that unless you're pushing to the official `cloudstateio` Docker hub repository, you will want to override the default registry/username to something else, this can be done by setting system properties on sbt startup:

```bash
GCP_AGENT_ENABLED=true sbt -Ddocker.registry=gcr.io -Ddocker.username=mygcrproject
```

Now you can build and push an image to that repo by running:

```
dockerBuildNativeInMemory publish
```

### Building and pulling an image

If you want to pull the image to make it available to your local docker daemon so you can do some testing locally on your machine, a `publishLocalPull` command is provided. When you run this, after building the image, it will push it to a registry running on the remote agent. Then it will do a `docker pull` using the local Docker daemon, from the registry running on the remote agent. The reason for doing this `push`/`pull` behavior, rather than doing a `save` followed by `load`, is that the save/load exports the full image as a tarball, meaning every time you run it, you have to download every layer including the base image. In contrast, the `push`/`pull` only pulls layers that you don't already have in your local docker daemon, the result being that you only download the native image layer, not the entire image including the base image.

## Starting the native image locally

Since the native image needs to call back into the host to connect to the user function, the user function needs to be told to bind to `0.0.0.0`. It will depend on the user function as to how you do this, but here's an example, run from a node user function:

```
HOST=0.0.0.0 node index.js
```

Now, run the docker image. If using Mac or Windows, you can run:

```
docker run --rm -e USER_FUNCTION_HOST=host.docker.internal -p 9000:9000 \
  cloudstateio/cloudstate-proxy-native-dev-mode:latest
```

If on Linux, there's no automatic way to know what the hosts IP address is, so instead:

```
docker run --rm -e USER_FUNCTION_HOST=$(ip -4 addr show docker0 | grep -Po 'inet \K[\d.]+') \
  -p 9000:9000 cloudstateio/cloudstate-proxy-native-dev-mode:latest
```

## Running the TCK against the native image

To run the TCK against the native image, start sbt with the `config.resource` property set to `native-image.conf`, eg:

```bash
GCP_AGENT_ENABLED=true sbt -Dconfig.resource=native-image.conf
```

Now after building the dev mode native image, you can run the integration tests:

```
dockerBuildNativeDevMode publishLocalPull
tck/it:test
```

## Dealing with preemptible issues

By default, the agent build uses preemptible instances, which are about 5 times cheaper than on demand instances. The downside of this, sometimes, the instances will be terminated unexpectedly. To avoid this, it's recommended that you set your default compute zone to one that is in lower demand. For example, in a given region, zone a is typically in highest demand, so zone b or c is often a better choice. Also, some regions have high demand than others, `us-east1` for example is probably a bad choice. Be aware that by default, N2 instances are used, and these are not available in every zone. The machine type can be changed using the `GCP_MACHINE_TYPE` environment variable.

If you still have issues with instances being terminated, you can switch to on demand instances by setting the `GCP_PREEMPTIBLE` environment variable to `false`.

When dealing with preemptible instances, there are a few commands that can be run to help. `gcpAgentStatus` will tell you the status of the agent. If `Terminated`, this indicates that the instance has been preemptively terminated. When this has happened, run `gcpAgentReset`, this will restart the agent and port forwarding if necessary. The agent can be manually managed using `gcpAgentStart`, `gcpAgentStop` and `gcpAgentPortForward`.
