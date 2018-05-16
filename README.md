lustre-docker-kdc
=================

Lustre-specific Docker container for a Heimdal Kerberos 5 KDC.

The purpose is to provide a KDC ready for use with Lustre, suitable for testing but not for production as-is.

---

# Dependencies
- Docker
- jq 1.4

---

# Usage

### Check your configuration
The default configuration is likely to be fine for your first steps, validate it using the `config` command.

```
./kdc config
```

You will receive a list of relevant configuration information. The defaults are derived from your hosts' configuration to allow for a quick test setup.

**Example output: `./kdc config`**

    System
      fqdn:      hostname.domain.name
    KDC
      host:      127.0.0.1
      port:      48088
    Kerberos
      domain:    domain.name
      realm:     DOMAIN.NAME
      principal: lustre_root/hostname.domain.name@DOMAIN.NAME, password:


### Build the docker image
```
./kdc build
```

This will render the image which is based on plain ubuntu 16.04. Additionally the packages `heimdal-kdc` as well as `libsasl2-modules-gssapi-heimdal` are installed. The latter is useful only if you extend this container image by further applications making use of Kerberos authentication via SASL2's GSSAPI.


### Run the container
```
./kdc start
```

This will start the container in detached mode, allowing you to keep on working with this shell without having to fork another process. The container name is 'kdc' by default.

Then you will be instructed how to setup Kerberos and Lustre to use them in conjunction.
It implies copying following files on corresponding nodes:

* krb5.conf on all server and clients nodes
* keytab files generated under the Keytabs directory, on each corresponding node.

Then, as simple as A-B-C, setup Lustre in 3 steps:

* on all Lustre nodes (servers, clients), declare the program to emit authentication requests. This is simply done by creating the file `/etc/request-key.d/lgssc.conf` with the following content:
```create lgssc * * /usr/sbin/lgss_keyring %o %k %t %d %c %u %g %T %P %S```
* on server nodes only (MDS, OSS), start the daemon that is responsible for checking authentication credentials. Directly run in a shell:
```# lsvcgssd -vv -k```
* on MGS node, activate Kerberos authentication for your file system. From a shell, launch:
```# lctl conf_param <fsname>.srpc.flavor.default=krb5n```

### Watch the KDC server log file
```
docker exec -it kdc tail -f /var/log/heimdal-kdc.log
```

### Run a quick test
```
./kdc test
```

A network connection to the KDC is attempted.



### Stop the container
```
./kdc stop
```

This will stop the KDC server, stop and remove the container and additionally remove the temporary keytab and configuration files.
Please make sure to remove potential credential caches `/tmp/krb5cc_lustre*` on all Lustre nodes.


### Cleanup
```
./kdc clean
```

This will remove the Docker image.


### Customize your configuration
You have to use a JSON configuration file to customize the setup. The default filename for the JSON file is `kdc.json` (in the project's root directory) but may be configured by the environment variable `KDC_CONFIG`.

Please see `samples/kdc.json` as an example.

**samples/kdc.json**

    {
      "principals": [
        {
          "id": "lustre_mds/mds_server.example.com@EXAMPLE.COM"
        },
        {
          "id": "lustre_oss/oss_server.example.com@EXAMPLE.COM"
        },
        {
          "id": "lustre_root/client_node.example.com@EXAMPLE.COM"
        },
        {
          "id": "userA@EXAMPLE.COM",
          "password": "dummy"
        }
      ],
      "domain": "example.com",
      "realm": "EXAMPLE.COM",
      "ip": "127.0.0.1",
      "port": 48088
    }

#### Kerberos principal
| config node |
|-------------|
| id          |

#### Kerberos password
| config node |
|-------------|
| password    |

**Note**: `lustre_mds`, `lustre_oss` and `lustre_root` principals must be passwordless for Lustre to work.

#### Kerberos domain name
| config node | default                                  |
|-------------|------------------------------------------|
| domain      | hostname cut off output of `hostname -f` |

#### Kerberos realm name
| config node | default                          |
|-------------|----------------------------------|
| realm       | capitalized value of domain name |

**Note**: it is common practice to simply use the domain-name but all capitalized for this.

#### External KDC IP
| config node |
|-------------|
| ip          |

#### External KDC port
| config node |
|-------------|
| port        |

#### KDC hostname == container name
| env. variable | default   |
|---------------|-----------|
| KDC\_HOST\_NAME | `kdc`     |


---

# Reference

```
./kdc start|stop|build|clean|config
```

## config
Shows configuration information as contained in `kdc.json`.

## build
Builds the docker image.

## start
Starts the container in detached mode while also producing a Kerberos configuration file (`krb5.conf`) as well as Kerberos keytabs (`krb5.keytab*`).

## test
Checks if the KDC is reachable and accepting connections.

## stop
Stops the container and deletes `krb5.conf` as well as `krb5.keytab*`.

## clean
Removes the docker image.


---

# Original author

* [Till Toenshoff](https://github.com/tillt) ([@ttoenshoff](https://twitter.com/ttoenshoff))
