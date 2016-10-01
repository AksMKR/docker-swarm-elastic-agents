# GoCD Elastic agent plugin for Docker Swarm [![Build Status](https://snap-ci.com/gocd-contrib/docker-swarm-elastic-agents/branch/master/build_image)](https://snap-ci.com/gocd-contrib/docker-swarm-elastic-agents/branch/master)

## Building the code base

To build the jar, run `./gradlew clean test assemble`

## Is this production ready?

It depends.

The plugin, as it is currently implemented is meant to be a very simple plugin to demonstrate how to get started with GoCD [elastic agent](https://plugin-api.go.cd/current/elastic-agents) feature. This plugin terminates docker containers very aggressively (within a minute or two of the agent being idle). Depending on your usage, this may not be desirable. If this behavior is undesirable to you, you may need to fork this plugin and [tweak it a bit](https://github.com/gocd-contrib/docker-swarm-elastic-agents/blob/master/src/main/java/cd/go/contrib/elasticagents/docker/executors/ServerPingRequestExecutor.java) so the docker containers are not terminated as aggressively.

## Usage instructions

* Download and install Docker for your favorite OS from https://docs.docker.com/engine/installation/

If you already have it running it on a mac, make sure to restart it (see https://github.com/docker/for-mac/issues/17#mobyaccess). Time drift is known to cause the plugin to not work, because the timestamps returned by the docker API has drifted from the host.

A good way to know if there's a time drift is to run `docker ps` —

    ```
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
    e0754c9f4cdb        alpine:latest       "/bin/sh"           32 minutes ago      Up 17 seconds                           test
    809f310ba1e4        ubuntu:trusty       "/bin/bash"         33 minutes ago      Up About a minute                       reverent_raman
    ```

Notice how the `CREATED` and `STATUS` are several minutes apart for a recently created container.

* Download the latest GoCD installer from https://go.cd/download

    ```shell
    $ unzip go-server-VERSION.zip
    $ mkdir -p go-server-VERSION/plugins/external
    ```
* Download the docker plugin (https://github.com/gocd-contrib/docker-swarm-elastic-agents/releases)
* Copy the docker plugin to the go server directory

    ```
    $ cp build/libs/docker-swarm-elastic-agents-0.1-SNAPSHOT.jar /path/to/go-server-VERSION/plugins/external
    ```

* Start the server and configure the plugin (turn on debug logging to get more logs, they're not that noisy)

  On linux/mac

    ```shell
    $ GO_SERVER_SYSTEM_PROPERTIES='-Dplugin.cd.go.contrib.elastic-agent.docker-swarm.log.level=debug' ./server.sh
    ```

  On windows

    ```
    C:> set GO_SERVER_SYSTEM_PROPERTIES='-Dplugin.cd.go.contrib.elastic-agent.docker-swarm.log.level=debug'
    C:> server.cmd
    ```

To configure the plugin, navigate to the plugin settings page on your GoCD server http://localhost:8153/go/admin/plugins and setup the following settings for the docker plugin.

```
Go Server Host — https://YOUR_IP_ADDRESS:8154/go — do not use "localhost"
Docker URI (for mac and linux) — unix:///var/run/docker.sock
Auto register timeout - between 1-3 minutes
```

Now setup the config.xml —

* add `agentAutoRegisterKey="some-secret-key"` to the `<server/>` tag.
* setup a job —

```xml
<server agentAutoRegisterKey="...">
  <elastic>
    <profiles>
      <profile id="docker.unit-tests" pluginId="cd.go.contrib.elastic-agent.docker-swarm">
        <!-- The following properties are currently supported -->
        <property>
          <!-- Allows you to select the docker image that the build should run with -->
          <key>Image</key>
          <value>gocdcontrib/ubuntu-docker-elastic-agent</value>
        </property>
        <property>
          <!-- Allows you to set the environment variables when starting the docker container -->
          <key>Environment</key>
          <value>
            JAVA_HOME=/opt/java
            MAKE_OPTS=-j8
          </value>
        </property>
      </profile>
    </profiles>
  </elastic>
</server>
...
<pipelines group="defaultGroup">
  <pipeline name="Foo">
    <materials>
      <git url="YOUR GIT URL" />
    </materials>
    <stage name="defaultStage">
      <jobs>
        <job name="defaultJob" elasticProfileId="docker.unit-tests">
          <tasks>
            <exec command="ls" />
          </tasks>
        </job>
      </jobs>
    </stage>
  </pipeline>
</pipelines>
```

## Troubleshooting

Enabling debug level logging can help you troubleshoot an issue with the elastic agent plugin. To enable debug level logs, edit the `/etc/default/go-server` (for Linux) to add:

```bash
export GO_SERVER_SYSTEM_PROPERTIES="$GO_SERVER_SYSTEM_PROPERTIES -Dplugin.cd.go.contrib.elastic-agent.docker-swarm.log.level=debug"
```

If you're running the server via `./server.sh` script —

```
$ GO_SERVER_SYSTEM_PROPERTIES="-Dplugin.cd.go.contrib.elastic-agent.docker-swarm.log.level=debug" ./server.sh
```
