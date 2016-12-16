# Prometheus Pushgateway

The Prometheus Pushgateway exists to allow ephemeral and batch jobs to
expose their metrics to Prometheus. Since these kinds of jobs may not
exist long enough to be scraped, they can instead push their metrics
to a Pushgateway. The Pushgateway then exposes these metrics to
Prometheus.

### Feature additions of this branch

* [contributor: Paul K. Gerke](http://www.diagnijmegen.nl/index.php/Person?name=Paul_Konstantin_Gerke):
    * Stale timers are needed in our case because we are not able to
      control our network setup and we want to establish one-directional
      traffic through a firewall. A push-gateway is very useful in this 
      case. Even though sketched as an anti-pattern by the original this
      is a required workaround in some cases. How to use them see below:


## Non-goals

The Pushgateway is explicitly not an _aggregator or distributed counter_ but
rather a metrics cache. It does not have a statsd-like semantics. The metrics
pushed are exactly the same as you would present for scraping in a permanently
running program.

For machine-level metrics, the
[textfile](https://github.com/prometheus/node_exporter/blob/master/README.md#textfile-collector)
collector of the Node exporter is usually more appropriate. The Pushgateway is
intended for service-level metrics.

The Pushgateway is not an _event store_. While you can use Prometheus as a data
source for
[Grafana annotations](http://docs.grafana.org/reference/annotations/), tracking
something like release events has to happen with some event-logging framework.


## Run it

Download binary releases for your platform from the
[release page](https://github.com/prometheus/pushgateway/releases) and unpack
the tarball.

If you want to compile yourself from the sources, you need a working Go
setup. Then use the provided Makefile (type `make`).

For the most basic setup, just start the binary. To change the address
to listen on, use the `-web.listen-address` flag. The `-persistence.file` flag
allows you to specify a file in which the pushed metrics will be
persisted (so that they survive restarts of the Pushgateway).

## Use it

### Configure the Pushgateway as a target to scrape

The Pushgateway has to be configured as a target to scrape by Prometheus, using
one of the usual methods. _However, you should always set `honor_labels: true`
in the scrape config_ (see [below](#about-the-job-and-instance-labels) for a
detailed explanation).

### Libraries

Prometheus client libraries should have a feature to push the
registered metrics to a Pushgateway. Usually, a Prometheus client
passively presents metric for scraping by a Prometheus server. A
client library that supports pushing has a push function, which needs
to be called by the client code. It will then actively push the
metrics to a Pushgateway, using the API described below.

### Command line

Using the Prometheus text protocol, pushing metrics is so easy that no
separate CLI is provided. Simply use a command-line HTTP tool like
`curl`. Your favorite scripting language has most likely some built-in
HTTP capabilities you can leverage here as well.

*Caveat: Note that in the text protocol, each line has to end with a
line-feed character (aka 'LF' or '\n'). Ending a line in other ways,
e.g. with 'CR' aka '\r', 'CRLF' aka '\r\n', or just the end of the
packet, will result in a protocol error.*

Examples:

* Push a single sample into the group identified by `{job="some_job"}`:

        echo "some_metric 3.14" | curl --data-binary @- http://pushgateway.example.org:9091/metrics/job/some_job

  Since no type information has been provided, `some_metric` will be of type `untyped`.

* Push something more complex into the group identified by `{job="some_job",instance="some_instance"}`:

        cat <<EOF | curl --data-binary @- http://pushgateway.example.org:9091/metrics/job/some_job/instance/some_instance
        # TYPE some_metric counter
        some_metric{label="val1"} 42
        # This one even has a timestamp (but beware, see below).
        some_metric{label="val2"} 34 1398355504000
        # TYPE another_metric gauge
        # HELP another_metric Just an example.
        another_metric 2398.283
        EOF

  Note how type information and help strings are provided. Those lines
  are optional, but strongly encouraged for anything more complex.

* Delete all metrics grouped by job and instance:

        curl -X DELETE http://pushgateway.example.org:9091/metrics/job/some_job/instance/some_instance

* Delete all metrics grouped by job only:

        curl -X DELETE http://pushgateway.example.org:9091/metrics/job/some_job

### Stale timers extension

* Stale timers are attached to each metric and can be configured with a
  "magic" setting (for backwards compatibility). If not used, stale lifetimes
  default to "Forever". To set them explicitly include the new 
  "#!set stale_timeout = [null|number]" setting
  
        cat <<EOF | curl --data-binary @- http://pushgateway.example.org:9091/metrics/job/some_job/instance/some_instance
        # metric with lifetime = forever (default)
        metric1{label="val1"} 42
        
        # All following metrics will have a lifetime of 10 seconds
        #!set stale_timeout = 10
        
        # This metric will only be stored in the push gateway for 10 seconds
        metric2{label="val2"} 1337
        
        # Reset stale timeout to forever (case sensitive!)
        #!set stale_timeout = null
        
        # This metric will be available forever
        metric3{label="val3"} 1234
        EOF

* Stale timeout information is added to the status-dashboard of the
  pushgateway, so just visit http://your-host:9091/ to see if
  the stale timers are configured correctly.

### About the job and instance labels

The Prometheus server will attach a `job` label and an `instance` label to each
scraped metric. The value of the `job` label comes from the scrape
configuration. When you configure the Pushgateway as a scrape target for your
Prometheus server, you will probably pick a job name like `pushgateway`. The
value of the `instance` label is automatically set to the host and port of the
target scraped. Hence, all the metrics scraped from the Pushgateway will have
the host and port of the Pushgateway as the `instance` label and a `job` label
like `pushgateway`. The conflict with the `job` and `instance` labels you might
have attached to the metrics pushed to the Pushgateway is solved by renaming
those labels to `exported_job` and `exported_instance`.

However, this behavior is usually undesired when scraping a
Pushgateway. Generally, you would like to retain the `job` and `instance`
labels of the metrics pushed to the Pushgateway. That's why you have set
`honor_labels: true` in the scrape config for the Pushgateway. It enables the
desired behavior. See the
[documentation](https://prometheus.io/docs/operating/configuration/#scrape_config)
for details.

This leaves us with the case where the metrics pushed to the Pushgateway do not
feature an `instance` label. This case is quite commen as the pushed metrics
are often on a service level and therefore not related to a particular
instance. Even with `honor_labels: true`, the Prometheus server will attach an
`instance` label if no `instance` label has been set in the first
place. Therefore, if a metric is pushed to the Pushgateway without an instance
label (and without instance label in the grouping key, see below), the
Pushgateway will export it with an emtpy instance label (`{instance=""}`),
which is equivalent to having no `instance` label at all but prevents the
server from attaching one.

### About timestamps

If you push metrics at time *t*<sub>1</sub>, you might be tempted to believe
that Prometheus will scrape them with that same timestamp
*t*<sub>1</sub>. Instead, what Prometheus attaches as a timestamp is the time
when it scrapes the Pushgateway. Why so?

In the world view of Prometheus, a metric can be scraped at any
time. A metric that cannot be scraped has basically ceased to
exist. Prometheus is somewhat tolerant, but if it cannot get any
samples for a metric in 5min, it will behave as if that metric does
not exist anymore. Preventing that is actually one of the reasons to
use a push gateway. The push gateway will make the metrics of your
ephemeral job scrapable at any time. Attaching the time of pushing as
a timestamp would defeat that purpose because 5min after the last
push, your metric will look as stale to Prometheus as if it could not
be scraped at all anymore. (Prometheus knows only one timestamp per
sample, there is no way to distinguish a 'time of pushing' and a 'time
of scraping'.)

You can still force Prometheus to attach a different timestamp by
using the optional timestamp field in the exchange format. However,
there are very few use cases where that would make
sense. (Essentially, if you push more often than every 5min, you
could attach the time of pushing as a timestamp.)

## API

All pushes are done via HTTP. The interface is vaguely REST-like.

### URL

The default port the push gateway is listening to is 9091. The path looks like

    /metrics/job/<JOBNAME>{/<LABEL_NAME>/<LABEL_VALUE>}

`<JOBNAME>` is used as the value of the `job` label, followed by any
number of other label pairs (which might or might not include an
`instance` label). The label set defined by the URL path is used as a
grouping key. Any of those labels already set in the body of the
request (as regular labels, e.g. `name{job="foo"} 42`)
_will be overwritten to match the labels defined by the URL path!_

Note that `/` cannot be used as part of a label value or the job name,
even if escaped as `%2F`. (The decoding happens before the path
routing kicks in, cf. the Go documentation of
[`URL.Path`](http://golang.org/pkg/net/url/#URL).)

### Deprecated URL

There is a _deprecated_ version of the URL path, using `jobs` instead
of `job`:

    /metrics/jobs/<JOBNAME>[/instances/<INSTANCENAME>]

If this version of the URL path is used _with_ the `instances` part,
it is equivalent to the URL path above with an `instance` label, i.e.

    /metrics/jobs/foo/instances/bar

is equivalent to

    /metrics/job/foo/instance/bar

(Note the missing pluralizations.)

However, if the `instances` part is missing, the Pushgateway will
automatically use the IP number of the pushing host as the 'instance'
label, and grouping happens by 'job' and 'instance' labels.

Example: Pushing metrics from host 1.2.3.4 using the deprecated URL path

    /metrics/jobs/foo

is equivalent to pushing using the URL path

    /metrics/job/foo/instance/1.2.3.4

### `PUT` method

`PUT` is used to push a group of metrics. All metrics with the
grouping key specified in the URL are replaced by the metrics pushed
with `PUT`.

The body of the request contains the metrics to push either as delimited binary
protocol buffers or in the simple flat text format (both in version 0.0.4, see
the
[data exposition format specification](https://docs.google.com/document/d/1ZjyKiKxZV83VI9ZKAXRGKaUKK2BIWCT7oiGBKDBpjEY/edit?usp=sharing)).
Discrimination between the two variants is done via the `Content-Type`
header. (Use the value `application/vnd.google.protobuf;
proto=io.prometheus.client.MetricFamily; encoding=delimited` for protocol
buffers, otherwise the text format is tried as a fall-back.)

The response code upon success is always 202 (even if the same
grouping key has never been used before, i.e. there is no feedback to
the client if the push has replaced an existing group of metrics or
created a new one).

_If using the protobuf format, do not send duplicate MetricFamily
proto messages (i.e. more than one with the same name) in one push, as
they will overwrite each other._

A successfully finished request means that the pushed metrics are
queued for an update of the storage. Scraping the push gateway may
still yield the old results until the queued update is
processed. Neither is there a guarantee that the pushed metrics are
persisted to disk. (A server crash may cause data loss. Or the push
gateway is configured to not persist to disk at all.)

### `POST` method

`POST` works exactly like the `PUT` method but only metrics with the
same name as the newly pushed metrics are replaced (among those with
the same grouping key).

### `DELETE` method

`DELETE` is used to delete metrics from the push gateway. The request
must not contain any content. All metrics with the grouping key
specified in the URL are deleted.

The response code upon success is always 202. The delete
request is merely queued at that moment. There is no guarantee that the
request will actually be executed or that the result will make it to
the persistence layer (e.g. in case of a server crash). However, the
order of `PUT`/`POST` and `DELETE` request is guaranteed, i.e. if you
have successfully sent a `DELETE` request and then send a `PUT`, it is
guaranteed that the `DELETE` will be processed first (and vice versa).

Deleting a grouping key without metrics is a no-op and will not result
in an error.

**Caution:** Up to version 0.1.1 of the Pushgateway, a `DELETE` request
using the following path in the URL would delete _all_ metrics with
the job label 'foo':

    /metrics/jobs/foo

Newer versions will interpret the above URL path as deprecated (see
above). Consequently, they will add the IP number of the client as an
instance label and then delete all metrics with the corresponding job
and instance label as their grouping key.

## Development

The normal binary embeds the files in `resources`. For development
purposes, it is handy to have a running binary use those files
directly (so that you can see the effect of changes immediately). To
switch to direct usage, type `make bindata-debug` just before
compiling the binary. Switch back to "normal" mode by typing `make
bindata-embed`. (Just `make` after a resource has changed will result
in the same.)

##  Contributing

Relevant style guidelines are the [Go Code Review
Comments](https://code.google.com/p/go-wiki/wiki/CodeReviewComments)
and the _Formatting and style_ section of Peter Bourgon's [Go:
Best Practices for Production
Environments](http://peter.bourgon.org/go-in-production/#formatting-and-style).

## Using Docker

You can deploy the Pushgateway using the [prom/pushgateway](https://registry.hub.docker.com/u/prom/pushgateway/) Docker image.

For example:

```bash
docker pull prom/pushgateway

docker run -d -p 9091:9091 prom/pushgateway
```


[travis]: https://travis-ci.org/prometheus/pushgateway
[hub]: https://hub.docker.com/r/prom/pushgateway/
[circleci]: https://circleci.com/gh/prometheus/pushgateway
[quay]: https://quay.io/repository/prometheus/pushgateway
