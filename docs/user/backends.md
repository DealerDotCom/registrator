# Backend Reference

Registrator supports a number of backing registries. In order for Registrator to
be useful, you need to be running one of these. Below are the Registry URIs to
use for supported backends and documentation specific to their features.

See also [Contributing Backends](../dev/backends.md).

## Consul

	consul://<address>:<port>
	consul-unix://<filepath>

Consul is the recommended registry since it specifically models services for
service discovery with health checks.

If no address and port is specified, it will default to `127.0.0.1:8500`.

Consul supports tags but no arbitrary service attributes.

### Consul HTTP Check

This feature is only available when using Consul 0.5 or newer. Containers
specifying these extra metadata in labels or environment will be used to
register an HTTP health check with the service.

```bash
SERVICE_80_CHECK_HTTP=/health/endpoint/path
SERVICE_80_CHECK_INTERVAL=15s
SERVICE_80_CHECK_TIMEOUT=1s		# optional, Consul default used otherwise
```

It works for services on any port, not just 80. If its the only service,
you can also use `SERVICE_CHECK_HTTP`.

### Consul Script Check

This feature is tricky because it lets you specify a script check to run from
Consul. If running Consul in a container, you're limited to what you can run
from that container. For example, curl must be installed for this to work:

```bash
SERVICE_CHECK_SCRIPT=curl --silent --fail example.com
```

The default interval for any non-TTL check is 10s, but you can set it with
`_CHECK_INTERVAL`. The check command will be interpolated with the `$SERVICE_IP`
and `$SERVICE_PORT` placeholders:

```bash
SERVICE_CHECK_SCRIPT=nc $SERVICE_IP $SERVICE_PORT | grep OK
```

### Consul TTL Check

You can also register a TTL check with Consul. Keep in mind, this means Consul
will expect a regular heartbeat ping to its API to keep the service marked
healthy.

```bash
SERVICE_CHECK_TTL=30s
```

## Consul KV

	consulkv://<address>:<port>/<prefix>
	consulkv-unix://<filepath>:/<prefix>

This is a separate backend to use Consul's key-value store instead of its native
service catalog. This behaves more like etcd since it has similar semantics, but
currently doesn't support TTLs.

If no address and port is specified, it will default to `127.0.0.1:8500`.

Using the prefix from the Registry URI, service definitions are stored as:

	<prefix>/<service-name>/<service-id> = <ip>:<port>

## Etcd

	etcd://<address>:<port>/<prefix>

Etcd works similar to Consul KV, except supports service TTLs. It also currently
doesn't support service attributes/tags.

If no address and port is specified, it will default to `127.0.0.1:2379`.

Using the prefix from the Registry URI, service definitions are stored as:

	<prefix>/<service-name>/<service-id> = <ip>:<port>

## SkyDNS 2

	skydns2://<address>:<port>/<domain>

SkyDNS 2 uses etcd, so this backend writes service definitions in a format compatible with SkyDNS 2.
The path may not be omitted and must be a valid DNS domain for SkyDNS.

If no address and port is specified, it will default to `127.0.0.1:2379`.

Using a Registry URI with the domain `cluster.local`, service definitions are stored as:

	/skydns/local/cluster/<service-name>/<service-id> = {"host":"<ip>","port":<port>}

SkyDNS requires the service ID to be a valid DNS hostname, so this backend requires containers to
override service ID to a valid DNS name. Example:

	$ docker run -d --name redis-1 -e SERVICE_ID=redis-1 -p 6379:6379 redis

## Zookeeper Store

The Zookeeper backend lets you publish ephemeral znodes into zookeeper. This mode is enabled by specifying a zookeeper path.  The zookeeper backend supports publishing a json znode body complete with defined service attributes/tags as well as the service name and container id. Example URIs:

	$ registrator zookeeper://zookeeper.host/basepath
	$ registrator zookeeper://192.168.1.100:9999/basepath

Within the base path specified in the zookeeper URI, registrator will create the following path tree containing a JSON entry for the service:

	<service-name>/<service-port> = <JSON>

The JSON will contain all infromation about the published container service. As an example, the following container start:

     docker run -i -p 80 -e 'SERVICE_80_NAME=www' -t ubuntu:14.04 /bin/bash

Will result in the zookeeper path and JSON znode body:

    /basepath/www/80 = {"Name":"www","IP":"192.168.1.123","PublicPort":49153,"PrivatePort":80,"ContainerID":"9124853ff0d1","Tags":[],"Attrs":{}}

## F5 BigIP Pool

F5 BigIP support uses the iControlREST API to add/remove/create nodes and pools in [Local Traffic Manager](https://f5.com/products/modules/local-traffic-manager). Pools will be named with the service name and nodes will be named with the service id minus the port. For now all configuration must live in the Common partition.

Example URI:

	$ registrator bigip://user:password@my-ltm-address

Starting a container with a single exposed port will produce the following:

	Pool: <service-name>
	    Pool Member: <service-id>:<external port>
	Node: <host>_<container name>
	    Address: <ip>
	
Starting a container with multiple ports will produce a pool per exposed port:

	Pool: <service-name>-<internal port 1>
	    Pool Member: <service-id>:<external port 1>
   	Pool: <service-name>-<internal port 2>
	    Pool Member: <service-id>:<external port 2>
	Node: <host>_<container name>
	    Address: <ip>
	    
Concrete example:

	$ docker run -p 45678:80 -p 45679:443 nginx
	
Results in

	Pool: nginx-80
	      |
	      +- Pool Member: node01_nginx:45678
	         |
	         +- node01_nginx: 172.17.0.40
	Pool: nginx-443
	      |
	      +- Pool Member: node01_nginx:45679
	         |
	         +- node01_nginx: 172.17.0.40

It is possible to control which containers are added as a node in LTM by requring that an explicit `SERVICE_NAME` be defined. This will cause registartor to ignore any containers missing this property.

	$ docker run -e REQUIRE_NAME=true registrator bigip://user:password@my-ltm
