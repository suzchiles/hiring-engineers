# The Exercise

## Setting up the environment

### The stack

I have a personal AWS Lightsail account, which had an unused slot for a new environment. I created a LAMP stack with Ubuntu 18.04.1 LTS:
![aws-lamp-1](https://user-images.githubusercontent.com/7598087/64481396-f6a44500-d18f-11e9-9f3e-45b3d02adef5.jpg)

I also installed and configured Apache2:

```
sudo apt update
sudo apt install apache2
```

Verified the successful installation of Apache2 at http://34.215.146.115/

### Datadog 

Signing up for a Datadog account was quick and easy. I thought it was a great experience for the user. I easily installed an Ubuntu agent from [https://app.datadoghq.com/account/settings#agent/ubuntu](url)

## Collecting metrics

### Adding tags to datadog configuration file.

I updated the datadog basic configuration file and added two tags: `/etc/datadog-agent/datadog.yaml`

```
#########################
## Basic Configuration ##
#########################

## @param api_key - string - required
## The Datadog API key to associate your Agent's data with your organization.
## Create a new API key here: https://app.datadoghq.com/account/settings
#
api_key: 2b2d1d1dd155341f8544dfade7ce1753

## @param site - string - optional - default: datadoghq.com
## The site of the Datadog intake to send Agent data to.
## Set to 'datadoghq.eu' to send data to the EU site.

site: datadoghq.com
...
## @param hostname - string - optional - default: auto-detected
## Force the hostname name.
#
hostname: Suzanne-Host
...
## @param tags  - list of key:value elements - optional
## List of host tags. Attached in-app to every metric, event, log, trace, and service check emitted by this Agent.
##
## Learn more about tagging: https://docs.datadoghq.com/tagging/
#
tags:
- environment:sandbox
- availability-zone:portland-oregon
...
```

This is the host map after this update to datadog.yaml:

![host-tags](https://user-images.githubusercontent.com/7598087/64481562-a8dd0c00-d192-11e9-8bd2-7bd71498e41b.jpg)

### Installing and configuring MySQL

Installed MySQL on the LAMP stack,

```
sudo apt update
sudo apt install mysql-server
sudo mysql_secre_installation
```

When prompted, I created a password for root.

After installing MySQL, I followed the (excellent) instructions to configure MySQL at ```/etc/datadog-agent/conf.d/mysql.c/conf.yaml``` with:

```
init_config:
instances:
	- server: 127.0.0.1
	user: datadog
	pass: dd123456
	port: 3306                     
```

Note the config file sets up a user(datadog) and password (dd123456), which is required by the next step. 

### Create a custom agent check

Followed the instructions at `https://docs.datadoghq.com/developers/write_agent_check/?tab=agentv6` to create the custom check for `my_metric`.

#### Write a Python script

I wrote this Python script, importing the Python random function.

```
from datadog_checks.checks import AgentCheck

import random

class mymetric(AgentCheck):
    def check(self, instance):
        x=random.randint(1,1000)
        self.gauge ('My_Metric', x,tags=['TAG_KEY:TAG_VALUE'])
        
```

I then saved the file as `/etc/datadog-agent/checks.d/custom_test.py`

#### Write the agent check configuration file

I then wrote the configuration file for the custom check.

```
init-config:

instances:
- url: http://34.215.146.115/
- min_collection_interval: 45

```

I saved the file to `/etc/datadog-agent/conf.d/custom_test.yaml`.

#### Write the configuration file for the check

The useful Datadog check was helpful for checking and debugging both the Python script and the .yaml file:

```
sudo -u dd-agent -- datadog-agent check custom-agent
```

#### Review Datadog configuration file 

I next reviewed `datadog.yaml` configuration file to ensure that the agent was configured to enable this check. I set up DogStatsD to enable the check.

```
############################
## DogStatsD Configuration ##
############################

## @param use_dogstatsd - boolean - optional - default: true Set this option to 
## false to disable the Agent DogStatsD server.
#
use_dogstatsd: true

## @param dogstatsd_port - integer - optional - default: 8125 Override the Agent 
## DogStatsD port. Note: Make sure your client is sending to the same UDP port.
#
dogstatsd_port: 8125
## @param bind_host - string - optional - default: localhost The host to listen on 
## for Dogstatsd and traces. This is ignored by APM when 
## `apm_config.non_local_traffic` is enabled and ignored by DogStatsD when 
## `dogstatsd_non_local_traffic` is enabled. The trace-agent uses this host to send 
## metrics to. The `localhost` default value is invalid in IPv6 environments where 
## dogstatsd listens on "::1". To solve this problem, ensure Dogstatsd is listening 
## on IPv4 by setting this value to "127.0.0.1".
#
bind_host: localhost
## @param dogstatsd_buffer_size - integer - optional - default: 8192 The buffer 
## size use to receive statsd packets, in bytes.
#
dogstatsd_buffer_size: 8192

## @param dogstatsd_stats_buffer - integer - optional - default: 10 Set how many 
## items should be in the DogStatsD's stats circular buffer.
#
dogstatsd_stats_buffer: 10

## @param dogstatsd_stats_port - integer - optional - default: 5000 The port for 
## the go_expvar server.
#
dogstatsd_stats_port: 5000

## @param dogstatsd_metrics_stats_enable - boolean - optional - default: false Set 
## this parameter to true to have DogStatsD collects basic statistics (count/last 
## seen) about the metrics it processsed. Use the Agent command "dogstatsd-stats" 
## to visualize those statistics.
#
dogstatsd_metrics_stats_enable: false
```

## Visualizing Data

The dashboard I created can be viewed at https://app.datadoghq.com/dashboard/u7q-cym-3dy/suzannes-scripted-timeboard?from_ts=1567912507836&to_ts=1567913407836&live=true&tile_size=m

![](/Users/suzannechiles/Desktop/datadog test/scripted-timeboard.jpg)

I experimented with several types of widgets, so you’ll see some extra widgets.

## Create a timecard

The documentation for the Datadog API gave me all the help I needed to create a .json file to create a scripted dashboard:

```
"title": "Suzanne's Scripted Timeboard",
	"description": "It's scripted!",
	"widgets": [
		{
			"definition": {
				"type": "timeseries",
				"requests": [
					{
						"q": "avg:My_Metric{*}",
						"display_type": "line",
						"style": {
							"palette": "dog-purple",
							"line_type": "solid",
							"line_width": "normal"
						}
					}
				],
				"title": "Timeline for My_Metric"
			}
		},
		{
			"definition": {
				"type": "change",
				"requests": [
					{
						"q": "avg:My_Metric{host:Suzanne-Host} by {availability-zone}",
						"change_type": "absolute",
						"compare_to": "hour_before",
						"increase_good": true,
						"order_by": "change",
						"order_dir": "desc"
					}
				],
				"title": "Change in the average of values for My_Metric"
			}
		},
		{
			"definition": {
				"type": "query_value",
				"requests": [
					{
						"q": "avg:My_Metric{*}",
						"aggregator": "avg"
					}
				],
				"title": "Average of My_Metric values",
				"autoscale": true,
				"precision": 0
			}
		}
	],
	"template_variables": [
		{
			"name": "var",
			"default": "*",
			"prefix": "check_name"
		}
	],
	"layout_type": "ordered",
	"is_read_only": true
}
```

I clicked the gear in the top right corner and selected Import dashboard json. I then copied the contents of the json file and pasted it it the Import dashboard tile. 

### Create an anomaly monitor

I found that creating a monitor was quite easy using the built-in tool at `https://app.datadoghq.com/monitors#/create`. 

I chose to create an anomaly monitor for the `datadog.trace_agent.cpu_percent` attribute. Here’s the page at https://app.datadoghq.com/monitors#11460910/edit:

![anomaly-monitor](/Users/suzannechiles/Desktop/datadog test/anomaly-monitor.jpg)

I then used the Export Monitor button to see the json content for the monitor.

```
{
	"name": "Agent CPU percentage anomaly monitor",
	"type": "query alert",
	"query": "avg(last_1d):anomalies(avg:datadog.trace_agent.cpu_percent{*}, 'basic', 2, direction='above', alert_window='last_1h', interval=300, count_default_zero='true') >= 1",
	"message": "Keep an eye on the percentage of CPU resources used by the agent. @suzanne.chiles@me.com",
	"tags": [],
	"options": {
		"notify_audit": true,
		"locked": false,
		"timeout_h": 0,
		"silenced": {},
		"include_tags": true,
		"no_data_timeframe": null,
		"require_full_window": false,
		"new_host_delay": 300,
		"notify_no_data": false,
		"renotify_interval": 0,
		"escalation_message": "",
		"threshold_windows": {
			"recovery_window": "last_1h",
			"trigger_window": "last_1h"
		},
		"thresholds": {
			"critical": 1,
			"critical_recovery": 0
		}
	}
}
```

### Add the anomaly monitor to the timeboard

I used the same technique to add the monitor to the timeboard as I did for the timeseries dashboard (Gear > Import dashboard json > import).

### Adjust the timeboard’s time frame 

This one was a bit of a puzzle, but then I remembered seeing it on the Keyboard shortcuts popup. Using the **[** and **]** keys, I was able to adjust the time range to show a five minute time range.

![dash-monitor-anomaly-5sec](/Users/suzannechiles/Desktop/datadog test/dash-monitor-anomaly-5sec.jpg)


## Conclusion

I really enjoyed this test, in spite of a few frustrations. I learned more about Datadog this week than I did in several months at New Relic. Thank you for such a great introduction to the Datadog.

# Retrieving real-time Google Analytics metrics in Datadog

## Overview

In this blog post, we’ll explain how you can retrieve Google Analytics data from the Real Time API and send it as a regular metric to Datadog. This integration of Google Analytics:

- Supports multiple profiles (views)
- Handles active users (`rt:activeUsers`) and Pageviews (`rt:pageviews`) metrics
- Allows for custom tagging for each profile
- Allows for dimension-tag mapping for Pageviews metric

## Setup

Files and other resources for this integration are found at https://github.com/bithauschile/datadog-ga.

### Installation

The installation and configuration of this integration is broken down into three steps. First, you’ll get a service account and API key. Next you’ll authorize the service account to query Google Analytics. Finally, you’ll install and configure the check to run automatically.

#### Step 1: Get service account and API key

1. Go to https://console.developers.google.com.
2. Use the My Project dropdown to select an existing project or to create a new project.
3. Open your project.
4. Enable Google Analytics API from the API Library.
5. From the Google Analytics API sidebar, select Credentials.
6. Click Create Credentials > Service account key.
7. Select "New service account" and enter a name.
8. Select json for key file.
9. Download the key information in the JSON format.
10. Note the  service account email (ie: my-svc-account@my-project.12345.iam.gserviceaccount.com).

#### Step 2: Authorize service account to query analytics

1. Log in at  https://analytics.google.com.
2. Select the Admin option from the sidebar.
3. Select the account you want to integrate. You may also select the specific property you want to give access.
4. Select the User Management option.
5. Add Read & Analyze permissions for your service account email address.
6. Select the new service account and choose View user’s account details from the stacked menu.
7. Select the property and view you want to collect metrics from.
8. Note the View ID--you’ll need it later.

#### Step 3: Install the check

The procedure in this step are based on Ubuntu Linux.

1. Clone or download https://github.com/bithauschile/datadog-ga.
2. Install the Python libraries.
3. Use `pip` to install the Google API client for Python: `/opt/datadog-agent/embedded/bin/pip install --upgrade google-api-python-client`.
4. Copy `ga.yaml` to `/etc/datadog-agent/conf.d/`.
5. Copy `ga.py` to `/etc/datadog-agent/checks.d/‘.
6. Copy the api key json file to the server in a directory the agent can access, such as `/etc/datadog-agent/conf.d/`.
7. Configure your check by adding the account information  and properties you want to integrate in `/etc/datadog-agent/conf.d/ga.yaml`. In this example, the query divides the pageviews into three dimensions: country, city, device.  

```
init_config:
    min_collection_interval: 55
    service_account_email: my-svc-account@my-project.12345.iam.gserviceaccount.com
    key_file_location: /etc/datadog-agent/conf.d/key.json

  instances:

    - profile: ga:123456789
      tags:
       - env:test
      pageview_dimensions:
       - rt:country
       - rt:city
       - rt:deviceCategory 
```

Note: Google Analytics generates a by-minute result in the Real Time response without a timestamp. To correlate Analytics data with other metrics, set `min_collection_interval` value to approximately 60 seconds.

7. You can check your configuration of this integration with the built-in Datadog check: `sudo -u dd-agent -- datadog-agent check ga`.
8. Restart your agent with `sudo service datadog-agent start`. See [Agent Commands](https://docs.datadoghq.com/agent/guide/agent-commands/?tab=agentv6#start-the-agentt) for information about starting, stopping, and restarting your Agent for all supported platforms.

## Data Collected

Any of these dimensions and metrics may be used in your yaml configuration file as an instance. You can define multiple instances in your yaml configuration.

### User

#### Dimensions

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 	 <col width="60%"> 
			<tbody>
				<tr>
					<td>
						<strong>rt:userType</strong>
					</td>
					<td>
						The average number of instances that are available for streaming or are currently streaming
					</td>
				</tr>
			</tbody>
</table>

#### Metrics

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
			<tbody>
				<tr>
					<td>
						<strong>rt:activeUsers</strong>
					</td>
					<td>
						The number of users interacting with the property right now.
					</td>
				</tr>
			</tbody>
</table>

### Time

#### Dimensions

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
	<tbody>
		<tr>
			<td>
				<strong>rt:minutesAgo</strong>
			</td>
			<td>
				The number of minutes ago a hit occurred.
			</td>
		</tr>
	</tbody>
</table>

### Traffic sources

#### Dimensions

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
<tbody>
	<tr>
		<td>
			<strong>rt:referralPath</strong>
			</td>
			<td>
				The path of the referring URL (e.g. document.referrer). If someone places a link to your property on their website, this element contains the path of the page that contains the referring link. This value is only set when `rt:medium=referral`.
			</td>
		</tr>
	</tbody>
	<tr>
		<td>
			<strong> rt:campaign</strong>
			</td>
			<td>
				For manual campaign tracking, it is the value of the utm_campaign campaign tracking parameter. Otherwise, its value is `(not set)`.  
			</td>
		</tr>
		<tr>
			<td>
				<strong>rt:source</strong>
			</td>
			<td>
				The source of referrals to your property. When using manual campaign tracking, the value of the `utm_source` campaign tracking parameter. When using Google Ads autotagging, the value is `google`. Otherwise the domain of the source referring the user to your property (for example, `document.referrer`). The value may also contain a port address. If the user arrived without a referrer, the value is `(direct)`.
			</td>	
		</tr>
		<tr>
			<td>
				<strong>rt:medium</strong>
			</td>
			<td>
				The type of referrals to your property. When using manual campaign tracking, the value of the `utm_medium` campaign tracking parameter. When using Google Ads autotagging, the value is `ppc`. If the user comes from a search engine detected by Google Analytics, the value is `organic`. If the referrer is not a search engine, the value is `referral`. If the user came directly to the property, and `document.referrer` is empty, the value is `(direct)`. 
			</td>
		<tr>
		<td>
			<strong>rt:trafficType</strong>
			</td>
			<td>
				This dimension is similar to `rt:mediu`m for constant values such as `organic`, `referral`, `direct`, etc. It is different for custom referral types. As an example, if you add the `utm_campaign` parameter to your URL with value <strong>email</strong>, `rt:medium` will be <strong>email</strong> but rt:trafficType will be <strong>custom</strong>. 
			</td>
		</tr>
<tr>
		<td>
			<strong>rt:keyword</strong>
			</td>
			<td>
				For manual campaign tracking, it is the value of the `utm_term` campaign tracking parameter. Otherwise, its value is `(not set)`.
			</td>
		</tr>
</table>

### Goal conversions

#### Dimensions

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
<tbody>
	<tr>
		<td>
			<strong>rt:goalId</strong>
			</td>
			<td>
				A string. Corresponds to the goal ID. 
			</td>
		</tr>
	</tbody>
</table>

#### Metrics

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
<tbody>
	<tr>
		<td>
			<strong>rt:goalXXValue</strong>
			</td>
			<td>
				The total numeric value for the requested goal number, where XX is a number between 1 and 20. 
			</td>
		</tr>
  <td>
			<strong>rt:goalValueAll</strong>
			</td>
			<td>
				The total numeric value for all goals defined for your view (profile).  
			</td>
		</tr>
<td>
			<strong>rt:goalXXCompletions</strong>
			</td>
			<td>
				The total number of completions for the requested goal number, where XX is a number between 1 and 20. 
			</td>
		</tr>
<td>
			<strong>rt:goalCompletionsAll</strong>
			</td>
			<td>
				The total number of completions for all goals defined for your view (profile). 
			</td>
		</tr>
	</tbody>
</table>

### Platform / Device 

#### Dimensions

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
<tbody>
	<tr>
		<td>
			<strong>rt:browser</strong>
			</td>
			<td>
				The names of browsers used by users to your property.
			</td>
		</tr>
	<tr>
		<td>
			<strong> rt:browserVersion</strong>
		</td>
		<td>
			The browser versions used by users to your property. 
		</td>
	</tr>
	<tr>
		<td>
			<strong>rt:operatingSystem</strong>
		</td>
		<td>
			The operating system used by users to your property. 
		</td>
	</tr>
	<tr>
		<td>
			<strong>rt:operatingSystemVersion</strong>
		</td>
		<td>
			The version of the operating system used by users to your property. 
		</td>
	</tr>
	<tr>
		<td>
			<strong>rt:deviceCategory</strong>
		</td>
		<td>
			The type of device: Desktop, Tablet, or Mobile. 
		</td>
	</tr>
	<tr>
		<td>
			<strong>rt:mobileDeviceBranding</strong>
		</td>
		<td>
			Mobile manufacturer or branded name (for example: Samsung, HTC, Verizon, T-Mobile) 
		</td>
	</tr>
	<tr>
		<td>
			<strong>rt:mobileDeviceMode</strong>
		</td>
		<td>
			Mobile device model (for example: Nexus S) 
		</td>
	</tr>
</tbody>
</table>

### Geo

#### Dimensions

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
<tbody>
	<tr>
		<td>
			<strong>rt:country</strong>
			</td>
			<td>
				he countries of website users, derived from IP addresses. 
			</td>
		</tr>
	<tr>
		<td>
			<strong>rt:region</strong>
			</td>
			<td>
				The region of users to your property, derived from IP addresses. In the U.S., a region is a state, such as `New York`. 
			</td>
		</tr>
	<tr>
		<td>
			<strong>rt:city</strong>
			</td>
			<td>
				The cities of users, derived from IP addresses. 
			</td>
		</tr>
	<tr>
		<td>
			<strong>rt:latitude</strong>
			</td>
			<td>
				The approximate latitude of the user's city. Derived from IP address. Locations north of the equator are represented by positive values and locations south of the equator by negative values. 
			</td>
		</tr>
	<tr>
		<td>
			<strong>rt:longitude</strong>
			</td>
			<td>
				The approximate longitude of the user's city. Derived from IP address. Locations east of the prime meridian are represented by positive values and locations west of the prime meridian by negative values. 
			</td>
		</tr>
	</tbody>
</table>

### Page Tracking

#### Dimensions

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
<tbody>
	<tr>
		<td>
			<strong>rt:pagePath</strong>
			</td>
			<td>
				A page on your property specified by path and/or query parameters. 
			</td>
		</tr>
	<tr>
		<td>
			<strong>rt:pageTitle</strong>
			</td>
			<td>
				The title of a page. Keep in mind that multiple pages might have the same page title. 
			</td>
		</tr>

```
</tbody>
```

</table>

#### Metrics

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
<tbody>
	<tr>
		<td>
			<strong>rt:pageviews</strong>
			</td>
			<td>
				The total number of page views. 
			</td>
		</tr>
	</tbody>
</table>

### App Tracking

#### Dimensions

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
<tbody>
	<tr>
		<td>
			<strong>rt:appName</strong>
			</td>
			<td>
				The name of the application. 
			</td>
		</tr>
		<tr>
			<td>
				<strong>rt:appVersion</strong>
			</td>
				<td>
					The version of the application.
			</td>
		</tr>
		<tr>
			<td>
				<strong>rt:screenName</strong>
			</td>
				<td>
					The name of a screen.		
				</td>
		</tr>
	</tbody>
</table>

#### Metrics

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
<tbody>
	<tr>
		<td>
			<strong>rt:screenViews</strong>
			</td>
			<td>
				The total number of screen views. 
			</td>
		</tr>
	</tbody>
</table>

### Event Tracking

#### Dimensions

<table class="table table-vertical-mobile table-metrics"> 
  <col width="40%">
 <col width="60%"> 
<tbody>
	<tr>
		<td>
			<strong>rt:eventAction</strong>
			</td>
			<td>
				The action of the event. 
			</td>
		</tr>
	<tr>
		<td>
			<strong>rt:eventCategory</strong>
			</td>
			<td>
				The category of the event. 
			</td>
		</tr>
	<tr>
		<td>
			<strong>rt:eventLabel</strong>
			</td>
			<td>
				The label of the event. 
			</td>
		</tr>
	</tbody>
</table>

#### Metrics

<table class="table table-vertical-mobile table-metrics">
   <col width="40%">
 <col width="60%"> 
<tbody>
	<tr>
		<td>
			<strong>rt:totalEvents</strong>
			</td>
			<td>
				The total number of events for the view (profile), across all categories. 
			</td>
		</tr>
	</tbody>
</table>

## Credits

This integration was written by [bithaus](https://developers.google.com/analytics/devguides/reporting/realtime/v3/reference/). Visit their site to learn more about this integration as well as their other initiatives.

## References

[Google Analytics Real Time API](https://developers.google.com/analytics/devguides/reporting/realtime/v3/reference/) 

[Google API Explorer—Analytics Real Time](https://developers.google.com/apis-explorer/#p/analytics/v3/analytics.data.realtime.get)

[Writing a custom Datadog Agent check](https://docs.datadoghq.com/developers/write_agent_check/?tab=agentv6/)
