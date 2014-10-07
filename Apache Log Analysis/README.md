# Apache one-liners for Log file analysis

The following scripts assume that:

* The ``Combined Log Format``, or an extension of it is used. 
* The log file is at location ``log/access``. Modify according to the actual log file location.
* The analysis is performed on the last 1 million requests. Modify accordingly.

## Most frequent requests

### Most frequent requests (method + URL without querystring)

	tail -n 1000000 log/access | awk -F"[ ?]" '{print $6, $7}' | sort | uniq -c | sort -rn | head -20

Notice how ``awk`` uses  the regular expression``"[ ?]"`` as a separator. 
This means that both characters ``space`` and ``?`` are separators, which causes the url to be split if a ``?`` character is found, and therefore separates it from the querystring.

### Most frequent requests (method + URL with querystring)

	tail -n 1000000 log/access | awk '{print $6, $7}' | sort | uniq -c | sort -n -r | head -n 20

### Most frequest referrers

	tail -n 1000000 log/access | awk -F\" '{print $6}' | sort | uniq -c | sort -rn | head -20

## Slowest requests
Note: Requires an extension of the ``Combined Log Format`` that includes the ``%D`` parameter. For example:

	LogFormat "%h %l %u %t \"%r\" %>s %b %D \"%{Referer}i\" \"%{User-agent}i\""

### Slowest requests

	tail -n 1000000 log/access | awk -F" " '{print $4, $5, $6, $7, $10, $11}' |  sort -h -r -k6 | head -n 20

### Slowest requests with a status code other than 500

	tail -n 1000000 log/access | awk -F" " '($9 ~! 500){print $4, $6, $7, $9, $10, $11}' | sort -h -r -k6 | head -n 20


## Requests with the largest response

### Requests with the largest response

	tail -n 1000000 log/access | awk -F" " '{print $4, $5, $6, $7, $10, $11}' |  sort -h -r -k5 | head -n 20

### Requests with the largest response, with a status code other than 500

	tail -n 1000000 log/access | awk -F" " '($9 ~! 500){print $4, $6, $7, $9, $10, $11}' | sort -h -r -k5 | head -n 20

## Crashes

### Latest requests to crash the server (generated a status code 500 )

	tail -n 1000000 log/access | awk -F" " '($9 == 500){print $4, $6, $7, $9, $10, $11}' | tail -n 20

Notice how ``awk`` filters its input based on the condition that the 9th field(status code) is equal to 500.	

## Totals

### Total number of requests per day
Outputs the total number of requests per day

	tail -n 1000000 log/access | awk '{print $4}' | cut -d: -f1 | uniq -c

Once a peak has been spotted on a particular day, we can "zoom in" on that day, and view the total number of requests per hour. See next script	

### Total number of requests per hour, for a certain day
Outputs the total number of requests per hour, for the 23rd of January

	tail -n 1000000 log/access | grep "23/Jan" | cut -d[ -f2 | cut -d] -f1 | awk -F: '{print $2":00"}' | sort -n | uniq -c

Once a peak has been spotted on a particular hour, we can "zoom in" on the hour, and view the total number of requests per minute. See next script.

### Total number of requests per minute, for a certain hour
Outputs the total number of requests per minute for 15:00 - 15:59, on 02/Sep/2013

	tail -n 1000000 log/access | grep "02/Sep/2013:15" | cut -d[ -f2 | cut -d] -f1 | awk -F: '{print $2":"$3}' | sort -nk1 -nk2 | uniq -c | awk '{ if ($1 > 10) print $0}'

Once a peak has been spotted on a particular minute, we can "zoom in" on that minute, and analyze those requests more closely. See next script.

### All requests, on a particular minute
Outputs all requests that took place on 15:52 of 02/Sep/2013

	tail -n 1000000 log/access | grep "02/Sep/2013:15:52"

## Sources
* [View level of traffic with Apache log](http://www.inmotionhosting.com/support/website/server-usage/view-level-of-traffic-with-apache-access-log)
* [One liners for Apache log files](http://blog.nexcess.net/2011/01/21/one-liners-for-apache-log-files/)
* [Analyzing Apache log files](http://www.the-art-of-web.com/system/logs/#.UidkbWT0-p0)

