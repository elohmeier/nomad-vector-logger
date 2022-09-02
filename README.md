<a href="https://zerodha.tech"><img src="https://zerodha.tech/static/images/github-badge.svg" align="right" /></a>

# nomad-vector-logger

A daemon which continuously watches jobs running in a [Nomad](https://www.nomadproject.io/) cluster and templates out a CSV file of current allocations. [Vector](https://vector.dev/) uses [Enrichment tables](https://vector.dev/highlights/2021-11-18-csv-enrichment/) which allows Vector to enrich the logs with metadata from an external source. For now, Vector can use CSVs to lookup records based on certain keys and get the metadata to add to logs.
## Why

### The Problem

Currently, Nomad stores all application logs inside `$NOMAD_DATA_DIR/$NOMAD_ALLOC_DIR/logs/` directory. The limitation here is that these logs don't come with any metadata about the task/job/allocation. If you're running multiple applications on the same host, a log collecting agent cannot distinguish these logs. For `docker` driver, this is a solved problem by configuring the `log_driver` for the container.

However for `raw_exec` and `exec` this still seems to be an issue until [10219](https://github.com/hashicorp/nomad/issues/10219) is addressed.

### The Solution

- Nomad provides an [Events stream](https://github.com/mr-karan/nomad-events-sink) which continuously streams events from the Nomad cluster.`nomad-vector-logger` is a Go daemon which subscribes to the Events stream on topic `Allocation` and listens for allocation updates.
- It then **templates** out a CSV file to list the current allocations and their metadata such as `NodeID`, `Namesapce`, `Task Name`, `JobName`, `AllocID` etc.
- Vector reads the CSV and uses enrichment tables to lookup the CSV file for enriching logs with Nomad metadata.

You can see a sample [CSV file](./examples/nomad_remap.csv) that is generated by this program. Vector is configured to lookup this CSV for collecting logs as demonstrated [here](./examples/vector.tpl.toml).

#### Before

Logs without any metdata on `/opt/nomad/data/alloc/$ALLOC_ID/alloc/logs`:

```
==> proxy.stdout.0 <==
192.168.29.76 - - [02/Sep/2022:08:58:35 +0000] "GET / HTTP/1.1" 200 27 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:103.0) Gecko/20100101 Firefox/103.0" "-"
```

#### After

This is an example JSON log collected from `nginx` task running with `raw_exec` task driver on Nomad, collected using `vector`:

```json
{
    "file": "/opt/nomad/data/alloc/d91eded6-3820-c507-c571-2cb0a4237912/alloc/logs/proxy.stdout.0",
    "host": "pop-os",
    "message": "192.168.29.76 - - [02/Sep/2022:08:58:35 +0000] \"GET / HTTP/1.1\" 200 27 \"-\" \"Mozilla/5.0 (X11; Linux x86_64; rv:103.0) Gecko/20100101 Firefox/103.0\" \"-\"",
    "nomad_alloc_id": "d91eded6-3820-c507-c571-2cb0a4237912",
    "nomad_group_name": "nginx",
    "nomad_job_name": "nginx",
    "nomad_namespace": "default",
    "nomad_node_name": "pop-os",
    "nomad_task_name": "proxy",
    "source_type": "file",
    "timestamp": "2022-09-02T08:58:43.598703195Z"
}
```

## Deploy

- This is meant to run inside a Nomad cluster and should have proper ACL to listen to `Allocation:*` events.
- This is meant to be run as a `system` job. Each allocation of this program is responsible to configure `vector` for the allocations running on the host.
- Vector should be deployed as a system job as well and should have access to the same directory that this program uses to generate the files.

You can choose one of the various deployment options:

### Binary

Grab the latest release from [Releases](https://github.com/mr-karan/nomad-vector-logger/releases).

To run:

```
$ ./nomad-vector-logger.bin --config config.toml
```

### Nomad

View a sample deployment file at [examples/deployment.nomad](./examples/deployment.nomad).

### Docker 

`ghcr.io/mr-karan/nomad-vector-logger`

## Configuration

Refer to [config.sample.toml](./config.sample.toml) for a list of configurable values.

### Environment Variables

All config variables can also be populated as env vairables by prefixing `NOMAD_VECTOR_LOGGER_` and replacing `.` with `__`.

For eg: `app.data_dir` becomes `NOMAD_VECTOR_LOGGER_app__data_dir`.

## Contribution

Please feel free to open a new issue for bugs, feedback etc.

## LICENSE

[LICENSE](./LICENSE)
