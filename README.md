[![Stories in Ready](https://badge.waffle.io/buger/gor.png?label=ready)](https://waffle.io/buger/gor)
[![Build Status](https://travis-ci.org/buger/gor.png?branch=master)](https://travis-ci.org/buger/gor)

## About

Gor is a simple http traffic replication tool written in Go.
Its main goal is to replay traffic from production servers to staging and dev environments.


Now you can test your code on real user sessions in an automated and repeatable fashion.
**No more falling down in production!**

Here is basic workflow: The listener server catches http traffic and sends it to the replay server or saves to file.The replay server forwards traffic to a given address.


![Diagram](http://i.imgur.com/9mqj2SK.png)


## Examples

### Capture traffic from port
```bash
# Run on servers where you want to catch traffic. You can run it on each `web` machine.
sudo gor --input-raw :80 --output-tcp replay.local:28020

# Replay server (replay.local).
gor --input-tcp replay.local:28020 --output-http http://staging.com
```

### Using 1 Gor instance for both listening and replaying
It's recommended to use separate server for replaying traffic, but if you have enough CPU resources you can use single Gor instance.

```
sudo gor --input-raw :80 --output-http "http://staging.com"
```

### Guarantee of replay and HTTP input
Due to how traffic interception works, there is chance of missing requests. If you want guarantee that requests will be replayed you can use http input, but it will require changes in your app as well. 

```
sudo gor --input-http :28019 --output-http "http://staging.com"
```

Then in your application you should send copy (e.g. like reverse proxy) all incoming requests to Gor http input. 

### Following redirects
If you have a scenario where following redirects is usefull you can do it like with:

```
gor --input-tcp replay.local:28020 --output-http http://staging.com --output-http-redirects 10
```
The given example will follow up to 10 redirects per request.

## Advanced use

### Rate limiting
Every input and output support rate limiting. It can be useful if you want
forward only part of production traffic and not overload your staging
environment. 

There are 2 limiting algorithms: absolute or percentage based. 

Absolute: If for current second it reached specified requests limit - disregard the rest, on next second counter reseted.

Percentage: For input-file it will slowdown or speedup request execution, for the rest it will use random generator to decide if request pass or not based on weight you specified. 

You can specify your desired limit using the
"|" operator after the server address:

#### Limiting replay using absolute number
```
# staging.server will not get more than 10 requests per second
gor --input-tcp :28020 --output-http "http://staging.com|10"
```

#### Limiting listener using percentage based limiter
```
# replay server will not get more than 10% of requests 
# useful for high-load environments
gor --input-raw :80 --output-tcp "replay.local:28020|10%"
```

### Load testing

Currently it supported only by `input-file` and only when using percentage based limiter. Unlike default limiter for `input-file` instead of dropping requests it will slowdown or speedup request emitting.

```
# Replay from file on 2x speed 
gor --input-file "requests.gor|200%" --output-http "staging.com"
```


### Filtering 

#### Match on regexp of url
```
# only forward requests being sent to the api... domains
gor --input-raw :8080 --output-http staging.com --output-http-url-regexp ^www.
```

#### Filter based on regexp of header
```
# only forward requests with an api version of 1.0x
gor --input-raw :8080 --output-http staging.com --output-http-header-filter api-version:^1\.0\d
```

#### Filter based on hash of header
```
# send 1/32 of all users consistently to staging
gor --input-raw :8080 --output-http staging.com --output-http-header-hash-filter user-id:1/32
```

### Forward to multiple addresses

You can forward traffic to multiple endpoints. Just add multiple --output-* arguments.
```
gor --input-tcp :28020 --output-http "http://staging.com"  --output-http "http://dev.com"
```

#### Splitting traffic
By default it will send same traffic to all outputs, but you have options to equally split it:

```
gor --input-tcp :28020 --output-http "http://staging.com"  --output-http "http://dev.com" --split-output true
```

### Saving requests to file
You can save requests to file, and replay them later:
```
# write to file
gor --input-raw :80 --output-file requests.gor

# read from file
gor --input-file requests.gor --output-http "http://staging.com"
```

**Note:** Replay will preserve the original time differences between requests.

### Injecting headers

Additional headers can be injected/overwritten into requests during replay. This may be useful if you need to identify requests generated by Gor or enable feature flagged functionality in an application:

```
gor --input-raw :80 --output-http "http://staging.server" \
    --output-http-header "User-Agent: Replayed by Gor" \
    --output-http-header "Enable-Feature-X: true"
```

## Filtering HTTP methods

Requests not matching a specified whitelist can be filtered out. For example to strip non-nullipotent requests:

```
gor --input-raw :80 --output-http "http://staging.server" \
    --output-http-method GET \
    --output-http-method OPTIONS
```

### Basic Auth

If your development or staging environment is protected by Basic Authentication then those credentials can be injected in during the replay:

```
gor --input-raw :80 --output-http "http://user:pass@staging .com"
```

Note: This will overwrite any Authorization headers in the original request.

#### Rewrite the target urls based on a mapping
```
# rewrite url to match the following
gor --input-raw :8080 --output-http staging.com --output-http-rewrite-url /xml_test/interface.php:/api/service.do
```


## Stats 
### ElasticSearch 
For deep response analyze based on url, cookie, user-agent and etc. you can export response metadata to ElasticSearch. See [ELASTICSEARCH.md](ELASTICSEARCH.md) for more details.

```
gor --input-tcp :80 --output-http "http://staging.com" --output-http-elasticsearch "es_host:api_port/index_name"
```

## Additional help

Feel free to ask question directly by email or by creating github issue.

## Latest releases (including binaries)

https://github.com/buger/gor/releases

## Command line reference
`gor -h` output:
```
  -cpuprofile="": write cpu profile to file
  -input-dummy=[]: Used for testing outputs. Emits 'Get /' request every 1s
  -input-file=[]: Read requests from file: 
	gor --input-file ./requests.gor --output-http staging.com
  -input-http=[]: Read requests from HTTP, should be explicitly sent from your application:
	# Listen for http on 9000
	gor --input-http :9000 --output-http staging.com
  -input-raw=[]: Capture traffic from given port (use RAW sockets and require *sudo* access):
	# Capture traffic from 8080 port
	gor --input-raw :8080 --output-http staging.com
  -input-tcp=[]: Used for internal communication between Gor instances. Example: 
	# Receive requests from other Gor instances on 28020 port, and redirect output to staging
	gor --input-tcp :28020 --output-http staging.com
  -memprofile="": write memory profile to this file
  -output-dummy=[]: Used for testing inputs. Just prints data coming from inputs.
  -output-file=[]: Write incoming requests to file: 
	gor --input-raw :80 --output-file ./requests.gor
  -output-http=[]: Forwards incoming requests to given http address.
	# Redirect all incoming requests to staging.com address 
	gor --input-raw :80 --output-http http://staging.com
  -output-http-elasticsearch="": Send request and response stats to ElasticSearch:
	gor --input-raw :8080 --output-http staging.com --output-http-elasticsearch 'es_host:api_port/index_name'
  -output-http-header=[]: Inject additional headers to http reqest:
	gor --input-raw :8080 --output-http staging.com --output-http-header 'User-Agent: Gor'
  -output-http-header-filter=[]: A regexp to match a specific header against. Requests with non-matching headers will be dropped:
	 gor --input-raw :8080 --output-http staging.com --output-http-header-filter api-version:^v1
  -output-http-header-hash-filter=[]: Takes a fraction of requests, consistently taking or rejecting a request based on the FNV32-1A hash of a specific header. The fraction must have a denominator that is a power of two:
	 gor --input-raw :8080 --output-http staging.com --output-http-header-hash-filter user-id:1/4
  -output-http-method=[]: Whitelist of HTTP methods to replay. Anything else will be dropped:
	gor --input-raw :8080 --output-http staging.com --output-http-method GET --output-http-method OPTIONS
  -output-http-redirects=0: Enable how often redirects should be followed.
  -output-http-rewrite-url=[]: Rewrite the requst url based on a mapping:
	gor --input-raw :8080 --output-http staging.com --output-http-rewrite-url /xml_test/interface.php:/api/service.do
  -output-http-stats=false: Report http output queue stats to console every 5 seconds.
  -output-http-url-regexp=: A regexp to match requests against. Anything else will be dropped:
	 gor --input-raw :8080 --output-http staging.com --output-http-url-regexp ^www.
  -output-http-workers=-1: Gor uses dynamic worker scaling by default.  Enter a number to run a set number of workers.
  -output-tcp=[]: Used for internal communication between Gor instances. Example: 
	# Listen for requests on 80 port and forward them to other Gor instance on 28020 port
	gor --input-raw :80 --output-tcp replay.local:28020
  -output-tcp-stats=false: Report TCP output queue stats to console every 5 seconds.
  -split-output=false: By default each output gets same traffic. If set to `true` it splits traffic equally among all outputs.
  -stats=false: Turn on queue stats output
  -verbose=false: Turn on verbose/debug output
```

## Building from source

1. Setup standard Go environment http://golang.org/doc/code.html and ensure that $GOPATH environment variable properly set.
2. `go get github.com/buger/gor`.
3. `cd $GOPATH/src/github.com/buger/gor`
4. `go build` to get binary, or `go test` to run tests

## Development
Project contains Docker environment.

1. Build container: `make dbuild`
2. Run all tests: `make dtest`. Run specific test: `make dtest ARGS=-test.run=**regexp**`
3. Bash access to container: `make dbash`. Inside container you have python to run simple web server `python -m SimpleHTTPServer 8080` and `curl` to make http requests. 


## Questions and support 

All bug-reports and suggestions should go though Github Issues or our [Google Group](https://groups.google.com/forum/#!forum/gor-users). Or you can just send email to gor-users@googlegroups.com

If you have some private questions you can send direct mail to leonsbox@gmail.com

## FAQ

### What OS are supported?
For now only Linux based. *BSD (including MacOS is not supported yet, check https://github.com/buger/gor/issues/22 for details)

### Why does the `--input-raw` requires sudo or root access?
Listener works by sniffing traffic from a given port. It's accessible
only by using sudo or root access.

### I'm getting 'too many open files' error
Typical linux shell has a small open files soft limit at 1024. You can easily raise that when you do this before starting your gor replay process:
  
  ulimit -n 64000

More about ulimit: http://blog.thecodingmachine.com/content/solving-too-many-open-files-exception-red5-or-any-other-application

### What do the stats commands do?
Gor can report stats on the output-tcp and output-http request queues. Stats are reported to the console every 5 seconds in the form `latest,mean,max,count,count/second` by using the `-output-http-stats` and `-output-tcp-stats` options.

Examples:

```
2014/04/23 21:17:50 output_tcp:latest,mean,max,count,count/second
2014/04/23 21:17:50 output_tcp:0,0,0,0,0
2014/04/23 21:17:55 output_tcp:1,1,2,68,13
2014/04/23 21:18:00 output_tcp:1,1,2,92,18
2014/04/23 21:18:05 output_tcp:1,1,2,119,23
2014/04/23 21:18:10 output_tcp:1,0,1,95,19
2014/04/23 21:18:15 output_tcp:1,1,2,92,18
2014/04/23 21:18:20 output_tcp:1,1,2,108,21
2014/04/23 21:18:25 output_tcp:1,1,2,117,23
2014/04/23 21:18:30 output_tcp:1,1,2,113,22
2014/04/23 21:18:35 output_tcp:21,20,21,132,26
2014/04/23 21:18:40 output_tcp:100,99,100,99,19
```

```
Version: 0.8
2014/04/23 21:19:46 output_http:latest,mean,max,count,count/second
2014/04/23 21:19:46 output_http:0,0,0,0,0
2014/04/23 21:19:51 output_http:0,0,0,0,0
2014/04/23 21:19:56 output_http:0,0,0,0,0
2014/04/23 21:20:01 output_http:1,0,1,50,10
2014/04/23 21:20:06 output_http:1,1,4,72,14
2014/04/23 21:20:11 output_http:1,0,1,179,35
2014/04/23 21:20:16 output_http:1,0,1,148,29
2014/04/23 21:20:21 output_http:1,1,2,91,18
2014/04/23 21:20:26 output_http:1,1,2,150,30
2014/04/23 21:18:15 output_http:100,99,100,70,14
2014/04/23 21:18:21 output_http:100,99,100,55,11
2014/04/23 21:18:28 output_http:100,99,100,55,11
2014/04/23 21:18:34 output_http:100,99,100,57,11
2014/04/23 21:18:41 output_http:100,99,100,61,12
2014/04/23 21:18:48 output_http:100,99,100,56,11
2014/04/23 21:18:56 output_http:100,99,100,58,11
2014/04/23 21:19:01 output_http:100,99,100,31,6
2014/04/23 21:19:08 output_http:100,99,100,61,12
2014/04/23 21:19:15 output_http:100,99,100,64,12
2014/04/23 21:19:21 output_http:100,99,100,70,14
2014/04/23 21:19:28 output_http:100,99,100,61,12
2014/04/23 21:19:35 output_http:100,99,100,56,11
```

### How can I tell if I have bottlenecks?
Key areas that sometimes experience bottlenecks are the output-tcp and output-http functions which have internal queues for requests. Each queue has an upper limit of 100. Enable stats reporting to see if any queues are experiencing bottleneck behavior.
 
#### output-http bottlenecks
When running a Gor replay the output-http feature may bottleneck if:

  * the replay has inadequate bandwidth. If the replay is receiving or sending more messages than its network adapter can handle the output-http-stats  may report that the output-http queue is filling up. See if there is a way to upgrade the replay's bandwidth.
  * with `--output-http-workers` set to anything other than `-1` the `-output-http` target is unable to respond to messages in a timely manner. The http output workers which take messages off the output-http queue, process the request, and ensure that the request did not result in an error may not be able to keep up with the number of incoming requests. If the replay is not using dynamic worker scaling (`--output-http-workers=-1`)  The optimal number of output-http-workers can be determined with the formula `output-workers = (Average number of requests per second)/(Average target response time per second)`.

#### output-tcp bottlenecks
When using the Gor listener the output-tcp feature may bottleneck if:

  * the replay is unable to accept and process more requests than the listener is able generate. Prior to troubleshooting the output-tcp bottleneck, ensure that the replay target is not experiencing any bottlenecks. 
  * the replay target has inadequate bandwidth to handle all its incoming requests.  If a replay target's incoming bandwidth is maxed out the output-tcp-stats may report that the output-tcp queue is filling up. See if there is a way to upgrade the replay's bandwidth.

### The CPU average across my load-balanced targets is higher than the source
If you are replaying traffic from multiple listeners to a load-balanced target and you use sticky sessions, you may observe that the target servers have a higher CPU load than the listener servers. This may be because the sticky session cookie of the original load balancer is not honored by the target load balancer thus resulting in requests that would normally hit the same target server hitting different servers on the backend thus reducing some caching benefits gained via the load balancing.  Try running just one listener against one replay target and see if the CPU utilization comparison is more accurate.
 
### How does dynamic http worker scaling work?
By using the Gor setting `--output-http-workers=-1` Gor will create more http output workers when the http output queue length is greater than 10.  The number of workers created (N) is equal to the queue length at the time which it is checked and found to have a length greater than 10. The queue length is checked every time a message is written to the http output queue.  No more workers will be spawned until that request to spawn N workers is satisfied.  If a dynamic worker cannot process a message at that time, it will sleep for 100 milliseconds. If a dynamic worker cannot process a message for 2 seconds it dies.  

## Tuning

To achieve the top most performance you should tune the source server system limits:

    net.ipv4.tcp_max_tw_buckets = 65536
    net.ipv4.tcp_tw_recycle = 1
    net.ipv4.tcp_tw_reuse = 0
    net.ipv4.tcp_max_syn_backlog = 131072
    net.ipv4.tcp_syn_retries = 3
    net.ipv4.tcp_synack_retries = 3
    net.ipv4.tcp_retries1 = 3
    net.ipv4.tcp_retries2 = 8
    net.ipv4.tcp_rmem = 16384 174760 349520
    net.ipv4.tcp_wmem = 16384 131072 262144
    net.ipv4.tcp_mem = 262144 524288 1048576
    net.ipv4.tcp_max_orphans = 65536
    net.ipv4.tcp_fin_timeout = 10
    net.ipv4.tcp_low_latency = 1
    net.ipv4.tcp_syncookies = 0


## Contributing

1. Fork it
2. Create your feature branch (git checkout -b my-new-feature)
3. Commit your changes (git commit -am 'Added some feature')
4. Push to the branch (git push origin my-new-feature)
5. Create new Pull Request

## Companies using Gor

* [Granify](http://granify.com)
* [GOV.UK](https://www.gov.uk) ([Government Digital Service](http://digital.cabinetoffice.gov.uk/))
* [theguardian.com](http://theguardian.com)
* [TomTom](http://www.tomtom.com/)
* [3SCALE](http://www.3scale.net/)
* [Optionlab](http://www.opinionlab.com)
* To add your company drop me a line to github.com/buger or leonsbox@gmail.com
