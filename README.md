# iRODS C++ REST Mid-Tier API

This REST API is designed to be deployed in front of an iRODS Server to provide an HTTP REST interface into the iRODS protocol.

This REST API requires an iRODS Server v4.2.0 or greater.

## Quickstart

The iRODS C++ REST API can be installed via package manager and managed via `systemctl` alongside an existing iRODS Server:

```
# REST API - install
$ sudo apt install irods-client-rest-cpp
OR
$ sudo yum install irods-client-rest-cpp
```

nginx, rsyslog, and logrotate can be configured and restarted before starting the REST API:

```
# nginx - configure and restart
$ sudo cp /etc/irods_client_rest_cpp/irods_client_rest_cpp_reverse_proxy.conf.template /etc/nginx/sites-available/irods_client_rest_cpp_reverse_proxy.conf
$ sudo ln -s /etc/nginx/sites-available/irods_client_rest_cpp_reverse_proxy.conf /etc/nginx/sites-enabled/irods_client_rest_cpp_reverse_proxy.conf
$ sudo rm /etc/nginx/sites-enabled/default
$ sudo systemctl restart nginx

# rsyslog - configure and restart
$ sudo cp /etc/irods_client_rest_cpp/irods_client_rest_cpp.conf.rsyslog /etc/rsyslog.d/00-irods_client_rest_cpp.conf
$ sudo cp /etc/irods_client_rest_cpp/irods_client_rest_cpp.logrotate /etc/logrotate.d/irods_client_rest_cpp
$ sudo systemctl restart rsyslog

# REST API - configure and start
$ sudo cp /etc/irods_client_rest_cpp/irods_client_rest_cpp.json.template /etc/irods_client_rest_cpp/irods_client_rest_cpp.json
$ sudo systemctl restart irods_client_rest_cpp
```

## Building this repository
This is a standard CMake project which may be built with either Ninja or Make.

This project depends on `irods-dev(el)` and `irods-runtime` version 4.3.0 (available from https://packages.irods.org). The minimum required CMake is 3.12.0. You can install the latest `irods-externals-cmake` package if needed (also available from https://packages.irods.org).

Once these are installed, the other `irods-externals-*` versions will be revealed through building with `cmake` and may change over time as this code base develops.

Prerequisites (available from https://packages.irods.org):
```
irods-dev(el)
irods-runtime
irods-externals-boost
irods-externals-clang
irods-externals-json
irods-externals-pistache
irods-externals-spdlog
```

Create and enter a build directory, run CMake (make sure it is visible in your `$PATH`), and then make the REST API package:
```
$ mkdir build && cd build && cmake ..
$ make -j package
```

Note: For machines with fewer than 4 cores or heavy workloads, building using the `-j` option with `make` should be done with caution. Refer to the manual for more information on this topic: https://www.gnu.org/software/make/manual/make.html#Parallel-Execution

## Building with Docker

This repository provides a Dockerfile with all of the tools needed to build a package for Ubuntu 20.04.

The builder image itself must first be built. This can be done by running the following:
```bash
cd docker/builder
docker build -t irods-client-rest-cpp-builder:ubuntu-20.04 .
```

This will produce a Docker Image with tag `irods-client-rest-cpp-builder:ubuntu-20.04`. The tag can be anything, as long as it is used consistently.

Once built, the builder can be run to build packages from source code on the host machine. The script run by the container takes the following inputs:

1. A volume mount for the source directory (this repository) on the host
2. A volume mount for the output build files on the host
3. A volume mount for the package by itself on the host

Here is an example usage:
```bash
docker run -t --rm \
           -v ${sourcedir}:/src:ro \
           -v ${builddir}:/build \
           -v ${packagedir}:/packages \
           irods-client-rest-cpp-builder:ubuntu-20.04
```
`$sourcedir`, `$builddir`, and `$packagedir` should be full paths to the source code directory, the desired output directory for build artifacts, and the desired output directory for the built package.

Once the build has completed, you should find a `.deb` file on the host machine where you specified `$packagedir` to be.

Integrating this into your workflow is an exercise left to the reader.

## Configuration files
The REST API provides an executable for each individual API endpoint. These endpoints may be grouped behind a reverse proxy in order to provide a single port for access.

The service relies on a configuration file in `/etc/irods_client_rest_cpp` which dictates which ports are used. Two template files are placed there by the package:
```bash
/etc/irods_client_rest_cpp/irods_client_rest_cpp.json.template
/etc/irods_client_rest_cpp/irods_client_rest_cpp_reverse_proxy.conf.template
```

## Starting the service
To start the REST API service, run the following commands:
```bash
$ sudo cp /etc/irods_client_rest_cpp/irods_client_rest_cpp.json.template /etc/irods_client_rest_cpp/irods_client_rest_cpp.json # configuration
$ sudo systemctl start irods_client_rest_cpp # start the service
```

If everything was successful, you now have a functional REST API service.

If you modify the configuration file (i.e. port numbers, log level, etc.). You'll need to restart the service for the changes to take affect.

## Starting the reverse proxy using Nginx
This section assumes you have a functional REST API service.

The first thing you must do is install `nginx`. Once installed, run the following commands to enable the reverse proxy:
```bash
$ sudo cp /etc/irods_client_rest_cpp/irods_client_rest_cpp_reverse_proxy.conf.template /etc/nginx/sites-available/irods_client_rest_cpp_reverse_proxy.conf # configuration
$ sudo ln -s /etc/nginx/sites-available/irods_client_rest_cpp_reverse_proxy.conf /etc/nginx/sites-enabled/irods_client_rest_cpp_reverse_proxy.conf # configuration
$ sudo systemctl restart nginx # start the service
```

If you modified any port numbers in the REST API's configuration file, you will need to adjust the reverse proxy's configuration file so that `nginx` can connect to the correct endpoint.

If you are getting a 404 error from the reverse proxy, make sure to remove the `default` file from `/etc/nginx/sites-enabled` directory. This default site may be listening on the same port as your reverse proxy and preventing the requests from reaching it.

## Enabling logging via Rsyslog and Logrotate
_This section assumes you've installed the C++ REST API package._

_If you are doing a non-package install, you'll need to copy the configuration files from the repository to the appropriate directories and restart rsyslog. The files of interest and their destinations are shown below._

The REST API uses rsyslog and logrotate for log file management. To enable, run the following commands:
```bash
$ sudo cp /etc/irods_client_rest_cpp/irods_client_rest_cpp.conf.rsyslog /etc/rsyslog.d/00-irods_client_rest_cpp.conf
$ sudo cp /etc/irods_client_rest_cpp/irods_client_rest_cpp.logrotate /etc/logrotate.d/irods_client_rest_cpp
$ sudo systemctl restart rsyslog
```

The log file will be located at `/var/log/irods_client_rest_cpp/irods_client_rest_cpp.log`.

The log level for each endpoint can be adjusted by modifying the `"log_level"` option in `/etc/irods_client_rest_cpp/irods_client_rest_cpp.json`. The following values are supported:
- trace
- debug
- info
- warn
- error
- critical

## HTTP and SSL
It is highly advised to run the service with HTTPS enabled. The administrative API endpoint (i.e. _/admin_) is implemented to accept passwords in **plaintext**. This is on purpose as it removes the password obfuscation requirements from applications built upon the C++ REST API.

Please refer to your proxy server's (nginx, apache httpd, etc.) documentation for enabling SSL communication.

## Starting the services with Docker Compose

This repository provides a Docker Compose project which runs a local iRODS C++ REST client package in an Ubuntu 20.04-based container alongside an nginx reverse-proxy service.

To run, the following things will be required:

1. A `.deb` package built for Ubuntu 20.04 residing on the filesystem of the local host (see [Building with Docker](#building-with-docker)).
2. An iRODS server reachable by the containers providing the Compose project services on the host machine.

The Compose project is found under `docker/runner` along with the attendant files.

Copy the `.deb` package into `docker/runner` (i.e. `cp /path/to/package.deb ./docker/run`). The package will be installed as part of the Docker image built for the REST client service. If you are having trouble, check to make sure that the name of the file you copied matches the `local_package` build argument under the `irods-client-rest-cpp` service definition in the `./docker/runner/docker-compose.yml` file.

You will also need to modify `./docker/runner/irods_client_rest_cpp.json` to match the desired iRODS client environment. You may also wish to change the `jwt_signing_key` outside of an experimental context.

Now that things are set up, let's run it:
```
cd ./docker/runner
docker-compose up
```

The nginx and REST client service Docker images should be built and then containers spawned for each service. The nginx service exposes port 80 on the host machine by default. If you wish to change this, edit the `./docker/runner/docker-compose.yml` file and restart the service.  Note: The ports for the REST services are not exposed on the host machine - only the nginx reverse-proxy port is exposed.

To stop the service:
```
docker-compose down
```

To modify configuration settings for the REST services, edit the `./docker/runner/irods_client_rest_cpp.json` file on the host machine and restart the services:
```
docker-compose restart
```

## Interacting with the API endpoints
The design of this API uses JWTs to contain authorization and identity. The Auth endpoint must be invoked first in order to authenticate and receive a JWT. This token will then need to be included in the Authorization header of each subsequent request. This API follows a [HATEOAS](https://en.wikipedia.org/wiki/HATEOAS#:~:text=Hypermedia%20as%20the%20Engine%20of,provide%20information%20dynamically%20through%20hypermedia.) design which provides not only the requested information but possible next operations on that information.

## Error messages
Unless otherwise specified, successful operations by this client should return with an empty body. If some error is reported, it will follow this format:
```json
{
	"error_code": integer,
	"error_message": string
}
```

### /admin
The administration interface to the iRODS Catalog which allows the creation, removal and modification of users, groups, resources, and other entities within the zone. **ALL input parameters must be defined. If an input parameter is not to be used, set it to nothing (e.g. arg7=)**

**Method**: POST

**Parameters**
- action: dictates the action taken: add, modify, or remove
- target: the subject of the action: user, zone, resource, childtoresc, childfromresc, token, group, rebalance, unusedAVUs, specificQuery
- arg2: generic argument, could be user name, resource name, depending on the value of `action` and `target`
- arg3: generic argument, see above
- arg4: generic argument, see above
- arg5: generic argument, see above
- arg6: generic argument, see above
- arg7: generic argument, see above

**Example CURL Command:**
```
curl -X POST -H "Authorization: ${TOKEN}" 'http://localhost/irods-rest/0.9.3/admin?action=add&target=resource&arg2=ufs0&arg3=unixfilesystem&arg4=/tmp/irods/ufs0&arg5=&arg6=tempZone&arg7='
```

**Returns**

"Success" or an iRODS exception

### /auth
This endpoint provides an authentication service for the iRODS zone, currently only native iRODS authentication is supported.

**Method**: POST

**Parameters:**
- Authorization Header of the form `Authorization: Basic ${SECRETS}` where ${SECRETS} is a base 64 encoded string of user_name:password

**Example CURL Command:**
```
export SECRETS=$(echo -n rods:rods | base64)
export TOKEN=$(curl -X POST -H "Authorization: Basic ${SECRETS}" http://localhost:80/irods-rest/0.9.3/auth)
```

**Returns:**

An encrypted JWT which contains everything necessary to interact with the other endpoints. This token is expected in the Authorization header for the other services.

### /get_configuration
This endpoint will return a JSON structure holding the configuration for an iRODS server

**Method**: GET

**Parameters**
- None

**Example CURL Command:**
```
curl -X GET -H "Authorization: ${TOKEN}" 'http://localhost/irods-rest/0.9.3/get_configuration' | jq
```

### /list
This endpoint provides a recursive listing of a collection, or stat, metadata, and access control information for a given data object.

**Method**: GET

**Parameters**
- logical-path: The url encoded logical path which is to be listed
- stat: Boolean flag to indicate stat information is desired
- permissions: Boolean flag to indicate access control information is desired
- metadata: Boolean flag to indicate metadata is desired
- offset: number of records to skip for pagination
- limit: number of records desired per page

**Example CURL Command:**
```
curl -X GET -H "Authorization: ${TOKEN}" 'http://localhost/irods-rest/0.9.3/list?logical-path=%2FtempZone%2Fhome%2Frods&stat=0&permissions=0&metadata=0&offset=0&limit=100' | jq
```

**Returns**

A JSON structured response within the body containing the listing, or an iRODS exception
```
{
  "_embedded": [
    {
      "logical_path": "/tempZone/home/rods/subcoll",
      "type": "collection"
    },
    {
      "logical_path": "/tempZone/home/rods/subcoll/file0",
      "type": "data_object"
    },
    {
      "logical_path": "/tempZone/home/rods/subcoll/file1",
      "type": "data_object"
    },
    {
      "logical_path": "/tempZone/home/rods/subcoll/file2",
      "type": "data_object"
    },
    {
      "logical_path": "/tempZone/home/rods/file0",
      "type": "data_object"
    }
  ],
  "_links": {
    "first": "/irods-rest/0.9.3/list?logical-path=%2FtempZone%2Fhome%2Frods&stat=0&permissions=0&metadata=0&offset=0&limit=100",
    "last": "/irods-rest/0.9.3/list?logical-path=%2FtempZone%2Fhome%2Frods&stat=0&permissions=0&metadata=0&offset=UNSUPPORTED&limit=100",
    "next": "/irods-rest/0.9.3/list?logical-path=%2FtempZone%2Fhome%2Frods&stat=0&permissions=0&metadata=0&offset=100&limit=100",
    "prev": "/irods-rest/0.9.3/list?logical-path=%2FtempZone%2Fhome%2Frods&stat=0&permissions=0&metadata=0&offset=0&limit=100",
    "self": "/irods-rest/0.9.3/list?logical-path=%2FtempZone%2Fhome%2Frods&stat=0&permissions=0&metadata=0&offset=0&limit=100"
  }
}
```

### /logicalpath
Interactions for paths within the iRODS logical namespace.

**Method** POST

Create an entity in the iRODS logical namespace.

**Parameters**
- logical-path: The absolute path leading to the data object or collection to be created.
- collection: Indicates that a collection is being created. Defaults to "0".
- create-parent-collections: Indicates that parent collections of the destination collection should be created if necessary. Only applicable when `&collection=1` is used. Defaults to "0".

**Example CURL command**
```
# Equivalent to: imkdir -p /tempZone/home/rods/a/b/c
curl -X POST -H "Authorization: ${TOKEN}" "http://localhost/irods-rest/0.9.3/logicalpath?logical-path=/tempZone/home/rods/a/b/c&collection=1&create-parent-collections=1"
```

**Returns**
Nothing on success.

**Method** DELETE

Deletes a data object or a collection.

**Parameters**
- logical-path: The absolute path leading to the data object or collection to be deleted.
- no-trash: Don't send to trash, delete permanently. Optional, defaults to false.
- recursive: Recursively delete contents of a collection. Optional, defaults to false.
- unregister: Unregister data objects instead of deleting them. Optional, defaults to false.

**Example CURL command**
```
curl -X DELETE -H "Authorization: ${TOKEN}" "http://localhost:80/irods-rest/0.9.3/logicalpath?logical-path=/tempZone/home/rods/hello.cpp&no-trash=1"
```

**Returns**
Nothing on success.

### /logicalpath/rename
Renames a data object or collection

**Method**: POST

**Parameters**
- src: The path to the data object or collection to be renamed.
- dst: New name of the target data object or collection.

**Example CURL command**
```
curl -X POST -H "Authorization: ${TOKEN}" "http://localhost:80/irods-rest/0.9.3/logicalpath/rename?src=/tempZone/home/rods/hello&dst=/tempZone/home/rods/goodbye"
```

**Returns**
Nothing on success.

### /logicalpath/replicate
Replicates a data object into some resource

**Method**: POST

**Parameters**:
  - all: If selected, updates all stale copies.
  - recursive:  Required to replicate a collection. Replicates the whole subtree. Does nothing if used on a data object.
  - thread-count: The number of threads to use for replication.
  - replica-number: Specifies the particular replicate to copy from. Typically not needed.
  - dst-resource: Resource to which to replicate the data object or collection.
  - src-resource: Resource from which to replicate the data object or collection.
  - logical-path: The logical path of the data object or collection to replicate.
  - admin-mode: Required for an admin to replicate other users' data objects.

**Example CURL command**:
```
curl -X POST -H "Authorization: ${TOKEN}" 'http://localhost/irods-rest/0.9.3/logicalpath/replicate?logical-path=/tempZone/home/rods/hello.cpp&dst-resource=ufs0'
```

### /logicalpath/trim

**Method**: POST
**Parameters**:
  - recursive: Required to trim a collection. Trims the whole subtree. Does nothing if used on a data object.
  - minimum-age-in-minutes: The minimum age in minutes foe a data object to be a candidate for trimmed. If a data object is younger, nothing will happen to it.
  - minimum-number-of-remaining-replicas: The minimum number of copies to leave after trimming. Defaults to 2.
  - replica-number: Determines the replica to trim.
  - src-resource: If specified, only replicas on this resource will be candidates for trimming.
  - admin-mode: Required for an admin to trim replicas of other users' data objects.

**Example CURL command**:
```
curl -X POST -H "Authorization: ${TOKEN}" 'http://localhost/irods-rest/0.9.3/logicalpath/trim?logical-path=/tempZone/home/rods/foo&src-resource=ufs0'
```

**Returns**
Nothing on success.

### /metadata
This endpoint allows executing multiple metadata operations on a single object atomically.

**Method**: POST

**Parameters**
The commands provided to the API are passed as JSON in the data payload.
See [here](https://docs.irods.org/4.3.0/doxygen/atomic__apply__metadata__operations_8h.html) for the format of these commands.

**Example CURL Command:**
```
curl -X POST \
-H "Authorization: ${TOKEN}" \
-H "Content-Type: application/json" \
--data '{
  "entity_name": "/tempZone/home/rods/dir",
  "entity_type": "collection",
  "operations": [
    {
      "operation": "add",
      "attribute": "attribute",
      "value": "value",
      "units": "unit"
    }
  ]
}' \
http://localhost/irods-rest/0.9.3/metadata
```

**Returns**
Nothing on success

### /put_configuration
This endpoint will write the url encoded JSON to the specified files in `/etc/irods`

**Method**: PUT

**Parameters**
- cfg: a url encoded json string of the format
```JSON
[
    {
        "file_name":"test_rest_cfg_put_1.json",
        "contents": {
            "key0":"value0",
            "key1": "value1"
        }
    },
    {
        "file_name":"test_rest_cfg_put_2.json",
        "contents": {
            "key2": "value2",
            "key3": "value3"
        }
    }
]
```

**Example CURL Command:**
```
export CONTENTS="%5B%7B%22file_name%22%3A%22test_rest_cfg_put_1.json%22%2C%20%22contents%22%3A%7B%22key0%22%3A%22value0%22%2C%22key1%22%20%3A%20%22value1%22%7D%7D%2C%7B%22file_name%22%3A%22test_rest_cfg_put_2.json%22%2C%22contents%22%3A%7B%22key2%22%20%3A%20%22value2%22%2C%22key3%22%20%3A%20%22value3%22%7D%7D%5D"
curl -X PUT -H "Authorization: ${TOKEN}" "http://localhost/irods-rest/0.9.3/put_configuration?cfg=${CONTENTS}"
```

**Returns**
Nothing on success

### /query
This endpoint provides access to the iRODS General Query language, which is a generic query service for the iRODS catalog.

**Method**: GET

**Parameters**
- query: A url encoded GenQuery string
- limit: The max number of rows to return
- offset: Number of rows to skip for paging
- type: Either 'general' or 'specific'
- case-sensitive: Affects string matching in GenQuery. Defaults to 1
- distinct: Requests distinct rows from GenQuery. Defaults to 1

**Example CURL Command:**
```
curl -X GET -H "Authorization: ${TOKEN}" 'http://localhost/irods-rest/0.9.3/query?limit=100&offset=0&type=general&query=SELECT%20COLL_NAME%2C%20DATA_NAME%20WHERE%20COLL_NAME%20LIKE%20%27%2FtempZone%2Fhome%2Frods%25%27' | jq
```

**Returns**
A JSON structure containing the query results
```
{
  "_embedded": [
    [
      "/tempZone/home/rods",
      "file0"
    ],
    [
      "/tempZone/home/rods/subcoll",
      "file0"
    ],
    [
      "/tempZone/home/rods/subcoll",
      "file1"
    ],
    [
      "/tempZone/home/rods/subcoll",
      "file2"
    ]
  ],
  "_links": {
    "first": "/irods-rest/0.9.3/query?query=SELECT%20COLL_NAME%2C%20DATA_NAME%20WHERE%20COLL_NAME%20LIKE%20%27%2FtempZone%2Fhome%2Frods%25%27&limit=100&offset=0&type=general&case-sensitive=1&distinct=1",
    "last": "/irods-rest/0.9.3/query?query=SELECT%20COLL_NAME%2C%20DATA_NAME%20WHERE%20COLL_NAME%20LIKE%20%27%2FtempZone%2Fhome%2Frods%25%27&limit=100&offset=0&type=general&case-sensitive=1&distinct=1",
    "next": "/irods-rest/0.9.3/query?query=SELECT%20COLL_NAME%2C%20DATA_NAME%20WHERE%20COLL_NAME%20LIKE%20%27%2FtempZone%2Fhome%2Frods%25%27&limit=100&offset=0&type=general&case-sensitive=1&distinct=1",
    "prev": "/irods-rest/0.9.3/query?query=SELECT%20COLL_NAME%2C%20DATA_NAME%20WHERE%20COLL_NAME%20LIKE%20%27%2FtempZone%2Fhome%2Frods%25%27&limit=100&offset=0&type=general&case-sensitive=1&distinct=1",
    "self": "/irods-rest/0.9.3/query?query=SELECT%20COLL_NAME%2C%20DATA_NAME%20WHERE%20COLL_NAME%20LIKE%20%27%2FtempZone%2Fhome%2Frods%25%27&limit=100&offset=0&type=general&case-sensitive=1&distinct=1"
  },
  "count": "4",
  "total": "4"
}
```


### /stream
Stream data into and out of an iRODS data object

**Method**: GET and PUT

**Parameters**
- logical-path: The url encoded logical path to a data object
- offset: The offset in bytes into the data object (Defaults to 0)
- count: The maximum number of bytes to read or write.
  - Required for GET requests.
  - On a GET, this parameter is limited to signed 32-bit integer.
  - On a PUT, this parameter is limited to signed 64-bit integer.
- truncate: Truncates the data object on open
  - Defaults to "true".
  - Applies to PUT requests only.

**Returns**

PUT: Nothing, or iRODS Exception

GET: The data requested in the body of the response

**Example CURL Command:**
```
curl -X PUT -H "Authorization: ${TOKEN}" [-H "irods-ticket: ${TICKET}"] -d"This is some data" 'http://localhost/irods-rest/0.9.3/stream?logical-path=%2FtempZone%2Fhome%2Frods%2FfileX&offset=10'
```
or
```
curl -X GET -H "Authorization: ${TOKEN}" [-H "irods-ticket: ${TICKET}"] 'http://localhost/irods-rest/0.9.3/stream?logical-path=%2FtempZone%2Fhome%2Frods%2FfileX&offset=0&count=1000'
```

### /ticket
This endpoint provides a service for the generation of an iRODS ticket to a given logical path, be that a collection or a data object.

**Method**: GET

**Parameters:**
- logical-path: The url encoded logical path to a collection or data object for which access is desired
- type: The type of ticket to create. The value must be either read or write. Defaults to read
- use-count: The maximum number of times the ticket can be used. Defaults to 0 (unlimited use)
- write-file-count: The maximum number of writes allowed to a data object. Defaults to 0 (unlimited writes)
- write-byte-count: The maximum number of bytes allowed to be written to data object. Defaults to 0 (unlimited bytes)
- seconds-until-expiration: The number of seconds before the ticket will expire. Defaults to 0 (no expiration)
- users: A comma-separated list of iRODS users who are allowed to use the generated ticket
- groups: A comma-separated list of iRODS groups that are allowed to use the generated ticket
- hosts: A comma-separated list of hosts that are allowed to use the ticket

**Example CURL Command:**
```
curl -X GET -H "Authorization: ${TOKEN}" 'http://localhost/irods-rest/0.9.3/ticket?logical-path=%2FtempZone%2Fhome%2Frods%2Ffile0&type=write&write-file-count=10'
```

**Returns**

An iRODS ticket token within the **irods-ticket** header, and a URL for streaming the object.
```
{
  "headers": {
    "irods-ticket": ["CS11B8C4KZX2BIl"]
  },
  "url": "/irods-rest/0.9.3/stream?logical-path=%2FtempZone%2Fhome%2Frods%2Ffile0&offset=0&count=33064"
}
```

### /zonereport
Requests a JSON formatted iRODS Zone report, containing all configuration information for every server in the grid.

**Method**: GET

**Parameters**
- None

**Example CURL Command:**
```
curl -X GET -H "Authorization: ${TOKEN}" 'http://localhost/irods-rest/0.9.3/zonereport' | jq
```

**Returns**
JSON formatted Zone Report

```
{
  "schema_version": "file:///var/lib/irods/configuration_schemas/v3/zone_bundle.json",
  "zones": [
    {
    <snip>
    }]
}
```

## Running test suite

The test suite for this repository is heavily tied to the iRODS server's python test suite and affiliated libraries. As such, an iRODS server is required to be running on the same machine as the REST client in order to run the python tests.

Running the tests has been captured in a Compose project under `./docker/tester`. Like the Compose project in `./docker/runner`, a .deb package built for Ubuntu 20.04 is required in the build context. The package is installed when the Compose project is run.

To run the tests using Docker Compose, copy the .deb file to `./docker/tester` and run the following:
```bash
cd ./docker/tester
docker-compose up
```

The Compose project will set up the iRODS server and REST client, and then run the python test suite. If you are having trouble, check to make sure that the name of the file you copied matches the `local_package` environment variable under the `irods-client-rest-cpp-tester` service definition in the `./docker/tester/docker-compose.yml` file.

When the tests have completed, the `irods-client-rest-cpp-tester` service will end, but the Compose project will not be brought down by itself. This can be done by running the following:
```bash
docker-compose down
```

To run a specific test, add the following line to the `irods-client-rest-cpp` service stanza in the `docker-compose.yml` file:
```yaml
command: ["--run_s test_irods_client_rest_cpp.TestClientRest.test_name_here"]
```
This will override the `CMD` defined in the Dockerfile which passes the specific test to run to the `ENTRYPOINT`.
