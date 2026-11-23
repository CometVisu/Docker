Abstract base docker container for the CometVisu
================================================

This container can be used as a base for own Docker containers that contain the
[CometVisu](https://www.cometvisu.org/). It contains an Apache / PHP combo with
the knxd (0.0.5.1). Also, RRD support for the diagram plugin is implemented.

This container is available at DockerHub as 
[cometvisu/cometvisuabstractbase](https://hub.docker.com/r/cometvisu/cometvisuabstractbase/).

**NOTE:** This is just the abstract base for a CometVisu container. When you are
looking for a ready to use container of the CometVisu you should look at
[cometvisu/cometvisu](https://hub.docker.com/r/cometvisu/cometvisu/).

Environment parameters:
-----------------------

|Parameter              | Default                   |Description|
|-----------------------|---------------------------|-----------|
|BACKEND_NAME           | knxd                      |Explicitly set a backend name, e.g `openhab`, `mqtt` or `knxd`, not needed if you use the default backend|
|BACKEND_KNXD           |                           |URL to access the `knxd` backend|
|BACKEND_MQTT           |                           |URL to access the `mqtt` backend|
|BACKEND_OPENHAB        |                           |Path to the openHAB REST-API|
|CGI_URL_PATH           | /cgi-bin/                 |**Deprecated:** Special case for `knxd` and openHAB: URL prefix to the `cgi-bin` resources|
|KNX_TRANSPORT          | knxd                      |Transport to the KNX bus: `knxd` (bundled or external knxd), `ip-tunnel` (KNXnet/IP tunneling client) or `ip-routing` (KNXnet/IP multicast routing). The `ip-*` transports need no knxd and never start the bundled one. Empty means `knxd`; an unknown value stops the container|
|KNX_IP_HOST            | *(empty)*                 |`ip-tunnel`: address of the KNX IP interface or KNX IP router - **required** for `ip-tunnel`|
|KNX_IP_PORT            | 3671                      |`ip-tunnel` and `ip-routing`: KNXnet/IP UDP port|
|KNX_IP_LOCAL_ADDR      | *(OS choice)*             |`ip-tunnel`: local address that is advertised to the device, useful behind NAT or on a multi-homed host|
|KNX_IP_NAT             | 0                         |`ip-tunnel`: `1` enables the NAT mode of the tunnel connection|
|KNX_IP_HEARTBEAT_SEC   | 60                        |`ip-tunnel`: interval of the connection state request in seconds|
|KNX_IP_ACK_TIMEOUT_MS  | 1000                      |`ip-tunnel`: timeout in ms before a telegram is repeated|
|KNX_IP_RETRIES         | 3                         |`ip-tunnel`: number of repetitions per telegram|
|KNX_IP_MULTICAST       | 224.0.23.12               |`ip-routing`: multicast group of the KNXnet/IP routing|
|KNX_IP_IFACE           | *(default route)*         |`ip-routing`: network interface for the multicast group|
|KNX_IP_INDIVIDUAL_ADDR | 15.15.250                 |`ip-routing`: individual address (`A.B.C`, A and B in `0..15`, C in `0..255`, not `0.0.0`) used as the source of the routing telegrams - must be a free address of the installation|
|KNX_INTERFACE          | *(empty)*                 |**Only for `KNX_TRANSPORT=knxd`:** information for the knxd to connect to the KNX bus, e.g. `iptn:172.17.0.1:3700`|
|KNX_INTERFACE_OPTIONS  | -B single -A retry-delay=1|**Only for `KNX_TRANSPORT=knxd`:** additional parameters for the KNX interface of the knxd|
|KNX_PA                 | 1.1.238                   |**Only for `KNX_TRANSPORT=knxd`:** PA for the knxd|
|KNX_CLIENT_PAS         | 1.1.239:10                |**Only for `KNX_TRANSPORT=knxd`:** PA and number of client connections (`<address>:<count>`) the knxd accepts - must be at least 2|
|KNXD_PARAMETERS        |                           |**Only for `KNX_TRANSPORT=knxd`:** complete parameter set for the knxd, replaces all other knxd parameters|
|NO_ENTRYPOINT_KNXD     | false                     |**Only for `KNX_TRANSPORT=knxd`:** `false`: start knxd inside this container. `extern`: do **not** start knxd, use the socket of an external knxd instead (see `KNXD_SOCKET`). `true`: start neither knxd nor the FastCGI backend. Ignored by the `ip-*` transports, which always start the FastCGI backend|
|KNXD_SOCKET            | /run/knx                  |**Only for `KNX_TRANSPORT=knxd`:** path to the knxd Unix socket. Only used when `NO_ENTRYPOINT_KNXD=extern`|
|BACKEND_PROXY_SOURCE   |                           |Special case for openHAB: proxy paths starting with this value, e.g. `/rest`|
|BACKEND_PROXY_TARGET   |                           |Special case for openHAB: target URL for proxying the requests to BACKEND_PROXY_SOURCE, e.g. `http://<openhab-server-ip-address>:8080/rest`|
|STOP_ON_BAD_HEALTH     | true                      |Stop container on failed health check when set to `true`. This will trigger a new start of the container when docker is configured to do so|
|ACCESS_LOG             | false                     |Show web server access log when set to `true`|
|APACHE_PORT            | 80                        |Expert option: port Apache listens on inside the container. When changed, publish with `-p <host-port>:<APACHE_PORT>` (`EXPOSE` stays `80`)|
|LONGPOLL_TIMEOUT_SEC   | 300                       |Max seconds to wait in long-poll for the `r` (read) backend request|
|FCGI_WORKERS           | 9                         |Number of FastCGI worker processes (at least 2, at most 256). Since cometvisu-knx-fcgi 2.x the workers use no knxd connection of their own, so this is independent of `KNX_CLIENT_PAS`. Two workers are reserved for `/w` and `/l`, therefore `FCGI_WORKERS - 2` clients can long-poll at the same time.|
|KNX_TX_MAX_PER_SEC     | 10                        |Upper bound for outgoing KNX telegrams per second over all clients (protects the bus)|
|DEBUG_BACKEND          | *(unset)*                 |Set to `1` to enable debug logging of the FastCGI backend to stderr|

**Note:** The `KNX_IP_*` variables are empty in the image, so the effective
value is the default of the FastCGI backend (as listed in the table). The
container only passes on the ones that are actually set, so an empty value
never overrides a backend default.

**Note:** `NO_ENTRYPOINT_KNXD`, `KNXD_SOCKET`, `KNX_INTERFACE`,
`KNX_INTERFACE_OPTIONS`, `KNX_PA`, `KNX_CLIENT_PAS` and `KNXD_PARAMETERS` only
configure the `knxd` transport. They are ignored (and not needed) for the
`ip-tunnel` and `ip-routing` transports.

### KNX

By default (`KNX_TRANSPORT=knxd`), the environment variables are configured to
set up a KNX connection via knxd. But make sure that `KNX_INTERFACE` and
`KNX_PA` are set correctly.

The environment variable `BACKEND_KNXD` must be the URL to the `l` login script
of the knxd backend.

Example:
```
BACKEND_NAME=knxd
BACKEND_KNXD=https://server.local/cgi-bin/l
KNX_INTERFACE=iptn:172.17.0.1:3700
KNX_PA=1.1.250
```

**Note:** When `BACKEND_KNXD` and `CGI_URL_PATH` are set at the same time
the value of `BACKEND_KNXD` is used.

**Note:** Since cometvisu-knx-fcgi 2.x the backend uses exactly **one** client
connection of knxd (the cache reader); the FastCGI workers only hand telegrams
to the shared queue. `FCGI_WORKERS` is therefore independent of
`KNX_CLIENT_PAS`; the client count only has to leave room for that one
connection plus a spare for tools like `knxtool`, so at least 2 client
connections are required - otherwise the container refuses to start.

#### External knxd

When knxd already runs outside of this container (e.g. on the host or in
another container) it must not be started twice. Set
`NO_ENTRYPOINT_KNXD=extern` to skip starting knxd while still starting the
FastCGI backend which then connects to the socket provided from outside:

```
docker run -d -p 8080:80 \
  -v /run:/host-run:ro \
  -e NO_ENTRYPOINT_KNXD=extern \
  -e KNXD_SOCKET=/host-run/knx \
  -e BACKEND_NAME=knxd \
  -e BACKEND_KNXD=https://server.local/cgi-bin/l \
  -e FCGI_WORKERS=7 \
  cometvisu/cometvisuabstractbase:latest
```

Notes for this mode:

* `KNXD_SOCKET` defaults to `/run/knx`, so when the socket is mounted exactly
there the variable doesn't need to be set.
* Mount the **directory** containing the socket rather than the socket file
itself. A bind mount of the socket file keeps pointing to the old inode when
the external knxd is restarted and recreates the socket.
* `KNX_PA`, `KNX_CLIENT_PAS`, `KNX_INTERFACE`, `KNX_INTERFACE_OPTIONS` and
`KNXD_PARAMETERS` are ignored - the external knxd brings its own configuration
(including the group cache, which must be enabled there for reads).
* `FCGI_WORKERS` only controls the number of FastCGI worker processes (at
least 2, at most 256); it does not depend on how many client connections the
external knxd accepts, because the backend uses exactly one of them.
* No health check for knxd is generated (neither the process nor the socket is
controlled by this container); only the FastCGI backend and Apache are checked.

**Note:** With `NO_ENTRYPOINT_KNXD=true` neither knxd nor the FastCGI backend is
started, i.e. the `knxd` backend is unavailable and `/cgi-bin/l` will fail.

### Direct KNXnet/IP (no knxd)

With `KNX_TRANSPORT=ip-tunnel` or `KNX_TRANSPORT=ip-routing` the FastCGI backend
reaches the KNX bus itself: the bundled knxd is **not** started (no matter what
`NO_ENTRYPOINT_KNXD` says) and no knxd health check is generated. Only the
FastCGI backend and Apache are checked. `BACKEND_NAME` stays `knxd` and
`BACKEND_KNXD` stays the URL of the `l` login resource - they name the CometVisu
backend (the protocol endpoint), not the transport to the bus.

`ip-tunnel` makes the CometVisu a tunneling client of a KNX IP interface or
KNX IP router, i.e. it behaves like a classical KNX device:

```
docker run -d -p 8080:80 -e KNX_TRANSPORT=ip-tunnel \
  -e KNX_IP_HOST=192.168.0.10 -e KNX_IP_NAT=1 cometvisu/cometvisu:latest
```

`ip-routing` joins the KNXnet/IP multicast group and takes part in the routing
like a KNX IP router:

```
docker run -d --network host -e KNX_TRANSPORT=ip-routing \
  -e KNX_IP_IFACE=eth0 -e KNX_IP_INDIVIDUAL_ADDR=15.15.250 \
  cometvisu/cometvisu:latest
```

Notes for both modes:

* No knxd is required - not even an external one. `KNXD_SOCKET` is not used and
  the knxd settings (`KNX_INTERFACE`, `KNX_PA`, `KNX_CLIENT_PAS`,
  `KNX_INTERFACE_OPTIONS`, `KNXD_PARAMETERS`) are ignored.
* A KNX IP interface offers only a few tunnel connections (often 4) and one of
  them is used by the CometVisu. When all are taken the container cannot
  connect to the bus.
* Multicast does not cross the Docker bridge network: `ip-routing` requires
  `--network host`. Because `-p` cannot be used with `--network host`, set
  `APACHE_PORT` when port 80 is already occupied on the host.
* With the default bridge network the container is behind NAT, so `ip-tunnel`
  needs either `KNX_IP_NAT=1` (the device has to support NAT) or `--network
  host`.
* Every participant of the multicast group can read the bus traffic and send
  telegrams - use `ip-routing` only in a trusted network segment.

### openHAB

The environment variable `BACKEND_OPENHAB` must be the path to the openHAB REST-API.
It might contain a username and password in the normal URL notation.

Example configuration for an openHAB backend (running on host `192.168.0.10`):

```
BACKEND_NAME=openhab
BACKEND_OPENHAB=/rest/
BACKEND_PROXY_SOURCE=/rest/
BACKEND_PROXY_TARGET=http://192.168.0.10:8080/rest/
```

**Note:** The username and password are stored in plain text on the Docker server
as well as in the JavaScript engine of the browser. And when no transport
encryption is used it is even transported in plaintext over the network.

**Note:** When `BACKEND_OPENHAB` and `CGI_URL_PATH` are set at the same time
the value of `BACKEND_OPENHAB` is used.

### MQTT

The environment variable `BACKEND_MQTT` must be the URL to the web socket
interface of the MQTT broker.
It might contain a username and password in the normal URL notation.

Example configuration for an MQTT backend (running on `timberwolf.local` at
port `443` with secure websockets):

```
BACKEND_NAME=mqtt
BACKEND_MQTT=wss://CometVisuUser:Password4MQTTBrokerThatIsNotVerySecure@timberwolf.local:443/proxy/mqtt/ws
```

**Note:** The username and password are stored in plain text on the Docker server
as well as in the JavaScript engine of the browser. And when no transport
encryption is used it is even transported in plaintext over the network.

Setup:
------

The CometVisu should be installed to the directory `/var/www/html`. This would
then result in the config files to be located at `/var/www/html/config` which
should most likely be a volume then.

The RRD files, when that feature is desired to be used, must be located in the
directory `/var/www/rrd/`. So this would also be a volume as the RRD files must
be created and filled up from an external source to this container.  
**NOTE:** the RRD files must be compatible in architecture as they can't be used
otherwise.

### Changing the Apache port (expert option)

Normally it is enough to publish the internal port `80` to any port on the
host, e.g. `-p 8080:80`. When the internal port itself has to change (e.g. with
`--network host` or when containers share a network namespace), set
`APACHE_PORT`:

```
docker run -d -p 8080:8081 -e APACHE_PORT=8081 cometvisu/cometvisuabstractbase:latest
```

Not available together with `NO_ENTRYPOINT_APACHE=true`, as in that case the
Apache configuration is entirely up to you.

FAQ:
----

* **Question:** Why does the log show a message like `apache2: Could not
  reliably determine the server's fully qualified domain name, using 172.17.0.2.
  Set the 'ServerName' directive globally to suppress this message`?  
  **Answer:** The Docker container was started without setting the parameter 
  `--hostname` (or, in short, `-h`).  
  Using Portainer this would be done at "Network" on the field "Domain Name".
