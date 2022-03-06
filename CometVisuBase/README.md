Abstract base docker container for the CometVisu
================================================

[![](https://images.microbadger.com/badges/version/cometvisu/cometvisuabstractbase.svg)](https://microbadger.com/images/cometvisu/cometvisuabstractbase "Get your own version badge on microbadger.com") [![](https://images.microbadger.com/badges/image/cometvisu/cometvisuabstractbase.svg)](https://microbadger.com/images/cometvisu/cometvisuabstractbase "Get your own image badge on microbadger.com")

This container can be used as a base for own Docker containers that contain the [CometVisu](https://www.cometvisu.org/). It contains an Apache / PHP combo with the knxd (0.0.5.1). Also RRD support for the diagram plugin is implemented.

This container is available at DockerHub as [cometvisu/cometvisuabstractbase](https://hub.docker.com/r/cometvisu/cometvisuabstractbase/).

**NOTE:** This is just the abstract base for a CometVisu contianer. When you are looking for a ready to use container of the CometVisu you should look at [cometvisu/cometvisu](https://hub.docker.com/r/cometvisu/cometvisu/).

Environment parameters:
-----------------------

|Parameter              |Default                  |Description|
|-----------------------|-------------------------|-----------|
|BACKEND_NAME           |                         |Explicitly set a backend name, e.g `openhab`, `mqtt` or `knxd`, not needed if you use the default backend|
|BACKEND_KNXD           |                         |URL to access the `knxd` backend|
|BACKEND_MQTT           |                         |URL to access the `mqtt` backend|
|BACKEND_OPENHAB        |                         |URL to access the `openhab` backend|
|CGI_URL_PATH           |/cgi-bin/                |**Depreciated:** Special case for `knxd` and openHAB: URL prefix to the `cgi-bin` resources|
|KNX_INTERFACE          |iptn:172.17.0.1:3700     |Special case for `knxd`: information for the knxd to connect to the KNX bus|
|KNX_PA                 |1.1.238                  |Special case for `knxd`: PA for the knxd|
|KNXD_PARAMETERS        |-u -d/var/log/eibd.log -c|Special case for `knxd`: additional startup parameters for the knxd|
|BACKEND_PROXY_SOURCE   |                         |Special case for openHAB: proxy paths starting with this value, e.g. `/rest`|
|BACKEND_PROXY_TARGET   |                         |Special case for openHAB: target URL for proxying the requests to BACKEND_PROXY_SOURCE, e.g. `http://<openhab-server-ip-address>:8080/rest`|
|STOP_ON_BAD_HEALTH     |false                    |Stop container on failed health check when set to `true`. This will trigger a new start of the container when docker is configured to do so|
|ACCESS_LOG             |false                    |Show web server access log when set to `true`|

### KNX

By default the environment variables are configured to setup a KNX connection.
But make sure that `KNX_INTERFACE` and `KNX_PA` are set correctly.

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

### openHAB

The environment variable `BACKEND_OPENHAB` must be the URL to the rest-ressouce.
It might contain a user name and password in the normal URL notation.

Example configuration for an openHAB backend (running on host `192.168.0.10`):

```
BACKEND_NAME=openhab
BACKEND_OPENHAB=/rest/cv/
BACKEND_PROXY_SOURCE=/rest/
BACKEND_PROXY_TARGET=http://192.168.0.10:8080/rest/
```

**Note:** The user name and password are stored in plain text on the Docker server
as well as in the JavaScript engine of the browser. And when no transport
encryption is used it is even transported in plaintext over the network.

**Note:** When `BACKEND_OPENHAB` and `CGI_URL_PATH` are set at the same time
the value of `BACKEND_OPENHAB` is used.

### MQTT

The environment variable `BACKEND_MQTT` must be the URL to the web socket
interface of the MQTT broker.
It might contain a user name and password in the normal URL notation.

Example configuration for an MQTT backend (running on `timberwolf.local` at
port `443` with secure websockets):

```
BACKEND_NAME=mqtt
BACKEND_MQTT=wss://CometVisuUser:Password4MQTTBrokerThatIsNotVerySecure@timberwolf.local:443/proxy/mqtt/ws
```

**Note:** The user name and password are stored in plain text on the Docker server
as well as in the JavaScript engine of the browser. And when no transport
encryption is used it is even transported in plaintext over the network.

Setup:
------

The CometVisu should be installed to the directory `/var/www/html`. This would then result in the config files to be located at `/var/www/html/config` which should most likely be a volume then.

The RRD files, when that feature is desired to be used, must be located in the directory `/var/www/rrd/`. So this would also be a volume as the RRD files must be created and filled up from an external source to this container.  
**NOTE:** the RRD files must be compatible in architecture as they can't be used otherwise.

FAQ:
----

* **Question:** Why does the log show a message like `apache2: Could not reliably determine the server's fully qualified domain name, using 172.17.0.2. Set the 'ServerName' directive globally to suppress this message`?  
  **Answer:** The Docker container was started without setting the parameter `--hostname` (or, in short, `-h`).  
  Using Portainer this would be done at "Network" on the field "Domain Name".