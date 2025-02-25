# Supported tags and respective `Dockerfile` links

See [Docker Hub](https://hub.docker.com/_/solr?tab=tags) for a list of image tags available to pull.
The currently supported tags can be found in [./TAGS](https://github.com/docker-solr/docker-solr/blob/389e7844c8405605a930fc30cc8029eb6027798e/TAGS).

For more information about this image and its history, please see [the relevant manifest file (`library/solr`)](https://github.com/docker-library/official-images/blob/master/library/solr). This image is updated via pull requests to [the `docker-solr/docker-solr` GitHub repo](https://github.com/docker-solr/docker-solr).

For detailed information about the virtual/transfer sizes and individual layers of each of the above supported tags, please see [the `solr/tag-details.md` file](https://github.com/docker-library/docs/blob/master/solr/tag-details.md) in [the `docker-library/docs` GitHub repo](https://github.com/docker-library/docs).

# What is Apache Solr™?

Apache Solr is highly reliable, scalable and fault tolerant, providing distributed indexing, replication and load-balanced querying, automated failover and recovery, centralized configuration and more. Solr powers the search and navigation features of many of the world's largest internet sites.

Learn more on [Solr's homepage](http://lucene.apache.org/solr/) and in the [Solr Reference Guide](https://solr.apache.org/guide/).

![logo](https://raw.githubusercontent.com/docker-library/docs/master/solr/logo.png)

# Getting started with the Docker image

Instructions below apply to `solr:8.0.0` and above.

## Running Solr with host-mounted directories

Typically users first want to run a single standalone Solr server in a container, with a single core for data, while storing data in a local directory.
This is a convenient mechanism for developers, and could be used for single-server production hosts too.

```console
$ mkdir solrdata
$ sudo chown 8983:8983 solrdata  # necessary on Linux, not Mac.
$ docker run -d -v "$PWD/solrdata:/var/solr" -p 8983:8983 --name my_solr solr:8 solr-precreate gettingstarted
```

Then with a web browser go to `http://localhost:8983/` to see the Admin Console (adjust the hostname for your docker host).
In the web UI if you click on "Core Admin" you should now see the "gettingstarted" core.

Next load some of the example data that is included in the container:

```console
$ docker exec -it my_solr post -c gettingstarted example/exampledocs/manufacturers.xml
```

In the UI, find the "Core selector" popup menu and select the "gettingstarted" core, then select the "Query" menu item. This gives you a default search for `*:*` which returns all docs. Hit the "Execute Query" button, and you should see a few docs with data. Congratulations!


# Single server with Docker-compose

You can use Docker Compose to run a single standalone server.
And you could use Docker Volumes instead of host-mounted directories.
For example, with a `docker-compose.yml` containing the following:

```yaml
version: '3'
services:
  solr:
    image: solr:8
    ports:
     - "8983:8983"
    volumes:
      - data:/var/solr
    command:
      - solr-precreate
      - gettingstarted
volumes:
  data:
```

you can simply run:

```console
docker-compose up -d
```

## Single-command demo

For quick demos of docker-solr, there is a single command that starts Solr, creates a collection called "demo", and loads sample data into it:

```console
$ docker run --name solr_demo -d -p 8983:8983 solr:8 solr-demo
```
## Distributed Solr

See [this example](https://github.com/docker-solr/docker-solr-examples/blob/master/docker-compose/docker-compose.yml)
for an example Docker Compose file that starts up Solr in a simple cluster configuration.

# How the image works

The container contains an install of Solr, as installed by the [service installation script](https://solr.apache.org/guide/8_9/taking-solr-to-production.html#service-installation-script).
This stores the Solr distribution in `/opt/solr`, and configures Solr to use `/var/solr` to store data and logs, using the `/etc/default/solr` file for configuration.
If you want to persist the data, mount a volume or directory on `/var/solr`.
Solr expects some files and directories in `/var/solr`; if you use your own directory or volume you can either pre-populate them, or let docker-solr copy them for you. See [init-var-solr](scripts/init-var-solr).
If you want to use custom configuration, mount it in the appropriate place. See below for examples.

The docker-solr distribution adds [scripts](https://github.com/docker-solr/docker-solr/tree/master/scripts) in `/opt/docker-solr/scripts` to make it easier to use under Docker, for example to create cores on container startup.

## Creating cores

When Solr runs in standalone mode, you create "cores" to store data. On a non-Docker Solr, you would run the server in the background, then use the [Solr control script](https://solr.apache.org/guide/8_9/solr-control-script-reference.html) to create cores and load data. With Docker-solr you have various options.

### Manually

The first is exactly the same: start Solr running in a container, then execute the control script manually in the same container:

```console
$ docker run -d -p 8983:8983 --name my_solr solr:8 
$ docker exec -it my_solr solr create_core -c gettingstarted
```

This is not very convenient for users, and makes it harder to turn it into configuration for Docker Compose and orchestration tools like Kubernetes.

### Using solr-precreate command

So, typically you will use the `solr-precreate` command which prepares the specified core and then runs Solr:

```console
$ docker run -d -p 8983:8983 --name my_solr solr:8 solr-precreate gettingstarted
```

The `solr-precreate` command takes an optional extra argument to specify a configset directory below `/opt/solr/server/solr/configsets/` or you can specify full path to a custom configset 
inside the container:

```console
$ docker run -d -p 8983:8983 --name my_solr -v $PWD/config/solr:/my_core_config/conf solr:8 solr-precreate my_core /my_core_config
```

N.B. When specifying the full path to the configset, the actual core configuration should be located inside that directory in the `conf` directory. See <https://solr.apache.org/guide/8_9/config-sets.html> for details.

### Using solr-create command

The third option is to use the `solr-create` command. This runs a Solr in the background in the container, then uses the Solr control script to create the core, then stops the Solr server and restarts it in the foreground. This method is less popular because the double Solr run can be confusing.

```console
$ docker run -d -p 8983:8983 --name my_solr solr:8 solr-create -c gettingstarted
```

### Custom set-up scripts

Finally, you can run your own command-line and specify what to do, and even invoke mounted scripts. For example:

```console
$ docker run -p 8983:8983 -v $PWD/mysetup.sh:/mysetup.sh --name my_solr solr:8 bash -c "precreate-core gettingstarted && source /mysetup.sh && solr-foreground"
```

## Creating collections

In a "SolrCloud" cluster you create "collections" to store data; and again you have several options for creating a core.

These examples assume you're running [this example cluster](https://github.com/docker-solr/docker-solr-examples/blob/master/docker-compose/docker-compose.yml)

The first way to create a collection is to go to the [Solr Admin UI](http://localhost:8983/), select "Collections" from the left-hand side navigation menu, then press the "Add Collection" button, give it a name, select the `_default` config set, then press the "Add Collection" button.

The second way is through the Solr control script on one of the containers:

```console
$ docker exec solr1 solr create -c gettingstarted2
```

The third way is to use a separate container:

```console
$ docker run -e SOLR_HOST=solr1 --network docs_solr solr solr create_collection -c gettingstarted3 -p 8983
```

The fourth way is to use the remote API, from the host or from one of the containers, or some new container on the same network (adjust the hostname accordingly):

```console
curl 'http://localhost:8983/solr/admin/collections?action=CREATE&name=gettingstarted3&numShards=1&collection.configName=_default'
```

If you want to use a custom config for your collection, you first need to upload it, and then refer to it by name when you create the collection.
See the Ref guide on how to use the [ZooKeeper upload](https://solr.apache.org/guide/8_9/solr-control-script-reference.html) or the [configset API](https://solr.apache.org/guide/8_9/configsets-api.html).


## Loading your own data

There are several ways to load data; let's look at the most common ones.

The most common first deployment is to run Solr standalone (not in a cluster), on a workstation or server, where you have local data you wish to load.
One way of doing that is using a separate container, with a mounted volume containing the data, using the host network so you can connect to the mapped port:

```console
# start Solr. Listens on localhost:8983
$ docker run --name my_solr -p 8983:8983 solr:8 solr-precreate books

# get data
$ mkdir mydata
$ wget -O mydata/books.csv https://raw.githubusercontent.com/apache/lucene-solr/master/solr/example/exampledocs/books.csv
$ docker run --rm -v "$PWD/mydata:/mydata" --network=host solr:8 post -c books /mydata/books.csv
```

If you use the [this example cluster](https://github.com/docker-solr/docker-solr-examples/blob/master/docker-compose/docker-compose.yml) the same works, or you can just start your loading container in the same network:

```console
$ docker run -e SOLR_HOST=solr1 --network=mycluster_solr solr solr create_collection -c books -p 8983
$ docker run --rm -v "$PWD/mydata:/mydata" --network=mycluster_solr solr:8 post  -c books /mydata/books.csv -host solr1
```

Alternatively, you can make the data available on a volume at Solr start time, and then load it from `docker exec` or a custom start script.


## solr.in.sh configuration

In Solr it is common to configure settings in [solr.in.sh](https://github.com/apache/solr/blob/HEAD/solr/bin/solr.in.sh),
as documented in the [Solr Reference Guide](https://solr.apache.org/guide/8_9/taking-solr-to-production.html#environment-overrides-include-file).

The `solr.in.sh` file can be found in `/etc/default`:

```console
$ docker run  solr:8 cat /etc/default/solr.in.sh
```

It has various commented-out values, which you can override when running the container, like:

```
$ docker run -d -p 8983:8983 -e SOLR_HEAP=800m solr
```

You can also mount your own config file. Do no modify the values that are set at the end of the file.

## Extending the image

The docker-solr image has an extension mechanism. At run time, before starting Solr, the container will execute scripts
in the `/docker-entrypoint-initdb.d/` directory. You can add your own scripts there either by using mounted volumes
or by using a custom Dockerfile. These scripts can for example copy a core directory with pre-loaded data for continuous
integration testing, or modify the Solr configuration.

Here is a simple example. With a `custom.sh` script like:

```console
#!/bin/bash
set -e
echo "this is running inside the container before Solr starts"
```

you can run:

```console
$ docker run --name solr_custom1 -d -v $PWD/custom.sh:/docker-entrypoint-initdb.d/custom.sh solr
$ sleep 5
$ docker logs solr_custom1 | head
/opt/docker-solr/scripts/docker-entrypoint.sh: running /docker-entrypoint-initdb.d/set-heap.sh
this is running inside the container before Solr starts

Starting Solr on port 8983 from /opt/solr/server
```

With this extension mechanism it can be useful to see the shell commands that are being executed by the `docker-entrypoint.sh`
script in the docker log. To do that, set an environment variable using Docker's `-e VERBOSE=yes`.

Instead of using this mechanism, you can of course create your own script that does setup and then call `solr-foreground`, mount that script into the container, and execute it as a command when running the container.

Other ways of extending the image are to create custom Docker images that inherit from this one.

## Debugging with jattach

The `jcmd`, `jmap` `jstack` tools can be useful for debugging Solr inside the container. These tools are not included with the JRE, but this image includes the [jattach](https://github.com/apangin/jattach) utility which lets you do much of the same.

    Usage: jattach <pid> <cmd> [args ...]
    
      Commands: 
        load : load agent library
        properties : print system properties
        agentProperties : print agent properties
        datadump : show heap and thread summary
        threaddump : dump all stack traces (like jstack)
        dumpheap : dump heap (like jmap)
        inspectheap : heap histogram (like jmap -histo)
        setflag : modify manageable VM flag
        printflag : print VM flag
        jcmd : execute jcmd command
    
Example comands to do a thread dump and get heap info for PID 10:

    jattach 10 threaddump
    jattach 10 jcmd GC.heap_info

# Updating from Docker-solr5-7 to 8

For Solr 8, the docker-solr distribution switched from just extracting the Solr tar, to using the [service installation script](https://solr.apache.org/guide/8_9/taking-solr-to-production.html#service-installation-script). This was done for various reasons: to bring it in line with the recommendations by the Solr Ref Guide, to make it easier to mount volumes, and because we were [asked to](https://github.com/docker-solr/docker-solr/issues/173).

This is a backwards incompatible change, and means that if you're upgrading from an older version, you will most likely need to make some changes. If you don't want to upgrade at this time, specify `solr:7` as your container image. If you use `solr:8` you will use the new style. If you use just `solr` then you risk being tripped up by backwards incompatible changes; always specify at least a major version.

Changes:

- The Solr data is now stored in `/var/solr/data` rather than `/opt/solr/server/solr`. The `/opt/solr/server/solr/mycores` no longer exists
- The custom `SOLR_HOME` can no longer be used, because various scripts depend on the new locations. Consequently, `INIT_SOLR_HOME` is also no longer supported.

# Running under tini

The docker-solr image runs Solr under [tini](https://github.com/krallin/tini), to make signal handling work better; in particular, this allows you to `kill -9` the JVM. If you run `docker run --init`, or use `init: true` in `docker-compose.yml`, or have added `--init` to `dockerd`, docker will start its `tini` and docker-solr will notice it is not PID 1, and just `exec` Solr. If you do not run with `--init`, then the docker entrypoint script detects that it is running as PID 1, and will start the `tini` present in the docker-solr image, and run Solr under that. If you really do not want to run `tini`, and just run Solr as PID 1 instead, then you can set the `TINI=no` environment variable.

# Out of memory handling

You can use the `OOM` environment variable to control the behaviour of the Solr JVM when an out-of-memory error occurs.
If you specify `OOM=exit`, docker-solr will add `-XX:+ExitOnOutOfMemoryError` to the JVM arguments, so that the JVM will exit.
If you specify `OOM=crash`, docker-solr will add `-XX:+CrashOnOutOfMemoryError` to the JVM arguments, so the JVM will crash and produces text and binary crash files (if core files are enabled).
If you specify `OOM=script`, docker-solr will add `-XX:OnOutOfMemoryError=/opt/docker-solr/scripts/oom_solr.sh`, so the JVM will run that script (and if you want to you can mount your own in its place).

# About this repository

This repository is available on [github.com/docker-solr/docker-solr](https://github.com/docker-solr/docker-solr), and the official build is on the [Docker Hub](https://hub.docker.com/_/solr/).

# License

Solr is licensed under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).

This repository is licensed under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).

Copyright 2015-2020 The Apache Software Foundation

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

# Releasing

When a new version of Apache Solr is released, a new docker image can be released by following the procedure outlined in [update.md](./update.md). Normally you open a new issue to volunteer for the release, and then file a PR once you have completed all the steps and run the tests.

# User Feedback

## Issues

Please report issues with this docker image on this [Github project](https://github.com/docker-solr/docker-solr).

For general questions about Solr, see the [Community information](http://solr.apache.org/community.html), in particular the solr-user mailing list.

## Contributing

If you want to contribute to Solr, see the [How To Contribute](http://solr.apache.org/community.html#how-to-contribute).

# History

This project was started in 2015 by [Martijn Koster](https://github.com/makuk66). In 2019 maintainership and copyright was transferred to the Apache Lucene/Solr project. Many thanks to Martijn for all your contributions over the years!
