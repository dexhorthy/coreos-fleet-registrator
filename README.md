Coreos Registrator Fleet
==========================

systemd unit file for running [registrator](https://github.com/gliderlabs/registrator) on coreos/fleet using etcd as the backend registry.


The Registrator Service File
----------------------------

#### Unit Section

Pretty standard stuff, service name and dependency on docker:

```
[Unit]
Description=Registrator Service
Requires=docker.service
After=docker.service
```

#### Service Section

First line gives us access to the coreos environment,
particularly `COREOS_PUBLIC_IPV4`, which will be important
for configuring registrator properly.

```
EnvironmentFile=/etc/environment
```

Next, disable any start timeout

```
TimeoutStartSec=0
```

Next three lines set to kill, remove, and pull the container before starting

```
ExecStartPre=-/usr/bin/docker kill registrator
ExecStartPre=-/usr/bin/docker rm -f registrator
ExecStartPre=/usr/bin/docker pull gliderlabs/registrator:latest
```

Finally, we configure the `ExecStart` command for running the registrator container.

```
ExecStart=/usr/bin/docker run --name registrator --volume "/var/run/docker.sock:/tmp/docker.sock" gliderlabs/registrator:latest -ip ${COREOS_PUBLIC_IPV4} etcd://${COREOS_PUBLIC_IPV4}:2379/services/registrator
```

The interesting params here are:

- `--volume "/var/run/docker.sock:/tmp/docker.sock"` mounts the host's docker socket into `/tmp/docker.sock`, where registrator expects it to be.
- `-ip ${COREOS_PUBLIC_IPV4}` configures the ip at which registrator will advertise exposed ports
- `etcd://${COREOS_PUBLIC_IPV4}:2379/services/registrator` tells registrator to register services in etcd running on the local machine, with a prefix of `/services/registrator`



#### Fleet Configuration

We use fleet's `Conflicts` setting to ensure
we only have a most one instance of `registrator` running per host:

```
[X-Fleet]
Conflicts=registrator@*
```


Running the example
---------------------

**Prerequisite** This example assumes you've set up a coreos cluster using the [coreos-vagrant](https://github.com/coreos/coreos-vagrant) project.
For this example, we'll use three machines -- `num_instances=3` in the project's `config.rb`. We'll control the services from `core-01`, but any
of the booted vagrant machines will do.

#### 1. Get the service definitions

From the root of your checked-out and booted [coreos-vagrant](https://github.com/coreos/coreos-vagrant) project:

```sh
vagrant ssh core-01
```

from there:

```sh
wget -O registrator@.service https://raw.githubusercontent.com/horthy/coreos-registrator-fleet/master/registrator@.service
wget -O nginx@.service https://raw.githubusercontent.com/horthy/coreos-registrator-fleet/master/nginx@.service
```


#### 2. Run registrator

Still ssh'd into `core-01`, submit the registrator service to fleet

```sh
fleetctl start registrator@{1..3}
```

We can verify they started up with `fleetctl list-units`

#### 3. Run the simple nginx example

Again from `core-01`, run the simple example nginx service included:

```sh
fleetctl start nginx@{1..3}
```

Again, verify they're running with `fleetctl list-units`

#### 4. Verify etcd contents

From `core-01`

```sh
etcdctl ls --recursive /services/registrator
```

You should see something like


```
/services/registrator/nginx-443
/services/registrator/nginx-443/a26598a3823a:nginx:443
/services/registrator/nginx-443/74893804070f:nginx:443
/services/registrator/nginx-443/51d468b1a73d:nginx:443
/services/registrator/nginx-80
/services/registrator/nginx-80/a26598a3823a:nginx:80
/services/registrator/nginx-80/74893804070f:nginx:80
/services/registrator/nginx-80/51d468b1a73d:nginx:80
```

showing that both exposed nginx ports are registered on three instances in the cluster.

We can verify nginx is running with curl:

```sh
IP_AND_PORT=`etcdctl get /services/registrator/nginx-80/51d468b1a73d:nginx:80`
echo $IP_AND_PORT # Outputs something like 172.17.8.102:32769
curl -I $IP_AND_PORT
```

We can check the rest of the services programmatically:

```
for service in `etcdctl ls --recursive /services/registrator/nginx-80/`
do
curl -I `etcdctl get $service`
done
```
