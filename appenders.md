---
layout: default
---

## Appenders

Appenders are destinations that log messages can be written to.

Log messages can be written to one or more of the following destinations at the same time:

* Text File
* $stderr or $stdout ( any IO stream )
* Syslog
* Graylog
* Elasticsearch
* Splunk
* Loggly
* Logstash
* New Relic
* Bugsnag
* HTTP(S)
* MongoDB
* Logger, log4r, etc.

To ensure no log messages are lost it is recommend to use TCP over UDP for logging purposes.
Due to the architecture of Semantic Logger any performance difference between TCP and UDP will not
impact the performance of you application.

### Text File

Log to file with the standard formatter:

~~~ruby
SemanticLogger.add_appender('development.log')
~~~

Log to file with the standard colorized formatter:

~~~ruby
SemanticLogger.add_appender('development.log', &SemanticLogger::Appender::Base.colorized_formatter)
~~~

Log to file in JSON format:

~~~ruby
SemanticLogger.add_appender('development.log', &SemanticLogger::Appender::Base.json_formatter)
~~~

For performance reasons the log file is not re-opened with every call.
When the log file needs to be rotated, use a copy-truncate operation rather
than deleting the file.

### IO Streams

Semantic Logger can log data to any IO Stream instance, such as $stderr or $stdout

~~~ruby
# Log errors and above to standard error:
SemanticLogger.add_appender($stderror, :error)
~~~

### Syslog

Log to a local Syslog daemon

~~~ruby
SemanticLogger.add_appender(SemanticLogger::Appender::Syslog.new)
~~~

Log to a remote Syslog server using TCP:

~~~ruby
appender        = SemanticLogger::Appender::Syslog.new(
  server: 'tcp://myloghost:514'
)

# Optional: Add filter to exclude health_check, or other log entries
appender.filter = Proc.new { |log| log.message !~ /(health_check|Not logged in)/ }

SemanticLogger.add_appender(appender)
~~~

Log to a remote Syslog server using UDP:

~~~ruby
appender        = SemanticLogger::Appender::Syslog.new(
  server: 'udp://myloghost:514'
)

# Optional: Add filter to exclude health_check, or other log entries
appender.filter = Proc.new { |log| log.message !~ /(health_check|Not logged in)/ }

SemanticLogger.add_appender(appender)
~~~

If logging to a remote Syslog server using UDP, add the following line to your `Gemfile`:

~~~ruby
gem 'syslog_protocol'
~~~

If logging to a remote Syslog server using TCP, add the following lines to your `Gemfile`:

~~~ruby
gem 'syslog_protocol'
gem 'net_tcp_client'
~~~

Note: `:trace` level messages are mapped to `:debug`.

### Graylog

Send all log messages to a centralized logging server running [Graylog](https://www.graylog.org).

The Graylog appender retains all Semantic information when forwarding to Graylog.
( I.e. Data is sent as JSON and avoids the loss of semantic information which would gave occurred
had the log information been converted to text.)

For Rails applications, or running bundler, add the following line to the file `Gemfile`:

~~~ruby
gem 'gelf'
~~~

Install gems:

~~~
bundle install
~~~

If not using Bundler:

~~~
gem install gelf
~~~

To add to a Rails application that already uses [Rails Semantic Logger](rails.html)
create a file called `<Rails Root>/config/initializers/graylog.rb` with
the following contents and restart the application.

To use the TCP Protocol:

~~~ruby
appender        = SemanticLogger::Appender::Graylog.new(
  server:   'localhost',
  port:     12201,
  protocol: :tcp,
  facility: Rails.application.class.name
)
# Optional: Add filter to exclude health_check, or other log entries
appender.filter = Proc.new { |log| log.message !~ /(health_check|Not logged in)/ }

SemanticLogger.add_appender(appender)
~~~

Or, to use the UDP Protocol:

~~~ruby
appender        = SemanticLogger::Appender::Graylog.new(
  server:   'localhost',
  port:     12201,
  protocol: :udp,
  facility: Rails.application.class.name
)
# Optional: Add filter to exclude health_check, or other log entries
appender.filter = Proc.new { |log| log.message !~ /(health_check|Not logged in)/ }

SemanticLogger.add_appender(appender)
~~~

If not using Rails, the `facility` can be removed, or set to a custom string describing the application.

Note: `:trace` level messages are mapped to `:debug`.

### Splunk

In order to write messages to the Splunk HTTP Collector, follow the Splunk instructions
to enable the [HTTP Event Collector](http://dev.splunk.com/view/event-collector/SP-CAAAE7F).
The instructions also include information on how to generate a token that needs to be supplied below.

~~~ruby
appender = SemanticLogger::Appender::SplunkHttp.new(
  url:   'http://localhost:8088/services/collector/event',
  token: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
)
# Optional: Exclude health_check log entries
appender.filter = Proc.new { |log| log.message !~ /(health_check|Not logged in)/}

SemanticLogger.add_appender(appender)
~~~

Once log messages have been sent, open the Splunk web interface and select Search.
To start with limit the message source to the newly added host:

If the machine's name generating the messages is `hostname`, Search for: `host=hostname`

Then change the output list to table view and select only these "interesting columns":

* host
* duration
* name
* level
* message

If HTTPS is being used for the Splunk HTTP Collector, update the url accordingly:

~~~ruby
appender = SemanticLogger::Appender::SplunkHttp.new(
  url:   'https://localhost:8088/services/collector/event',
  token: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
)
~~~

### Elasticsearch

Forward all log messages to Elasticsearch.

Example:

~~~ruby
appender = SemanticLogger::Appender::Elasticsearch.new(
  url:   'http://localhost:9200'
)

# Optional: Exclude health_check log entries
appender.filter = Proc.new { |log| log.message !~ /(health_check|Not logged in)/}

SemanticLogger.add_appender(appender)
~~~

### Loggly

After signing up with Loggly obtain the token by logging into Loggly.com
Navigate to `Source Setup` -> `Customer Tokens` and copy the token

Replace `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` below with the above token.

~~~ruby
appender = SemanticLogger::Appender::Http.new(
  url: 'http://logs-01.loggly.com/inputs/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/tag/semantic_logger/'
)

# Optional: Exclude health_check log entries
appender.filter = Proc.new { |log| log.message !~ /(health_check|Not logged in)/}

SemanticLogger.add_appender(appender)
~~~

Once log messages have been sent, open the Loggly web interface and select Search.
To isolate the new messages start with the following search: `tag:"semantic_logger"`

In the Field Explorer change to Grid view and add the following fields using `Add as column to Grid`:

* host
* duration
* name
* level
* message

If HTTPS is being used for the Splunk HTTP Collector, update the url accordingly:

~~~ruby
appender = SemanticLogger::Appender::SplunkHttp.new(
  url:   'https://localhost:8088/services/collector/event',
  token: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
)
~~~

### Logstash

Forward log messages to Logstash.

For Rails applications, or running bundler, add the following line to the file `Gemfile`:

~~~ruby
gem 'logstash-logger'
~~~

Install gems:

~~~
bundle install
~~~

If not using Bundler:

~~~
gem install logstash-logger
~~~

If running Rails create an initializer with the following code, otherwise add the code
to your program:

~~~ruby
require 'logstash-logger'

# Use the TCP logger
# See https://github.com/dwbutler/logstash-logger for further options
log_stash       = LogStashLogger.new(type: :tcp, host: 'localhost', port: 5229)

appender        = SemanticLogger::Appender::Wrapper.new(log_stash)

# Optional: Add filter to exclude health_check, or other log entries
appender.filter = Proc.new { |log| log.message !~ /(health_check|Not logged in)/ }

SemanticLogger.add_appender(appender)
~~~

Note: `:trace` level messages are mapped to `:debug`.

### Bugsnag

Forward `:info`, `:warn`, or `:error` log messages to Bugsnag.

Note: Payload information is not filtered, so take care not to push any sensitive
information when logging with tags or a payload.

Configure Bugsnag following the [Ruby Bugsnag documentation](https://bugsnag.com/docs/notifiers/ruby#sending-handled-exceptions).

To add to a Rails application that already uses [Rails Semantic Logger](rails.html)
create a file called `<Rails Root>/config/initializers/bugsnag.rb` with
the following contents and restart the application.

Send `:error` messages to Bugsnag:

~~~ruby
SemanticLogger.add_appender(SemanticLogger::Appender::Bugsnag.new)
~~~

Or, for a standalone installation add the code above after initializing Semantic Logger.

Send `:warn` and `:error` messages to Bugsnag:

~~~ruby
SemanticLogger.add_appender(SemanticLogger::Appender::Bugsnag.new(:warn))
~~~

Send `:info`, `:warn` and `:error` messages to Bugsnag:

~~~ruby
SemanticLogger.add_appender(SemanticLogger::Appender::Bugsnag.new(:info))
~~~

### NewRelic

NewRelic supports Error Events in both it's paid and free subscriptions.

Adding the New Relic appender will by default send `:error` and `:fatal` log entries to
New Relic as error events.

Note: Payload information is not filtered, so take care not to push any sensitive
information when logging with tags or a payload.

For Rails applications, or running bundler, add the following line to the bottom of the file `Gemfile`:

~~~ruby
gem 'newrelic_rpm'
~~~

Install gems:

~~~
bundle install
~~~

If not using Bundler:

~~~
gem install newrelic_rpm
~~~

To add to a Rails application that already uses [Rails Semantic Logger](rails.html)
create a file called `<Rails Root>/config/initializers/new_relic.rb` with
the following contents and restart the application.

~~~ruby
SemanticLogger.add_appender(SemanticLogger::Appender::NewRelic.new)
~~~

To also send warnings to Bugsnag:

~~~ruby
SemanticLogger.add_appender(SemanticLogger::Appender::NewRelic.new(:warn))
~~~

### MongoDB

Write log messages as documents into a MongoDB capped collection.

For Rails applications, or running bundler, add the following line to the file `Gemfile`:

~~~ruby
gem 'mongo'
~~~

Install gems:

~~~
bundle install
~~~

If not using Bundler:

~~~
gem install mongo
~~~

To add to a Rails application that already uses [Rails Semantic Logger](rails.html)
create a file called `<Rails Root>/config/initializers/mongodb.rb` with
the following contents and restart the application.

~~~ruby
client   = Mongo::MongoClient.new('localhost', 27017)
database = client['test']

appender = SemanticLogger::Appender::MongoDB.new(
  db:              database,
  collection_size: 1024**3, # 1.gigabyte
  application:     Rails.application.class.name
)
SemanticLogger.add_appender(appender)
~~~

The following is written to Mongo:

~~~javascript
> db.semantic_logger.findOne()
{
	"_id" : ObjectId("53a8d5b99b9eb4f282000001"),
	"time" : ISODate("2014-06-24T01:34:49.489Z"),
	"host_name" : "appserver1",
	"pid" : null,
	"thread_name" : "2160245740",
	"name" : "Example",
	"level" : "info",
	"level_index" : 2,
	"application" : "my_application",
	"message" : "This message is written to mongo as a document"
}
~~~

### HTTP(S)

The HTTP appender supports sending JSON messages to most services that can accept log messages
in JSON format via HTTP or HTTPS.

~~~ruby
appender = SemanticLogger::Appender::Http.new(
  url: 'http://localhost:8088/path'
)
# Optional: Exclude health_check log entries
appender.filter = Proc.new { |log| log.message !~ /(health_check|Not logged in)/}

SemanticLogger.add_appender(appender)
~~~

To send messages via HTTPS, change the url to:

~~~ruby
appender = SemanticLogger::Appender::Http.new(
  url: 'https://localhost:8088/path'
)
~~~

The JSON being sent can be modified as needed.

Example, customize the JSON being sent:

~~~ruby
appender = SemanticLogger::Appender::Http.new(
  url: 'https://localhost:8088/path'
) do |log, request, logger|
  h = log.to_h
  h[:application] = logger.application_name
  h[:host]        = logger.host_name

  # Change time from iso8601 to seconds since epoch
  h[:timestamp] = log.time.utc.to_f

  # Render to JSON
  h.to_json
end
~~~

### Logger, log4r, etc.

Semantic Logger can log to other logging libraries:

* Replace the existing log library to take advantage of the extensive Semantic Logger interface
while still writing to the existing destinations.
* Or, write to a destination not currently supported by Semantic Logger.

~~~ruby
ruby_logger = Logger.new(STDOUT)

# Log to an existing Ruby Logger instance
SemanticLogger.add_appender(ruby_logger)
~~~

Note: `:trace` level messages are mapped to `:debug`.

### Multiple Appenders

Messages can be logged to multiple appenders at the same time.

Example, log to a local file and to a remote Syslog server such as syslog-ng over TCP:

~~~ruby
require 'semantic_logger'
SemanticLogger.default_level = :trace
SemanticLogger.add_appender('development.log')
SemanticLogger.add_appender(SemanticLogger::Appender::Syslog.new(:server => 'tcp://myloghost:514'))
~~~

### Appender Logging Levels

The logging level for each appender can be set explicitly. This supports:

* Only write a sub-set of messages to a particular destination.
    * For example, level of `:error` will only send error messages to this appender
      when other appenders may also be writing `:info`, etc.

Stand alone example:

~~~ruby
require 'semantic_logger'

# Set default log level for new logger instances
SemanticLogger.default_level = :info

# Log all warning messages and above to warnings.log
SemanticLogger.add_appender('log/warnings.log', :warn)

# Log all trace messages and above to trace.log
SemanticLogger.add_appender('log/trace.log', :trace)

logger = SemanticLogger['MyClass']
logger.level = :trace
logger.trace "This is a trace message"
logger.info "This is an info message"
logger.warn "This is a warning message"
~~~

The output is as follows:

~~~bash
==> trace.log <==
2013-08-02 14:15:56.733532 T [35669:70176909690580] MyClass -- This is a trace message
2013-08-02 14:15:56.734273 I [35669:70176909690580] MyClass -- This is an info message
2013-08-02 14:15:56.735273 W [35669:70176909690580] MyClass -- This is a warning message

==> warnings.log <==
2013-08-02 14:15:56.735273 W [35669:70176909690580] MyClass -- This is a warning message
~~~

### Standalone

Appenders can be created and logged to directly with the same API as for all other logging.
Supports logging specific activity in the current thread to a separate appender without it
being written to the appenders that have been registered in Semantic Logger.

Example:

~~~ruby
require 'semantic_logger'

appender = SemanticLogger::Appender::File.new('separate.log', :info)

# Use appender directly, without using global Semantic Logger
appender.warn 'Only send this to separate.log'

appender.benchmark_info 'Called supplier' do
  # Call supplier ...
end
~~~

Note: Once an appender has been registered with Semantic Logger it must not be called
      directly, otherwise non-deterministic concurrency issues will arise when it is used across threads.

### [Next: Signals ==>](signals.html)