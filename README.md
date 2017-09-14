# splunk_internal_metrics

A Splunk app that transforms varies Splunk generated metrics values into a metrics store. This is achieved by cloning events in the pipeline and transforming them in the process. This allows us to ingest the same event multiple times and each time it has been modified to be a little different.

Any log file that contains AV pair where the value is a string, it is used as dimension, and any AV pair where the value is a number is accepted as a value (with the dimensions included).

Anything log line that fails to be turned into a metric or dimension will appear in metrics_internal_spam. Anything that is successfully converted to a metric will be stored in metrics_internal.

# How it works under the hood

Consider a log line as such:
<date> <preamble>, <attribute=value_number>, <attribute=value_string>, <attribute=value_number>
 
The date is parsed, so we don’t need it any more. We look at the preamble to create the metric_prefix. We scan the log line looking for attributes with a string values and then write these as indexed fields into _meta to become dimensions.
 
We transform the log line with regex to look like this:
<metric_prefix>#<attribute=value_number>, <attribute=value_string>, <attribute=value_number>
 
We clone the above line to pop the first attribute pair
<metric_prefix>#<attribute=<attribute=value_string>, <attribute=value_number>
 
We clone again:
<metric_prefix>#<attribute=<attribute=value_number>
 
We clone again, but regex fails so we write “finished” into the _raw:
finished
 
We have now the original event and four modified and cloned events:
 
1. <date> <preamble>, <attribute=value_number>, <attribute=value_string>, <attribute=value_number>
2. <metric.name>#<attribute=value_number>, <attribute=value_string>, <attribute=value_number>
3. <metric.name>#<attribute=<attribute=value_string>, <attribute=value_number>
4. <metric.name>#<attribute=<attribute=value_number>
5. finished
 
Actions now are
 
1. send to _internal index (still struggling to get this right so that remains the same as the true original with both source and sourcetype)
2. parse this into a metric = <metric.name>.<attribute>=value_number, it’s a number so write to metric store
3. parse this into a metric = <metric.name>.<attribute>=value_string, it’s a string so route to nullQueue
4. parse this into a metric = <metric.name>.<attribute>=value_number, it’s a number so write to metric store
5. this is “finished” so route to nullQueue
