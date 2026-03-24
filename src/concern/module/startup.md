# Startup

In the previous section, we have introduced the Config module of light-4j platform, and we can use values.yml to overwrite the default externalized config properties from all config files. 

We focus on the values.yml in the File System assuming the light-4j server is running stand-alone. However, we can also inject values.yml during the startup from the light-portal light-config-server. For any service built with the light-4j frameworks to leverage the light-config-server, we need to add some extra configuration files and environment variables to bootstrap the service. 

## Prerequisite

### startup.yml

Here is an example of startup.yml file for ai-gateway dev demo server.

```
# This is the config file to replace the default config loader to a customized one.
# For example, load the config files from the config server instead of filesystem.
# The following dummy entry is just to prevent the warning message during startup.
# dummy: dummyEntry
# For real config loader config, please follow the format below with your implementation.
# light-4j provides a DefaultConfigLoader that loads config files from the light-portal
# and fallback to local file systems for cached config files.
# The following is the config for the DefaultConfigLoader.
configLoaderClass: com.networknt.server.DefaultConfigLoader
# All variables below can be used to look up the config files for a particular service,
# but only the productId is required. If other variables are not provided, the default
# "current" value will be used from the config server.
host: dev.lightapi.net
serviceId: com.networknt.ai.gateway-1.0.0
envTag: DEV
# Indicate to the config server we prefer YAML format over JSON format. default is JSON.
acceptHeader: application/yaml
# The connect timeout for bootstrap from the config server. Default is 3 seconds.
timeout: 3000

```

### env variables

Before starting the service, we need to set up the following environment variables. These are used to bootstrap the service, connect to the config server, load config files, cache loaded files, and start the service with the config files.


```
CONFIG_SERVER_AUTHORIZATION=Bearer eyJraWQiOiJBWnAwMk1DdWNydVpmRkpieXlGd3VnIiwiYWxnIjoiUlMyNTYifQ.eyJpc3MiOiJ1cm46Y29tOm5ldHdvcmtudDpvYXV0aDI6djEiLCJhdWQiOiJ1cm46Y29tLm5ldHdvcmtudCIsImV4cCI6MjA4ODEwODIyNywianRpIjoiUlc4X1g5UHNmTUIxcmpIeUZ5Z0JKUSIsImlhdCI6MTc3Mjc0ODIyNywibmJmIjoxNzcyNzQ4MTA3LCJ2ZXIiOiIxLjAiLCJjaWQiOiIwMTljOTI3My0yNjYzLTdhOWUtODJmNC05NGY5ZjVmNzljM2EiLCJzY3AiOlsicG9ydGFsLnciLCJwb3J0YWwuciJdLCJyb2xlcyI6ImFHOXpkQzFoWkcxcGJpQjFjMlZ5IiwidXNlcklkIjoiMDE5NjRiMDUtNTUzMi03Yzc5LThjZGUtMTkxZGNiZDQyMWI4IiwiaG9zdCI6IjAxOTY0YjA1LTU1MmEtN2M0Yi05MTg0LTY4NTdlN2YzZGM1ZiIsImVtYWlsIjoic3RldmUuaHVAc3VubGlmZS5jb20iLCJlaWQiOiJzaDM1In0.uM7iTEHnOZKVtZk6evtar-IGhoh615er9UjQ0ozHfyLlYEBsUZK8KIk4h4R4gwgPX_ldbNBLcGT7bn2pd6JOwlfBM_VDtRZbtVjHRefaK1uYHOdRg9Ckn8xTFdYa9HCQkge7cRlgvx-HsHZn3404BeKa1YjBiJKCGRVHe7QXkKT5Mhsgma3MpQtQFnCVdswrJL1QHP2M0mjwgEH6Y1FnuBPey5IZYXkl_I4ecN2D4XM5I15sv7bvGVGLpdczJJP_YP9L-wFE_lGomst6_2FHQvF-VSry2eDtFS0o_lvW2MKrBB2roeqwZTRf5H9RiHSLVNjb4vvbQTvVfWEG3y7zGQ;CONFIG_SERVER_CLIENT_TRUSTSTORE_LOCATION=/home/steve/workspace/light-gateway/config/ai-gateway/config/bootstrap.truststore;CONFIG_SERVER_CLIENT_TRUSTSTORE_PASSWORD=password;CONFIG_SERVER_CLIENT_VERIFY_HOST_NAME=false;LIGHT_ENV=DEV
```

When using the Docker container, the following variables should be passed into the container.

```
export CONFIG_SERVER_CLIENT_TRUSTSTORE_LOCATION=/config/bootstrap.truststore
export CONFIG_SERVER_CLIENT_TRUSTSTORE_PASSWORD=password
export CONFIG_SERVER_CLIENT_VERIFY_HOST_NAME=false
export CONFIG_SERVER_AUTHORIZATION=eyJraWQiOiIxMDAiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJ1cm46Y29tOm5ldHdvcmtudDpvYXV0aDI6djEiLCJhdWQiOiJ1cm46Y29tLm5ldHdvcmtudCIsImV4c
CI6MTk0MTEyMjc3MiwianRpIjoiTkc4NWdVOFR0SEZuSThkS2JsQnBTUSIsImlhdCI6MTYyNTc2Mjc3MiwibmJmIjoxNjI1NzYyNjUyLCJ2ZXJzaW9uIjoiMS4wIiwiY2xpZW50X2lkIjoiZjdkNDIzNDgtYzY0Ny
00ZWZiLWE1MmQtNGM1Nzg3NDIxZTcyIiwic2NvcGUiOiJwb3J0YWwuciBwb3J0YWwudyIsInNlcnZpY2UiOiIwMTAwIn0.Q6BN5CGZL2fBWJk4PIlfSNXpnVyFhK6H8X4caKqxE1XAbX5UieCdXazCuwZ15wxyQJg
WCsv4efoiwO12apGVEPxIc7gpvctPrRIDo59dmTjfWH0p3ja0Zp8tYLD-5Sh65WUtJtkvPQk0uG96JJ64Da28lU4lGFZaCvkaS-Et9Wn0BxrlCE5_ta66Qc9t4iUMeAsAHIZJffOBsREFhOpC0dKSXBAyt9yuLDuD
t9j7HURXBHyxSBrv8Nj_JIXvKhAxquffwjZF7IBqb3QRr-sJV0auy-aBQ1v8dYuEyIawmIP5108LH8QdH-K8NkI1wMnNOz_wWDgixOcQqERmoQ_Q3g
export LIGHT_ENV=dev
export LIGHT_4J_CONFIG_DIR=/config
export LIGHT_CONFIG_SERVER_URI=https://localhost:8443
```
When using Kubernetes, all variables should be passed through as environment variables in the deployment. However, if you are starting the server locally, you don't want to put all of the variables to .profile as they are not easily changeable and might impact your other light-4j services developed on the same desktop. 

For me, I am putting the following to the .profile

```
export CONFIG_SERVER_CLIENT_TRUSTSTORE_LOCATION=/home/steve/networknt/light-4j/server/src/main/resources/config/bootstrap.truststore
export CONFIG_SERVER_CLIENT_TRUSTSTORE_PASSWORD=password
export CONFIG_SERVER_CLIENT_VERIFY_HOST_NAME=false
export CONFIG_SERVER_AUTHORIZATION=Bearer eyJraWQiOiIxMDAiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJ1cm46Y29tOm5ldHdvcmtudDpvYXV0aDI6djEiLCJhdWQiOiJ1cm46Y29tLm5ldHdvcmtudCIsImV4c
CI6MTk0MTEyMjc3MiwianRpIjoiTkc4NWdVOFR0SEZuSThkS2JsQnBTUSIsImlhdCI6MTYyNTc2Mjc3MiwibmJmIjoxNjI1NzYyNjUyLCJ2ZXJzaW9uIjoiMS4wIiwiY2xpZW50X2lkIjoiZjdkNDIzNDgtYzY0Ny
00ZWZiLWE1MmQtNGM1Nzg3NDIxZTcyIiwic2NvcGUiOiJwb3J0YWwuciBwb3J0YWwudyIsInNlcnZpY2UiOiIwMTAwIn0.Q6BN5CGZL2fBWJk4PIlfSNXpnVyFhK6H8X4caKqxE1XAbX5UieCdXazCuwZ15wxyQJg
WCsv4efoiwO12apGVEPxIc7gpvctPrRIDo59dmTjfWH0p3ja0Zp8tYLD-5Sh65WUtJtkvPQk0uG96JJ64Da28lU4lGFZaCvkaS-Et9Wn0BxrlCE5_ta66Qc9t4iUMeAsAHIZJffOBsREFhOpC0dKSXBAyt9yuLDuD
t9j7HURXBHyxSBrv8Nj_JIXvKhAxquffwjZF7IBqb3QRr-sJV0auy-aBQ1v8dYuEyIawmIP5108LH8QdH-K8NkI1wMnNOz_wWDgixOcQqERmoQ_Q3g
export LIGHT_ENV=dev

```


### jvm options

And the following two variable might be changed from project to project. We are going to put it into the command line with -D Java Options. 

```
-Dlight-4j-config-dir=config/ai-gateway/config -Dlight-config-server-uri=https://local.lightapi.net
```

With -Dlight-4j-config-dir we can get the externalized config folder for different service. 
With -Dlight-config-server-uri we can turn on or off config server bootstrap for the service. 

### bootstrap.truststore

In order to connect to the light-portal which is running with a self-signed certificate, you need to use the issued bootstrap.truststore in your local config folder. With the above jvm options, it should be in config/ai-gateway/config folder along with startup.yml file.

## Startup Sequence


## Fallback






