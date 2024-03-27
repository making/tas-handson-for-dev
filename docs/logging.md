You can check the application log with the `cf logs` command. You can check the latest log by adding the `--recent` option.

```bash
cf logs hello-cf --recent
```

If no option is given, the log will be traced.


```
cf logs hello-cf
```

Later we will also check the Apps Manager and App Metrics logs.


> Note: If you want to transfer the log for each application to an external Syslog or HTTP Server within the responsibility of the application engineer, you can use the function called [Syslog Drain](https://docs.vmware.com/en/VMware-Tanzu-Application-Service/5.0/tas-for-vms/services-log-management.html).
> <br>
> You can also use the Platform function called [Aggregate Syslog Drain](https://docs.vmware.com/en/VMware-Tanzu-Application-Service/5.0/tas-for-vms/config-sys-logs.html) to transfer all application logs to Syslog Server.