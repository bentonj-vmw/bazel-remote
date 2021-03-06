![Build status](https://badge.buildkite.com/c11240e6e9519111f2380dfcf5fcb49e69fd5b2326c11a3059.svg?branch=master)

# A remote build cache for [Bazel](https://bazel.build)

bazel-remote is a HTTP/1.1 and gRPC server that is intended to be used as a remote build cache for
[Bazel](https://bazel.build). The cache contents are stored in a directory on disk. One can specify a maximum cache
size and bazel-remote will automatically enforce this limit and clean the cache by deleting files based on their
last access time. The cache supports HTTP basic authentication with usernames and passwords being specified by a
`.htpasswd` file.

**Project status**: bazel-remote has been serving TBs of cache artifacts per day since April 2018, both on
commodity hardware and AWS servers. Outgoing bandwidth can exceed 15 Gbit/s on the right AWS instance type.

## HTTP/1.1 REST API

Cache entries are set and retrieved by key, and there are two types of keys that can be used:
1. Content addressed storage (CAS), where the key is the lowercase SHA256 hash of the stored value.
   The REST API for these entries is: `/cas/<key>` or with an optional but ignored cache pool name: `/<pool>/cas/<key>`.
2. Action cache, where the key is an arbitrary 64 character lowercase hexadecimal string.
   Bazel uses the SHA256 hash of an action as the key, to store the metadata created by the action.
   The REST API for these entries is: `/ac/<key>` or with an optional cache pool name: `/<pool>/ac/<key>`.

Values are stored via HTTP PUT requests, and retrieved via GET requests. HEAD requests can be used to confirm
whether a key exists or not.

## gRPC API

bazel-remote also has experimental support for the ActionCache, ContentAddressableStorage and Capabilities services in the
[Bazel Remote Execution API v2](https://github.com/bazelbuild/remote-apis/blob/master/build/bazel/remote/execution/v2/remote_execution.proto),
and the corresponding parts of the [Byte Stream API](https://github.com/googleapis/googleapis/blob/master/google/bytestream/bytestream.proto).

## Usage

If a YAML configuration file is specified by the `--config_file` command line
flag or `BAZEL_REMOTE_CONFIG_FILE` environment variable, then other command
line flags and environment variables are ignored. Otherwise, the flags and
environment variables listed in the help text below can be specified (flags
override the corresponding environment variables).

### Command line flags

```
$ ./bazel-remote --help
NAME:
   bazel-remote - A remote build cache for Bazel

USAGE:
   bazel-remote [global options] command [command options] [arguments...]

DESCRIPTION:
   A remote build cache for Bazel.

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --config_file value           Path to a YAML configuration file. If this flag is specified then all other flags are ignored. [$BAZEL_REMOTE_CONFIG_FILE]
   --dir value                   Directory path where to store the cache contents. This flag is required. [$BAZEL_REMOTE_DIR]
   --max_size value              The maximum size of the remote cache in GiB. This flag is required. (default: -1) [$BAZEL_REMOTE_MAX_SIZE]
   --host value                  Address to listen on. Listens on all network interfaces by default. [$BAZEL_REMOTE_HOST]
   --port value                  The port the HTTP server listens on. (default: 8080) [$BAZEL_REMOTE_PORT]
   --grpc_port value             The port the EXPERIMENTAL gRPC server listens on. Set to 0 to disable. (default: 9092) [$BAZEL_REMOTE_GRPC_PORT]
   --profile_host value          A host address to listen on for profiling, if enabled by a valid --profile_port setting. (default: "127.0.0.1") [$BAZEL_REMOTE_PROFILE_HOST]
   --profile_port value          If a positive integer, serve /debug/pprof/* URLs from http://profile_host:profile_port. (default: 0) [$BAZEL_REMOTE_PROFILE_PORT]
   --htpasswd_file value         Path to a .htpasswd file. This flag is optional. Please read https://httpd.apache.org/docs/2.4/programs/htpasswd.html. [$BAZEL_REMOTE_HTPASSWD_FILE]
   --tls_enabled                 This flag has been deprecated. Specify tls_cert_file and tls_key_file instead. (default: false) [$BAZEL_REMOTE_TLS_ENABLED]
   --tls_cert_file value         Path to a pem encoded certificate file. [$BAZEL_REMOTE_TLS_CERT_FILE]
   --tls_key_file value          Path to a pem encoded key file. [$BAZEL_REMOTE_TLS_KEY_FILE]
   --idle_timeout value          The maximum period of having received no request after which the server will shut itself down. Disabled by default. (default: 0s) [$BAZEL_REMOTE_IDLE_TIMEOUT]
   --s3.endpoint value           The S3/minio endpoint to use when using S3 cache backend. [$BAZEL_REMOTE_S3_ENDPOINT]
   --s3.bucket value             The S3/minio bucket to use when using S3 cache backend. [$BAZEL_REMOTE_S3_BUCKET]
   --s3.prefix value             The S3/minio object prefix to use when using S3 cache backend. [$BAZEL_REMOTE_S3_PREFIX]
   --s3.access_key_id value      The S3/minio access key to use when using S3 cache backend. [$BAZEL_REMOTE_S3_ACCESS_KEY_ID]
   --s3.secret_access_key value  The S3/minio secret access key to use when using S3 cache backend. [$BAZEL_REMOTE_S3_SECRET_ACCESS_KEY]
   --s3.disable_ssl              Whether to disable TLS/SSL when using the S3 cache backend.  Default is false (enable TLS/SSL). (default: false) [$BAZEL_REMOTE_S3_DISABLE_SSL]
   --disable_http_ac_validation  Whether to disable ActionResult validation for HTTP requests.  Default is false (enable validation). (default: false) [$BAZEL_REMOTE_DISABLE_HTTP_AC_VALIDATION]
   --help, -h                    show help (default: false)
```

### Example configuration file

```yaml
# These two are the only required options:
dir: path/to/cache-dir
max_size: 100

host: localhost
# The port to use for HTTP/HTTPS:
#port: 8080
# The port to use for (experimental) gRPC support:
#grpc_port: 9092

# If profile_port is specified, then serve /debug/pprof/* URLs here:
#profile_host: 127.0.0.1
#profile_port: 7070

# If you want to require simple authentication:
#htpasswd_file: path/to/.htpasswd

# Specify a certificate if you want to use HTTPS:
#tls_cert_file: path/to/tls.cert
#tls_key_file:  path/to/tls.key

# If specified, bazel-remote should exit after being idle
# for this long. Time units can be one of: "s", "m", "h".
#idle_timeout: 45s

# If set to true, do not validate that ActionCache
# items are valid ActionResult protobuf messages.
#disable_http_ac_validation: false

# At most one of the proxy backends can be selected:
#
#gcs_proxy:
#  bucket: gcs-bucket
#  use_default_credentials: false
#  json_credentials_file: path/to/creds.json
#
#s3_proxy:
#  endpoint: minio.example.com:9000
#  bucket: test-bucket
#  prefix: test-prefix
#  access_key_id: EXAMPLE_ACCESS_KEY
#  secret_access_key: EXAMPLE_SECRET_KEY
#  disable_ssl: true
#
#http_proxy:
#  url: https://remote-cache.com:8080/cache

# If set to a valid port number, then serve /debug/pprof/* URLs here:
#profile_port: 7070
# IP address to use, if profiling is enabled:
#profile_host: 127.0.0.1
```

## Docker

### Prebuilt Image

We publish docker images to [DockerHub](https://hub.docker.com/r/buchgr/bazel-remote-cache/) that you can use with
`docker run`. The below command will start the remote cache on port `9090` with the default maximum cache size of
`5 GiB`.

```bash
$ docker pull buchgr/bazel-remote-cache
$ docker run -v /path/to/cache/dir:/data -p 9090:8080 buchgr/bazel-remote-cache
```

Note that you will need to change `/path/to/cache/dir` to a valid directory where the docker container can write to
and read from. If you want the docker container to run in the background pass the `-d` flag right after `docker run`.

You can change the maximum cache size by appending the `--max_size=N` flag with `N` being the max. size in Gibibytes.

### Build your own

The below command will build a docker image from source and install it into your local docker registry.

```bash
$ bazel run :bazel-remote-image
```

## Build a standalone Linux binary

```bash
$ bazel build :bazel-remote
```

### Authentication

In order to pass a `.htpasswd` and/or server key file(s) to the cache inside a docker container, you first need
to mount the file in the container and pass the path to the cache. The below example also configures TLS which is technically optional but highly recommended in order to not send passwords in plain text.

```bash
$ docker run -v /path/to/cache/dir:/data \
-v /path/to/htpasswd:/etc/bazel-remote/htpasswd \
-v /path/to/server_cert:/etc/bazel-remote/server_cert \
-v /path/to/server_key:/etc/bazel-remote/server_key \
-p 9090:8080 buchgr/bazel-remote-cache --tls_enabled=true \
--tls_cert_file=/etc/bazel-remote/server_cert --tls_key_file=/etc/bazel-remote/server_key \
--htpasswd_file /etc/bazel-remote/htpasswd --max_size=5
```

### Profiling

To enable pprof profiling, specify a port with `--profile_port`.

If running inside docker, you will also need to set `--profile_host` to a
value other than `127.0.0.1` (`--profile_host=` with an empty value should
work) and add a `-p` mapping to the docker run commandline for the port.

See [Profiling Go programs with pprof](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/)
for more details.

## Configuring Bazel

Please take a look at Bazel's documentation section on [remote
caching](https://docs.bazel.build/versions/master/remote-caching.html#run-bazel-using-the-remote-cache).
