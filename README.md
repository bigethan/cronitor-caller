# Cronitor Node Library

[![Build Status](https://travis-ci.org/cronitorio/cronitor-js.svg?branch=master)](https://travis-ci.org/cronitorio/cronitor-js) [![codecov](https://codecov.io/gh/cronitorio/cronitor-js/branch/2.0.0/graph/badge.svg)](https://codecov.io/gh/cronitorio/cronitor-js)

[Cronitor](https://cronitor.io/) provides dead simple monitoring for cron jobs, daemons, data pipelines, queue workers, and anything else that can send or receive an HTTP request. The Cronitor Node library provides convenient access to the Cronitor API from applications written in server-side JavaScript.

## Documentation
See our [API docs](https://cronitor.io/docs/api) for a detailed reference information about the APIs this library uses for configuring monitors and sending telemetry pings.

## Installation
```
npm install cronitor --save
# or
yarn add cronitor
```


## Usage

### Authentication

The package needs to be configured with your account's `API key`, which is available on the [account settings](https://cronitor.io/settings) page. You can also optionally specify an `Api Version` (default: account default) and `Environment` (default: account default).


These can be supplied using the environment variables `CRONITOR_API_KEY`, `CRONITOR_API_VERSION`, `CRONITOR_ENVIRONMENT` or passed to the Cronitor function.

```javascript
const cronitor = require('cronitor')('cronitor_api_key', {apiVersion: '2020-10-01', environment: 'staging'});
```

You can also use a YAML config file to manage all of your monitors (_see Create and Update Monitors section below_). The path to this file can be supplied using the enviroment variable `CRONITOR_CONFIG` or set directly on the Cronitor object.

```javascript
const cronitor = require('cronitor')('cronitor_api_key', {config: './path/to/cronitor.yaml'});
// or
cronitor.readConfig('./path/to/cronitor.yaml')
```


### Integrate with Cron Libraries
If you are using a library like [node-cron](https://github.com/node-cron/node-cron) or [cron](https://github.com/kelektiv/node-cron), this package provides a lightweight wrapper to enable easy monitoring integration.

```javascript
const cron = require('cronitor')('cronitor_api_key');
const nodeCron = require('node-cron');

cron.wraps(nodeCron);

// note the first parameter is now the key that cronitor will use
// to create a new monitor the first time the function is called.
cron.schedule('SendWelcomeEmail', '*/5 * * * *', function() {
    console.log('Sending welcome email to new sign ups every five minutes.');
});
```

### Monitor Async Functions
Cronitor can wrap any function with telemetry pings as well, and can be used with any node system.

```javascript
const cronitor = require('cronitor')('cronitor_api_key');

let asyncWorker = cronitor.wrap(async function() {
    // do some async work
});

// cronitor will track the start time and the end time (promise resolved)
await asyncWorker();
```

### Monitor long running processes
If you have a long running process (Control-Loop, Daemon, Worker, etc) you might not care about the lifecycle (start/end),
and instead wish to record counts/error counts of these events instead. Use the `Event`
object to synchronously record loop ticks and asynchronously batch report these events to Cronitor.
The following example uses [sqs-consumer](https://github.com/bbc/sqs-consumer).

```javascript
const cronitor = require('cronitor')('cronitor_api_key');
const { Consumer } = require('sqs-consumer');

event = new cronitor.Event('monitor-key');

const app = Consumer.create({
  queueUrl: 'https://sqs.eu-west-1.amazonaws.com/account-id/queue-name',
  pollingWaitTimeMs: 100 // duration to wait before repolling the queue (defaults to 0).
  handleMessage: async (message) => {
    // do some work with `message`
  }
});

// Consumer is an event emitter and will emit one of the below events each time it is called.
// a message was processed
app.on('processed_message', () => {
    // increment the tick counter, no other side effects.
    event.tick();
});

 // the queue is empty
app.on('empty', () => {
    // record a tick and also record that no message was processed
    event.tick(0);
});

// an error occurred connectiong to SQS
app.on('error', (err) => {
    // .error is a special "tick" method for reporting error counts.
    // Use it to tell Cronitor your program is still running, but encountering errors.
    // Error rate alert thresholds are configurable.
  event.error();
});
oncon
app.start();
```

### Sending Telemetry Pings

If you simply want to send a heartbeat event or want finer control over when/how [telemetry pings](https://cronitor.io/docs/ping-api) are sent,
you can create a monitor instance and call `.ping` directly.

```javascript
// if authenticated pings are enabled, add your apiKey when creating a Ping object
const monitor = new cronitor.Monitor('heartbeat-monitor');
monitor.ping()

// optional params can be passed as an object.
// for a complete list see https://cronitor.io/docs/ping-api
monitor.ping({
    state: 'run|complete|fail|ok', // run/complete|fail used to measure lifecycle of a job.
    env: '', // the environment this is running in (development, staging, production)
    message: '', // optional message that will be displayed in alerts as well as monitor activity panel on your dashboard.
    metrics: {
        duration: 100,
        count: 4500,
        error_count: 10
    }
});
```

### Pause, Reset, Delete

```javascript
const monitor = new cronitor.Monitor('heartbeat-monitor');

monitor.pause(24) // pause alerting for 24 hours
monitor.unpause() // alias for .pause(0)
monitor.ok() // reset to a passing state alias for monitor.ping({state: ok})
monitor.delete() // destroy the monitor
```

## Create and Update Monitors

You can create monitors programatically using the `Monitor` object.
For details on all of the attributes that can be set see the [Monitor API](https://cronitor.io/docs/monitor-api) documentation.


```javascript
const cronitor = require('cronitor')('apiKey123');

const jobMonitor = cronitor.Monitor.put({
    type: 'job',
    key: 'send-customer-invoices',
    schedule: '0 0 * * *',
    assertions: [
        'metric.duration < 5 min'
    ],
    notify: ['devops-alerts-slack']
});

const uptimeMonitor = cronitor.Monitor.put({
    type: 'synthetic',
    key: 'Orders Api Uptime',
    schedule: 'every 45 seconds',
    assertions: [
        'response.code = 200',
        'response.time < 1.5s',
        'response.json "open_orders" < 2000'
    ]
})
```

You can also manage all of your monitors via a YAML config file.
This can be version controlled and synced to Cronitor as part of
a deployment pipeline or system update.

```javascript
const cronitor = require('cronitor')('apiKey123');

cronitor.readConfig('./cronitor.yaml'); // parse the yaml file of monitors

cronitor.validateConfig(); // send monitors to Cronitor for configuration validation

cronitor.applyConfig(); // sync the monitors from the config file to Cronitor
```

The `cronitor.yaml` file accepts the following attributes:

```yaml
api_key: 'optionally read Cronitor api_key from here'
api_version: 'optionally read Cronitor api_version from here'
environment: 'optionally set an environment for telemetry pings'

# configure all of your monitors with type "job"
# you may omit the type attribute and the key
# of each object will be set as the monitor key
jobs:
    nightly-database-backup:
        schedule: 0 0 * * *
        notify:
            - devops-alert-pagerduty
        assertions:
            - metric.duration < 5 minutes

    send-welcome-email:
        schedule: every 10 minutes
        assertions:
            - metric.count > 0
            - metric.duration < 30 seconds

# configure all of your monitors with type "synthetic"
synthetics:
    cronitor-homepage:
        request:
            url: https://cronitor.io
            regions:
                - us-east-1
                - eu-central-1
                - ap-northeast-1
        assertions:
            - response.code = 200
            - response.time < 2s

    cronitor-ping-api:
        request:
            url: https://cronitor.link/ping
        assertions:
            - response.body contains ok
            - response.time < .25s

events:
    production-deploy:
        notify:
            alerts: ['deploys-slack']
            events: true # send alert when the event occurs

```


## Contributing

Pull requests and features are happily considered! By participating in this project you agree to abide by the [Code of Conduct](http://contributor-covenant.org/version/2/0).

### To contribute

Fork, then clone the repo:

    git clone git@github.com:your-username/cronitor-js.git

Set up your machine:

    npm install

Make sure the tests pass:

    npm test

Make your change. Add tests for your change. Make the tests pass:

    npm test


Push to your fork and [submit a pull request]( https://github.com/cronitorio/cronitor-js/compare/)

