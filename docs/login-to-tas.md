Log in to the environment for Hands-on with the following command.

```bash
cf login -a https://api.sys.dhaka.cf-app.com --sso
```

When you run the command, you will be prompted to enter a passcode like this.


```
API endpoint: https://api.sys.dhaka.cf-app.com

Temporary Authentication Code ( Get one at https://login.sys.dhaka.cf-app.com/passcode ): 
```

Please log in to https://login.sys.dhaka.cf-app.com/passcode to obtain the passcode, and enter it in the terminal.

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/1c57294a-ce52-46d5-bbc3-9dd9f6147ac1">



If Org or Space is not set up, please create them with the following command. 

> Org and Space are the units of tenancy in TAS (Cloud Foundry). For more details, please refer to https://docs.vmware.com/en/VMware-Tanzu-Application-Service/5.0/tas-for-vms/roles.html. <br>
> Depending on the environment you are using, you may not have the permission to execute this command. In such cases, please request the creation of Org and Space to your administrator.

```bash
ORG_NAME=handson-${RANDOM}
SPACE_NAME=demo

cf create-org ${ORG_NAME}
cf create-space ${SPACE_NAME} -o ${ORG_NAME}
cf target -o ${ORG_NAME} -s ${SPACE_NAME}
```

You'll see the output similar to the following.

```
Creating org handson-22297 as tmaki...
OK

TIP: Use 'cf target -o "handson-22297"' to target new org
Creating space demo in org handson-22297 as tmaki...
OK

Assigning role SpaceManager to user tmaki in org handson-22297 / space demo as tmaki...
OK

Assigning role SpaceDeveloper to user tmaki in org handson-22297 / space demo as tmaki...
OK

TIP: Use 'cf target -o "handson-22297" -s "demo"' to target new space
API endpoint:   https://api.sys.dhaka.cf-app.com
API version:    3.157.0
user:           tmaki
org:            handson-22297
space:          demo
```

Check the information of the environment being accessed with the following command.

```bash
cf curl /info | jq .
```

```json
{
  "name": "Small Footprint VMware Tanzu Application Service",
  "build": "5.0.10-build.2",
  "support": "https://tanzu.vmware.com/support",
  "version": 0,
  "description": "https://docs.vmware.com/en/VMware-Tanzu-Application-Service/5.0/tas-for-vms/runtime-rn.html",
  "authorization_endpoint": "https://login.sys.dhaka.cf-app.com",
  "token_endpoint": "https://uaa.sys.dhaka.cf-app.com",
  "allow_debug": true,
  "user": "1c28899d-2767-4249-a994-3a4af63ca56e",
  "limits": {
    "memory": 2048,
    "app_uris": 4,
    "services": 16,
    "apps": 20
  },
  "usage": {
    "memory": 0,
    "apps": 0,
    "services": 0
  }
}
```

