You can use Apps Metrics to see application logs and metrics. Unlike `cf logs` command, logs are persistent and you can search past logs.

Click "View in App Metrics" in Apps Manager.

<img width="971" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/9b771856-2dc7-47b7-a3eb-2671b7815af9">

The following metrics are displayed.

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/a1ded056-f9ee-4961-8899-917bfd12bb18">

> Note: The three metrics displayed on the first line are called RED metrics and are generally the most important metrics to measure.
> 
> * **R**ate … App Metrics defaults to HTTP Request Count (per minute)
> * **E**rror … App Metrics defaults to 5XX error messages per minute (HTTP Request Errors) for the app.
> * **D**uration … App Metrics defaults to the average latency of the app (HTTP Request Latency in milliseconds). Percentile such as "95 percentile" is generally better.

Click "Logs" to display the application log.

> ⚠️ As of this writing, it seems that log search is not functioning at https://metrics.sys.dhaka.cf-app.com.


By default, only the application log is displayed. Click "App (Application)", select "ALL" or "RTR (Router)", and click "APPLY" button.

The access log is also displayed. It is good to check the access log also.
