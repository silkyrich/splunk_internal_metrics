# splunk_internal_metrics

A Splunk app that transforms varies Splunk generated metrics values into a metrics store

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
 
1)       send to _internal index (still struggling to get this right so that remains the same as the true original)
2)       parse this into a metric = <metric.name>.<attribute>=value_number, it’s a number so write to metric store
3)       parse this into a metric = <metric.name>.<attribute>=value_string, it’s a string so route to nullQueue
4)       parse this into a metric = <metric.name>.<attribute>=value_number, it’s a number so write to metric store
5)       this is “finished” so route to nullQueue
