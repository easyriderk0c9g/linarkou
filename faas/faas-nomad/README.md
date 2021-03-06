
[![Build Status](https://travis-ci.org/hashicorp/faas-nomad.svg)](https://travis-ci.org/hashicorp/faas-nomad)
[![Docker Repository on Quay](https://quay.io/repository/nicholasjackson/faas-nomad/status "Docker Repository on Quay")](https://quay.io/repository/nicholasjackson/faas-nomad)
[![OpenFaaS](https://img.shields.io/badge/openfaas-serverless-blue.svg)](https://www.openfaas.com)
[![Maintainability](https://api.codeclimate.com/v1/badges/c0c0865f3f5f1b9ba06c/maintainability)](https://codeclimate.com/github/hashicorp/faas-nomad/maintainability)
[![Test Coverage](https://api.codeclimate.com/v1/badges/c0c0865f3f5f1b9ba06c/test_coverage)](https://codeclimate.com/github/hashicorp/faas-nomad/test_coverage)

# OpenFaaS - Nomad Provider
This repository contains the OpenFaaS provider for the Nomad scheduler.  OpenFaaS allows you to run your private functions as a service.  Functions are packaged in Docker Containers which enables you to work in any language and also interact with any software which can also be installed in the container.

## OpenFaaS Architecture
For the simplest installation, only two containers need to run on the Nomad cluster:
1. OpenFaaS Gateway
2. OpenFaaS Nomad provider
However for OpenFaaS to automatically scale your function based on inbound requests and to be able to gather metrics from the Nomad provider two additional containers are optionally run.
3. Prometheus DB
4. StatsD server for Prometheus
5. Grafana for querying Prometheus data

![](images/openfaas_nomad.png)


### OpenFaaS Gateway
The gateway provides a common API which is used by the command line of for deploying functions.  In addition to this it also hosts a Prometheus metrics endpoint which is used to provide operational metrics.  The gateway does not itself interact with the Nomad cluster instead it delegates all requests to the Nomad provider.  

### Nomad provider
The Nomad provider is responsible for performing any actions on the Nomad server such as deploying new functions or scaling functions.  Also it also acts as a function proxy.  Metrics such as execution duration and other information are emitted by the proxy and captured by the StatsD server.  Prometheus regularly collects this information from the StatsD server and stores it as time series data.

### OpenFaaS Functions
The functions performing the work on OpenFaaS are packaged as Docker images.  When running on the cluster these functions do not provide any external interface; instead, interactions are performed through the Nomad provider.  When a function is deployed, it is registered with Consul's service catalog.  The provider uses this service catalog for service discovery to be able to locate and call the downstream function.

## Starting a local Nomad / Consul environment
First, ensure that you have a recent version of Nomad and Consul installed, the latest versions can be found at:  
[Consul Versions | HashiCorp Releases](https://releases.hashicorp.com/consul/)  
[Nomad Versions | HashiCorp Releases](https://releases.hashicorp.com/nomad/)

Make sure you download the correct architecture for your machine, binaries are available for most platforms, Mac, Windows, Linux, Arm, etc.

To get things up and running quickly you can run the bash script located in the root of this repository.

```bash
$ source ./startNomad.sh                                                                                                                                              
Discovered IP Address: 192.168.1.113                                                                                                                           
Starting Consul, redirecting logs to /Users/nicj/log/consul.log                                                                                                
Starting Nomad, redirecting logs to /Users/nicj/log/nomad.log                                                                                                  
NOMAD Running 
```

The startup script will set the advertised address to your primary local IP address and run both Nomad and Consul in the background redirecting the logs to your home folder.

## Using Vagrant for Local Development
Vagrant is a tool for provisioning dev environments. The `Vagrantfile` governs the Vagrant configuration:
1) Install Vagrant via [download links](https://www.vagrantup.com/downloads.html) or package manager
2) Install VirtualBox via [download links](https://www.virtualbox.org/wiki/Downloads) or preferred hypervisor of your choice (vagrant plugins may be required). VMWare Fusion is supported.
3) `vagrant up` (default VirtualBox) or `vagrant up --provider vmware_fusion`

The provisioners install Docker, Nomad, Consul, and Vault (via Saltstack) then launch OpenFaaS components with Nomad. If successful, the following services will be available over the private network (192.168.50.2):
- Nomad (v0.8.4) 192.168.50.2:4646
- Consul (v1.2.0) 192.168.50.2:8500
- Vault (v0.9.6) 192.168.50.2:8200
- FaaS Gateway (0.9.14) 192.168.50.2:8080

This setup is intended to streamline local development of the faas-nomad provider with a more complete setup of the hashicorp ecosystem. Therefore, it is assumed that the faas-nomad source code is located on your workstation, and or is configured to listen on 0.0.0.0:8080 when debugging/running the Go process.

## Starting a remote Nomad / Consul environment
If you would like to test OpenFaaS running on a cluster in AWS, a Terraform module and instructions can be found here:
[faas-nomad/terraform at master ?? hashicorp/faas-nomad ?? GitHub](https://github.com/hashicorp/faas-nomad/tree/master/terraform)

Regardless of which method you use interacting with OpenFaaS is the same.

## Running the OpenFaaS application
First, we need to start the OpenFaaS application, to do this there are two job files located in the folder `nomad_job_files` which set things up using sensible defaults.
To run the main application execute the following command:

```bash
$ nomad run ./nomad_job_files/faas.hcl

==> Monitoring evaluation "a3e54faa"
    Evaluation triggered by job "faas-nomadd"
    Allocation "28f60a54" created: node "867c6baa", group "faas-nomadd"
    Allocation "7223b65d" created: node "d196a533", group "faas-nomadd"
    Allocation "a4dbae6c" created: node "123e18c0", group "faas-nomadd"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "a3e54faa" finished with status "complete"
```

This job will start an instance of the OpenFaaS gateway and the Nomad provider on every node in the cluster.

We can then launch the monitoring job to start Prometheus and Grafana:

```bash
$ nomad run ./nomad_job_files/monitoring.hcl

==> Monitoring evaluation "7d9c46df"
    Evaluation triggered by job "faas-monitoring"
    Allocation "e20ace08" created: node "123e18c0", group "faas-monitoring"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "7d9c46df" finished with status "complete"
```

This job starts a single instance of Prometheus and Grafana on the Nomad cluster.

## Setting up Grafana to view application metrics
If you are not using the provided Terraform module, you will need to locate the node which Grafana is running on, assuming you have not changed the job file the port will be `3000`.
To log into Grafana use the default username and password `admin.`

![](images/grafana_login.png)

Once you have successfully logged in, the next step is to create a data source for the Prometheus server.

![](images/grafana_add_datasource.png)

Configure the options as shown ensuring that the URL points to the location of your Prometheus server.  The next step is to add a dashboard to view the data from the OpenFaaS gateway and provider.  A simple dash can be found at  `grafana\faas-dashboard.json`, let's add this to Grafana.  Clicking the `Import` button from the `Dashboards` menu will pop up a box like the one below.  Choose the file for the example dashboard and press import.

![](images/grafana_dashboard.png)

Assuming all went well, you should now see the dashboard in Grafana:

![](images/grafana_dashboard_details.png)

## Creating and deploying a function
To create functions, we can install the `faas-cli` tool, to get the CLI tool you can run the following command:

```bash
$ curl -sL https://cli.openfaas.com | sudo sh
```

Alternately if you are using a Mac, the cli is also available via `brew install faas-cli`

### Creating a new function
Changing to a new folder we can create a new function by running the following command in the CLI:

```bash
$ faas-cli new gofunction -lang go
#...
2017/11/17 11:35:49 Cleaning up zip file...

Folder: gofunction created.
  ___                   _____           ____
 / _ \ _ __   ___ _ __ |  ___|_ _  __ _/ ___|
| | | | '_ \ / _ \ '_ \| |_ / _` |/ _` \___ \
| |_| | |_) |  __/ | | |  _| (_| | (_| |___) |
 \___/| .__/ \___|_| |_|_|  \__,_|\__,_|____/
      |_|


Function created in folder: gofunction
Stack file written: gofunction.yml
``` 

The command will create two folders and one file in the current directory:

```bash
$ tree -L 1  
.
????????? gofunction
????????? gofunction.yml
????????? template

2 directories, 1 file
```

The `gofunction` folder is where the source code for your application will live by default there is the main entry point called `handler.go`:

```go
package function

import (
    "fmt"
)

// Handle a serverless request
func Handle(req []byte) string {
    return fmt.Sprintf("Hello, Go. You said: %s", string(req))
}
```

The Handle method receives the payload sent by calling the function as a slice of bytes and expects any output to be returned as a string.  For now, let's keep this function the same and run through the steps for building the function.  The first thing we need to do is to edit the `gofunction.yml.` file and change the image name so that we can push this to a Docker repo that our Nomad cluster will be able to pull.  Also, change the gateway address to the location of your OpenFaaS gateway.  Changing the gateway in this file saves us providing the location as an alternate parameter.

```yaml
provider:
  name: faas
  gateway: http://localhost:8080

functions:
  gofunction:
    lang: go
    handler: ./gofunction
    image: nicholasjackson/gofunction
```

### Building our new function
Next step is to build the function; we can do this with the `faas-cli build` command:

```bash
$ faas-cli build -yaml gofunction.yml 
#...
Step 16/17 : ENV fprocess "./handler"
 ---> Using cache
 ---> 5e39e4e30c60
Step 17/17 : CMD ./fwatchdog
 ---> Using cache
 ---> 2ae72de493b7
Successfully built 2ae72de493b7
Successfully tagged nicholasjackson/gofunction:latest
Image: gofunction built.
[0] < Builder done.
```

The `build` command execute the Docker build command with the correct Dockerfile for your language.  All code is compiled inside of the container as a multi-stage build before being packaged into an Image.

### Pushing the function to the Docker repository
We can either use the `faas-cli push` command to push this to a Docker repo, or we can manually push.

```bash
$ faas-cli push -yaml gofunction.yml 
[0] > Pushing: gofunction.
The push refers to a repository [docker.io/nicholasjackson/gofunction]
cc9df684d32a: Pushed 
4e12ae9c1d69: Pushed 
cdcffb5144dd: Pushed 
10d64a26ddb0: Pushed 
dbbae7ea208f: Pushed 
2aebd096e0e2: Pushed 
latest: digest: sha256:57c0143772a1e6f585de019022203b8a9108c2df02ff54d610b7252ec4681886 size: 1574
[0] < Pushing done.
```

### Deploying the function
To deploy the function we can again use the `faas-cli` tool to deploy the function to our Nomad cluster:

```bash
$ faas-cli deploy -yaml gofunction.yml
Deploying: gofunction.
Removing old function.
Deployed.
URL: http://192.168.1.113:8080/function/gofunction

200 OK
```

If you run the `nomad status` command, you will now see the additional job running on your Nomad cluster.

```bash
$ nomad status
ID                   Type     Priority  Status   Submit Date
OpenFaaS-gofunction  service  1         running  11/17/17 11:52:59 GMT
faas-monitoring      service  50        running  11/15/17 14:43:11 GMT
faas-nomadd          system   50        running  11/15/17 11:00:31 GMT
```

### Running the function
To run the function, we can simply curl the OpenFaaS gateway and pass our payload as a string:
```bash
$ curl http://192.168.1.113:8080/function/gofunction -d 'Nic'
Hello, Go. You said: Nic
```

or you can use the cli

```bash
$ echo "Nic" | faas-cli --gateway http://192.168.1.113:8080/ invoke gofunction
```

That is all there is to it, checkout the OpenFaaS community page for some inspiration and other demos.
[faas/community.md at master ?? openfaas/faas ?? GitHub](https://github.com/openfaas/faas/blob/master/community.md)

### Datacenters and Constraints
By default, the Nomad provider will use the datacenter of the Nomad agent or `dc1`. This can be overridden by setting one or more constraints `datacenter == value`.  Constraints for limiting CPU and memory can also be set `memory` is an integer representing Megabytes, `cpu` is an integer representing MHz of CPU where 1024 equals one core.

i.e.
```bash
$ faas-cli deploy --constraint 'datacenter == dc1' --constraint 'datacenter == dc2'
```

or from a stack file...
```yaml
functions:
  facedetect:
    lang: go-opencv
    handler: ./facedetect
    image: nicholasjackson/func_facedetect
    limits:
      memory: 512
      cpu: 1000
    constraints:
      "datacenter == test1"
```

### Annotations
Metadata can be added to the Nomad job definition through the use of the OpenFaaS annotation config.  The below example would add the key `git` to the `Meta` section of nomad job definition which can be accessed through the API.

```yaml
functions:
  facedetect:
    lang: go-opencv
    handler: ./facedetect
    image: nicholasjackson/func_facedetect
    annotations:
      git: https://github.com/alexellis/super-pancake-fn.git
```

### Secrets API
It is possible to integrate Vault secrets [https://docs.openfaas.com/reference/secrets/](https://docs.openfaas.com/reference/secrets/) with the Nomad provider. Follow these steps to have OpenFaaS integrate with Nomad + Vault:

1) First, we need to enable the approle auth backend in Vault:

   ```vault auth enable approle```

2) We also need to create a policy for faas-nomad and OpenFaaS functions:

   ```vault policy write openfaas policy.hcl```

   Policy file example: https://raw.githubusercontent.com/hashicorp/faas-nomad/master/provisioning/scripts/policy.hcl

   It is important that the policy contain: create, update, delete and list capabilities that match your secret backend prefix. In this case, path `secret/openfaas/*` will work with the default configuration.

   Also, faas-nomad takes care of renewing it's own auth token, so we need to make sure the policy uses path "auth/token/renew-self" and has the "update" capability.


3) Finally, let's setup the approle itself:
    ```bash
    curl -i \
      --header "X-Vault-Token: ${VAULT_TOKEN}" \
      --request POST \
      --data '{"policies": ["openfaas"], "period": "24h"}' \
      https://${VAULT_HOST}/v1/auth/approle/role/openfaas
    ```

    This creates the role attached to the policy we just created. The "period" property and duration is important for renewing long-running service Vault tokens.

    ```bash
    curl -i \
      --header "X-Vault-Token: ${VAULT_TOKEN}" \
      https://${VAULT_HOST}/v1/auth/approle/role/openfaas/role-id
    ```

    Produces the role_id needed for -vault_app_role_id cli argument.

    ```bash
    curl -i \
      --header "X-Vault-Token: ${VAULT_TOKEN}" \
      --request POST \
      https://VAULT_HOST}/v1/auth/approle/role/openfaas/secret-id
    ```

    Produces the secret_id needed for -vault_app_secret_id cli argument.

Let's assume the Vault parameters have been populated, and you're now running faas-nomad along with the other OpenFaaS components. Now, try out the new faas-cli secret commands:

```bash
faas-cli secret create grafana_api_token --from-literal=foo --gateway ${FAAS_GATEWAY}
```

Now we can use our newly created secret "grafana_api_token" in a new function we want to deploy:

```bash
faas-cli deploy --image acornies/grafana-annotate --secret grafana_api_token --env grafana_url=http://grafana.service.consul:3000
```

### Async functions
OpenFaaS has the capability to immediately return when you call a function and add the work to a nats streaming queue.  To enable this feature in addition to the OpenFaaS gateway and Nomad provider you must run a nats streaming server.  
To run the server please use the `nats.hcl` job file.

```bash
$ nomad run ./nomad_job_files/nats.hcl
```

You can then invoke a function using the `async-function` API, the call will be immediately retuned and OpenFaaS will queue your work for later execution.

```
curl -d '{...}' http://gateway:8080/async-function/{function_name}
```

### Configuration and Function timeouts
By Default a function is allowed to run for 30s before it is terminated, should you require longer running functions timeout is configurable by setting the flag `-function_timeout` on the Nomad provider e.g:

```hcl
 args = [
   "-nomad_region", "${NOMAD_REGION}",
   "-nomad_addr", "${NOMAD_IP_http}:4646",
   "-consul_addr", "${NOMAD_IP_http}:8500",
   "-statsd_addr", "${NOMAD_ADDR_statsd_statsd}",
   "-node_addr", "${NOMAD_IP_http}",
   "-logger_format", "json",
   "-logger_output", "/logs/nomadd.log"
   "-function_timeout", "5m"
]
```

This would set the timeout to 5m for a function.

### Contributing
The application including docker containers is built using goreleaser [https://goreleaser.com](https://goreleaser.com).  

#### Setup
* Clone this repo: `go get github.com/hashicorp/faas-nomad`
* Create a fork in your own github account
* Add a new git remote to $GOPATH/src/hashicorp/faas-nomad with your fork `git remote add fork git@github.com:/yourname/faas-nomad.git`

#### Building the application
`make build_all` this runs the command `goreleaser -snapshot -rm-dist -skip-validate`

#### Testing the application
`make test` runs all unit tests in the application, for continuous test running try [http://goconvey.co](http://goconvey.co)
