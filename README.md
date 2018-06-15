# Setup Metrics Server on an existing Kubernetes Cluster Hosted in Rancher

## The Problem

Some of the Kubernetes defaults changed as Heapster was deprecated in favor of Metrics-Server.  Metrics-Server also requires a TLS certificate CA file that was not required by Heapster.

## Summary of the Solution

One of these 
* Modify the node definitions to make sure the cluster is metrics server compatible.
* Fix existing nodes to support metrics-server.

## More Information

This method was adapted from [this gist](https://gist.github.com/superseb/90527bd079feac17f379964f50769e28) from [Sebastiaan van Steenis @superseb](https://gist.github.com/superseb).

## Setup

### Modify Cluster Definition

**Take Note** This part of the document applies to clusters provisioned with an IaaS-Backed Rancher Node Driver (i.e. Azure, EC2, Digital Ocean).  Not provisioned via a cloud provider Rancher Node Driver (i.e. AKS, EKS, GKE) and not created via Import, or Custom workflows.

You may still need to modify your cluster. Use this document as a guide, but since Rancher did not create your cluster, your mileage may vary.

#### Create an API Key

* Navigate to API & Keys
![API Keys Button](01-APIKeys.png)

* Add a Key
![Add Key Button](02-AddKey.png)

* Name and set the expiration for your key.
![Add API Key](03-AddAPIKey.png)

* Your API Key has been created. 
![API Key](04-APIKeyCreated.png)

#### Download and Login to the Rancher CLI

* Navigate to Settings
![Settings](05-Settings.png)

* Show the CACert and copy the contents
![CA Cert](06-CACert.png)

* Paste this into your text editor of choice and save the file
![Create cert file in text editor](07-vi.png)

* Download the rancher cli via the Download CLI link.  Make sure it's available in your path.
![Download CLI](08-DownloadCLI.png)

* Login to your Rancher server using the API token and ther CA Cert you created earlier.
```sh
[jason@vblinux metrics-server-on-rancher-2.0.2 ]$ rancher login --cacert cacert.pem  -t token-82bld:bszpsrvjsgk2srlzx9tcsmk2dmntfz7zcsnzb22pghgmkv8h4cxff5 https://jvb.ddns.net/v3 
INFO[0000] Only 1 project available: Default            
INFO[0000] Saving config to /home/jason/.rancher/cli2.json 
[jason@vblinux metrics-server-on-rancher-2.0.2 ]$ rancher projects
ID                      NAME      STATE     DESCRIPTION
c-nng62:project-9crgl   Default   active    Default project created for the cluster

```
* Export the CATTLE_ACCESS_KEY and CATTLE_SECRET_KEY settings from the API token you created.

```sh
export CATTLE_ACCESS_KEY=token-82bld
export CATTLE_SECRET_KEY=bszpsrvjsgk2srlzx9tcsmk2dmntfz7zcsnzb22pghgmkv8h4cxff5
```

#### Modify the cluster definitions

* Navigate to Cluster.  Select the View in API from the Menu
![View in API](09-ClusterViewInAPI.png)

* Click the Edit button.
![Edit Button](10-EditButton.png)

* Modify the rancherKubernetesEngineConfig section.  You'll want to make the textarea larger. Add the extra args as highlighted to KubeAPI, Kubelet, and KubeControlManager.
* * Kube API Changes (The extraArgs setting)
```sh
"kubeApi":{"podSecurityPolicy":false, "type":"/v3/schemas/kubeAPIService", "extraArgs":{"requestheader-client-ca-file":"/etc/kubernetes/ssl/kube-ca.pem", "requestheader-extra-headers-prefix":"X-Remote-Extra-", "requestheader-group-headers":"X-Remote-Group", "requestheader-username-headers":"X-Remote-User", "proxy-client-cert-file":"/etc/kubernetes/ssl/kube-proxy.pem", "proxy-client-key-file":"/etc/kubernetes/ssl/kube-proxy-key.pem"}}
```
* * Kubelet Changes (only the extraArgs setting)
```sh
"kubelet":{"failSwapOn":false, "type":"/v3/schemas/kubeletService", "extraArgs":{"authentication-token-webhook":"true"}}
```
![KubeAPI Changes](11-KubeApiChanges.png)
![Kubelet Changes](12-KubeletChanges.png))

* Rancher will now show a cURL command line for your API request.
![API Request](13-APIRequest.png)

* Either scroll to the bottom to execute the request, copy and paste the request into your command line and execute it.  I would recommend you use the command line.  I had to add a -k since I'm using self-signed certificates. I also had to remove the amazonElasticContainerServiceConfig, azureKubernetesServiceConfig,googleKubernetesEngineConfig settings.  Otherwise this will error.  See my long-winded output below.

```sh
curl -u "${CATTLE_ACCESS_KEY}:${CATTLE_SECRET_KEY}" \
> -k \
4SFAo> -X PUT \
> -H 'Accept: application/json' \
> -H 'Content-Type: application/json' \
> -d '{"agentImage":"rancher/rancher-agent:v2.0.2", "allocatable":{"cpu":"3", "memory":"2740896Ki", "pods":"330"}, "annotations":{"lifecycle.cattle.io/create.cluster-agent-controller":"true", "lifecycle.cattle.io/create.cluster-provisioner-controller":"true", "lifecycle.cattle.io/create.cluster-scoped-gc":"true", "lifecycle.cattle.io/create.mgmt-cluster-rbac-remove":"true"}, "apiEndpoint":"https://104.236.74.116:6443", "appliedPodSecurityPolicyTemplateId":"", "appliedSpec":{"defaultClusterRoleForProjectMembers":null, "defaultPodSecurityPolicyTemplateId":null, "description":"", "desiredAgentImage":"", "displayName":"jvb-digitalocean", "internal":false, "rancherKubernetesEngineConfig":{"addonJobTimeout":30, "authentication":{"strategy":"x509", "type":"/v3/schemas/authnConfig"}, "authorization":{"type":"/v3/schemas/authzConfig"}, "bastionHost":{"sshAgentAuth":false, "type":"/v3/schemas/bastionHost"}, "cloudProvider":{"type":"/v3/schemas/cloudProvider"}, "ignoreDockerVersion":true, "ingress":{"provider":"nginx", "type":"/v3/schemas/ingressConfig"}, "kubernetesVersion":"v1.10.1-rancher2-1", "network":{"canalNetworkProvider":{"iface":"", "type":"/v3/schemas/canalNetworkProvider"}, "plugin":"canal", "type":"/v3/schemas/networkConfig"}, "nodes":[{"address":"104.236.78.112", "hostnameOverride":"jvb-etcd1", "nodeId":"c-9gc6d:m-9cg56", "port":"22", "role":["etcd"], "sshAgentAuth":false, "sshKey":"-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQEAxq2QjeKEwd66Rh0QK9lWCt5lG0h9IOgApSvqGVG30ByFsZp6\nWAS65jU4qvydL1jku1iY8aRw2qlwoSL/p4+SNJMjToPfds/CoLg2pg2LdUdceHvE\ntVQ+xuo7KjjxIq5+HMAyD6XMXzu2pepaZMdYIu0JDBNMDZGWv+EQI9IegYOUeE64\nOdk6ZxWCJXtkb+1rb9wMKm9IMoqUV3kZvORJDTJmYvBqAwEaLr5T7JNAAEWH9EWc\niNCzw6rOgj2THwxqLbXGAkHUr/ROwLHDE4HZtejE6SXTR4371h9adCqcZGdR0PzW\nV/6Y4mj3gH5S9XqBnhjHx+YyqLFzVB0Y1Pw87wIDAQABAoIBADmRpWdXkbxTe4pB\ndaQfD82Kb6WeNZPJZhZ2SRkZErgc66hw6K6EgFugZFE3bZyAl5apr3elfIX/vwES\nER5OfcD6OarQj3nv7aeAeWVYW2aaTPMkefVI+f/Nc2NiqIyyHxzZnCfolPWradtd\nst+s258TLCHxnYH5A8OJxZF7mxdPs9NxGES9he31NyyTN7KN11Q1xKTgK8nlgUPE\nVeXDlxXUez5gvLTOw3Fm8HcNroPVv7onqlHcCnz2dc/G/K6bBvpm+/Mp2/ffsnxn\nVBK+/ejqB1GHf0am5BCfGRbvSy8f/7xWyASbzO5gbdM10PpaXnnhOdAiEKg2anrx\ntx0jVeECgYEA+o9QxhacrbNE6XlUNMtL4KYrnhMFs+3n4dS9P25TDMh66Pxo4CFM\nKaKOrtF4qiPVYk1mUhgYyQlZLyAqvXVV62EaO/kQStncFsQf9KSgwZKq5Khl4S55\n5IMgC0VMztAp4xU2SSYP3mlhEIgLbqXlsXgxpZqUc1yAY/tyVlI4uhkCgYEAyv3f\n8QjRJxdeUWp4LMnKT0GLpJlbLDj4gBNOm+qgsy09NGdrYEk/D3F9MCx3EcGG4a7c\nbSxWy/N91EBjSB7IrEqEwH8LiMnU5A3M7TjjS9noN0px6Hh+QY12FDDUVoLcW/2b\n1vFxY+x9J8OU/uSGl9lWB9E4ZIqQ8+9hJ2qZoEcCgYAFZAP23xGr4wl+hG+9Wwpc\n4PZbRkl/H4U3pJ4VuUfsJHwQey2+dLjwxGCRfkw8TN0Qal3jMkWGPb8/dk+1F8sr\nb0FjPO6OzbUggEO3vbr/ryoiFUVr3NYehnuwAmOup2ffTNuII1uAC4BdHLAyRztz\n4AQhbfv3p2vTutsw5D/4MQKBgEUP1NXzsv0CxVMbzyEl0k/es/zfNBa6l0kzyGXh\nkKUFOjKPEQgkQ3Ekonv2VNhUqGg2DsaR9EHVaLUC5wRNfOapP7siPjFewbCJRXRc\nC3DuBURyboiyHfp9f3PlHQkF4qTlhtS9Nl/RwM77isChzWzyQ4PLWc1TG3fytHIt\noXDXAoGBAKPTv3bwML7D48LWUuNbTT0z+PhTgNpkU4QY5AZJFws0/Bdz/tifAiER\nuQZAQY0ukr11DiCgjX4ZXtNvNVcclMIYOM9Zn/1vg+NxeuPj+MqdMjfOUO1kYOOW\n5CfYGoX+N2gTHLCZCxII8tGleflP7PjPmLsKfytzoU9C+O50xidz\n-----END RSA PRIVATE KEY-----", "type":"/v3/schemas/rkeConfigNode", "user":"root"}, {"address":"104.236.97.28", "hostnameOverride":"jvb-worker1", "nodeId":"c-9gc6d:m-tjm9d", "port":"22", "role":["worker"], "sshAgentAuth":false, "sshKey":"-----BEGIN RSA PRIVATE KEY-----\nMIIEpQIBAAKCAQEArw58CogEAF4UUW0I0IXeUjfZZ9Jx1LrI7CJeBPg+EKoUbUnV\nsLHpIDnq5bb/yNzLj8kCqaQjfP9+3m0HBIFJ1fp7rnl19oOET09IPIlK/GMurzM6\nUQSQG3bJOazuIroU3kQ2ED246ZCtZw1lSujUkLJRRXOOG08W+ypBLE+eoC7DhhP4\n3zLc6JsHMmDWGb3tEu4cdGfQUVxb9OLAZW1QZS1yJqgKySAy/Xkuvq/OsSyPGD9q\no1wyCP6K5/pr7cMzDtnfzx0QtIkfEmfJZYii6Mtg7SkIlJNjPUXjdJOFrYJPWmDI\n21Y6rU+oT3C5KeOIj9SEiJywVvJHXj7vOPMx7QIDAQABAoIBAQCgOKKhRPJcn776\n88/AgvYDzGgolT1oVfj131KsO8UanYCKsqxpx7w2qavryj18S1Ho7TKJr9hnXLVy\nwNlOb0L0HQk8nKCyGpdru4XDGmnObLza0AlNA98ds+/i4PsoP6Y/9LCC1us6oxeC\nnYkbMdT0BrFJ3sSvDAldosP52Xm24vKxZ5Gog6jwakj8gYnjD055pQa+HNZfejT4\nh1y4capMD2lrYnKfuRbDmwCDLdYcih6oALrQwqKtqn1Bn2ZHFcazMIbdBGOndB0W\nqp4SFAoJWuPASW2yl7AGNm1lqQzO0TcFaxB6FMdYPQArsE6xajogw32lcNbu0AyR\nRB6kkuiFAoGBANdmnfoldlB/hTU00YfHxeYBJpuamq+U0G8nJ8YSOrYh1zryFfmz\nURS7rjriw28f9o7oXIzv78P36fhiysfbouuIjxJDu1NtnFGvj5yNPFOL63X4CPWc\ngIu2nmyFObdxHMzCbPV6CDShsLXamSE4NU/i4ngT0UqQrpAXlEA0vYDbAoGBANAN\nM6ruAPtf/DPow7NzAFsxmkvcORPUQI1q8MCDXy6BLfaYSBH9P6feD+MFgyXIaSEJ\nT72E3QXu+8wzO0aK4ZKJcw/CG5F/cyK5JXr3JVS9Fwl2qtgaJYy0zy7gMqvklKL/\nMSa4+XH/Obp4xD3JT6FQvVsP8DKmrHyDu5suQA7XAoGAQCIsnUM0NAmT5uCgFlZz\n5OSo0yG4NtkPKlk1LXNWCc6Pn9xekKdEV/D9kkjcOaUBq1Eh5VC3Sdjxz+PSTqll\nm81gE+cYymFTbrrf/pPKx3FcnRZJzJ9ZYr/v8Y3FPoouMgUdngVEli8iRzMb7tkm\nfm+mUGxOH25umzZ6DJfnaIsCgYEAl1GizKWCjVJIqrCW9gOktK8lHBu2X9e2g6TO\nKhbiuX43Ie2sLXlWPQ/j2ih+NJLI88BEAQGLh+9DsEdqy2njmqLhzwpM5swdv5mN\nyxvKSRGmmbl3lOmfefFf5yYldKQx/0NASnuq4eDR6fBAOvgaNKf2sS6scwqK1Z0y\nbDF9Yc0CgYEAipy/GSgLzIxj0mOKtbcdhlq+X1g4Wd117ssu+SWhutOWQ+7Be2sr\ngizB2QNRa25r0Wt7w6Ho/RQvS10iRiGhYh5XvxjQdvtOecA//CfZQSECoqk4MbA4\nLXAw6e2XIXn6wJ/tXR6sxVqrXyJXVfHZRaaCq41wPix3xiGL/Q0Y8ms=\n-----END RSA PRIVATE KEY-----", "type":"/v3/schemas/rkeConfigNode", "user":"root"}, {"address":"104.236.74.116", "hostnameOverride":"jvb-control1", "nodeId":"c-9gc6d:m-vzgt8", "port":"22", "role":["controlplane"], "sshAgentAuth":false, "sshKey":"-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAzz4TVS9rBS4StWZiKGfops/Z5aOjmrY11WVfCQNQMgvpu9MO\nHn44A5DJOmEnHgZBEJNqJ7CrmyogVRlFZxUZpUlSIZS+zvqv5O/XIgK9B1NQ1O9y\nwPmiuH7If1BbivABe11d3dg/iytkWEsZcQLmpSBk4IQqFqVkPo+89icYUTaYE+N9\ntPygbvUaJRyrLKGe/2WIbBFEgvYU7LpgxzRoRsvuWS0FZXu6v7sCwlmwllwCR5Bh\nRDLlG5z7e9SRALRuqqEBwPuX2nSgFZXSXdd99khRqDMtC5aP+9vHz2yKtNNXB0Cg\na33GNvF6rW3WuH2n+B7dFT8/WXXgF7k1yM8p+wIDAQABAoIBAApEihKrYcME8Fpa\nczyLQWSaYbJV9DIjGvGyrw20Q1HnczIiF8389OuATef9jkGXj7uxgg3rW+U5Jhbx\nRJYB3p5ozo/pXbbFhtdx5pfByPnG7/dM0cOr2psHdQmRwqdUks445wCpySQtBzC0\nr9MAkMrXmwfgpHEwypGq73LGpxbG5cbbIW/UPoOBLtJV1qEgpr/e7A148Co6HBD9\nL8/3xjhqDxqjpYF3NXotUKxy+1PGierSxklCOFGpDgZEGku3NlAF5xqsTILJR5oc\n614LAqi6SzYYiBIy+NURKyNWMoEowwMeFxDWBvyzwZloQs4kXYhSiBvPngIjbgPw\nUH9TfrECgYEA1NxMdHWi0MU8IP0P47fn4e2oBNc9BJZjAsCM50KaOOfYcXvkI17Z\nWQO3Be7lORzgzDiGTlEmwVQM8Xhfrv3Cfat9HoVhmZDECZmrCSSSHB130GiJ9xV1\noB/Lfx26UT79BE/niVg+y/iGY6JRyv2L0EqhWMnqiXSAhUC0eBzRVikCgYEA+T5M\ngeEAiPYk9gpun/Vz0L0XAXYAqpWXEZzm2Q/y3hHmhEyhGH80y78bViwt67TwUNYj\nxJPsjrLgZ2p+OIP4Ku1ODt2rPsYedoWnBdsuxy33sbSuy5/zcVCrvoXplYUIX9BB\nB2Q1m/hZFxv+Jd/AHmxwygrcGLPiV3/fQa6D24MCgYAhYDmMuTVCTurSsVtgdWGi\nh0UlHSqwIXR85fwx9rt/2LqTcL2g3ipuTqrP/1lpzyFgZCyK8yaEgc1Vv24hXiC0\n28LOmlNnZM+YQcQTtuxCBEzZHMslNvQum02pLNweCSZbGrxPSMrv6JQJ41W34nGd\nw1XxOhfQXbgZLMAfZUVdoQKBgQDNp/YBBOMoH9IHojv9iM5zYlLlRbTvDFeGlOx2\nFo7TZfnLGF+kkvHBXIL166ZIYxgX21nQDi6W4EoBiwFzKJFQ+lX6NjXiqh7gtQ9H\n6b8DYo6K9wdzxRteWK4dmulOEmMCjBfcPXduptD8HIDsR1+Cuv+l9qf9ShBtGnng\n/FqYeQKBgQCqnK6M7WJ2Ju459sybPtGGCbvXdkme5CMatI/uqyRCp6xmpiCp+/zr\nQr0xaGXD2pgIy6betbon7SZohe7BaJM865pVAeW4QIU1QIZAaVA7O0ZI4XrKljgN\nheLf1izYb5/SKBlNKi5YhOawEKIfE97HyVvUg5WTv//t+s7m6wxLnA==\n-----END RSA PRIVATE KEY-----", "type":"/v3/schemas/rkeConfigNode", "user":"root"}], "services":{"etcd":{"extraArgs":{"election-timeout":"5000", "heartbeat-interval":"500"}, "snapshot":false, "type":"/v3/schemas/etcdService"}, "kubeApi":{"podSecurityPolicy":false, "type":"/v3/schemas/kubeAPIService"}, "kubeController":{"type":"/v3/schemas/kubeControllerService"}, "kubelet":{"failSwapOn":false, "type":"/v3/schemas/kubeletService"}, "kubeproxy":{"type":"/v3/schemas/kubeproxyService"}, "scheduler":{"type":"/v3/schemas/schedulerService"}, "type":"/v3/schemas/rkeConfigServices"}, "sshAgentAuth":false, "type":"/v3/schemas/rancherKubernetesEngineConfig"}, "type":"/v3/schemas/clusterSpec"},  "caCert":"LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN3akNDQWFxZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFTTVJBd0RnWURWUVFERXdkcmRXSmwKTFdOaE1CNFhEVEU0TURZd05USXlNRE14TkZvWERUSTRNRFl3TWpJeU1ETXhORm93RWpFUU1BNEdBMVVFQXhNSAphM1ZpWlMxallUQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQUxSaElPZEhFZm40CmFVMHozem9xZGNxZ3pwa2NVMTJnNlBBcis5Qk93S3lHRndDUWRNT1VmNndXcStOZGtlSitpaXRJd2NML3Y2ZjMKVHhET0pYQ0J5TnZWUUs4a0NteXdIdW9zREpPdGxXekdCb3BhSktpMURzVUU0Si9ORUNWeFlCZjluN3RMTWEwdAptMXNJWmF2QS9SZlBOcDRDelM5V2lNcmE3S05jUG96Q1BPS01xNzFBaHo0MG1LbFhlcXNxOFVhV0ZSbGwwdXVkClF3MG91dkxYdmxhUHB4ZmYya0FkZGthSUVKdUEreU5lMDU5OWordnY1aHlmTGNhSFZrb3JvL1JUd09BK0hvT3YKenVlUHJabEx3Q05nMTVqcGc3Qm9OU2xQT3A4d2hudkJYUEtVTE1PMVRWRGUxc0NuaGdnTlRqdVdqS3d2dHp3KwpzVmNMaHo5RDNjRUNBd0VBQWFNak1DRXdEZ1lEVlIwUEFRSC9CQVFEQWdLa01BOEdBMVVkRXdFQi93UUZNQU1CCkFmOHdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBSytmUTA3UGRrTFBHMlZOS0VMT3lwQVhUb2xxa3BTWVlucHEKcEY0eDVTZXVHMDZyTlltSmRCK2gyMTN4R0h5VFRRZUwzdGo0WlV1cmdkTFppdTl6RVA0Q05FTzlPMk5RZnRrNApCa09GdkdrV0lWM1hUNkk0YUxBTDhvdC9hVXBPYmpZa3JHZXBETDNlR0l5aGVzOHZQR1cxSlkwUWk5OVppT2h5Ckd4MTFaL0ZKWFp3MFBqT0s2OHRsWHpSNFdBSGtYeDZRVGxtc09xVzJNbS9kaUt5U3UveWdWd2R3N3BxblZSRWkKeG5vNFVsNlIrSlRSZzdVaVRocGNNY2JGTWRhaHNxV0VadlpET09DNlI3MTFsSC8zNVVXa0lSWk4za3IxbFR1VgpOWHowY2tsYXpMdzl3a0NFMUhoRFV5c1Q2U2lkc21Zb3UyaTBCSm5kbnBoMFptMXM5cHM9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K", "capacity":{"cpu":"3", "memory":"3048096Ki", "pods":"330"}, "componentStatuses":[], "conditions":[], "created":"2018-06-05T22:01:00Z", "creatorId":"user-hcj65", "description":"", "desiredAgentImage":"", "driver":"rancherKubernetesEngine", "id":"c-9gc6d", "limits":{"cpu":"10m", "memory":"190Mi", "pods":"0"}, "name":"jvb-digitalocean", "ownerReferences":[], "rancherKubernetesEngineConfig":{"addonJobTimeout":30, "authentication":{"strategy":"x509", "type":"/v3/schemas/authnConfig"}, "authorization":{"type":"/v3/schemas/authzConfig"}, "bastionHost":{"sshAgentAuth":false, "type":"/v3/schemas/bastionHost"}, "cloudProvider":{"type":"/v3/schemas/cloudProvider"}, "ignoreDockerVersion":true, "ingress":{"provider":"nginx", "type":"/v3/schemas/ingressConfig"}, "kubernetesVersion":"v1.10.1-rancher2-1", "network":{"canalNetworkProvider":{"iface":"", "type":"/v3/schemas/canalNetworkProvider"}, "plugin":"canal", "type":"/v3/schemas/networkConfig"}, "services":{"etcd":{"extraArgs":{"election-timeout":"5000", "heartbeat-interval":"500"}, "snapshot":false, "type":"/v3/schemas/etcdService"}, "kubeApi":{"podSecurityPolicy":false, "type":"/v3/schemas/kubeAPIService", "extraArgs":{"requestheader-client-ca-file":"/etc/kubernetes/ssl/kube-ca.pem", "requestheader-extra-headers-prefix":"X-Remote-Extra-", "requestheader-group-headers":"X-Remote-Group", "requestheader-username-headers":"X-Remote-User", "proxy-client-cert-file":"/etc/kubernetes/ssl/kube-proxy.pem", "proxy-client-key-file":"/etc/kubernetes/ssl/kube-proxy-key.pem"}}, "kubeController":{"type":"/v3/schemas/kubeControllerService"}, "kubelet":{"failSwapOn":false, "type":"/v3/schemas/kubeletService", "extraArgs":{"authentication-token-webhook":"true"}}, "kubeproxy":{"type":"/v3/schemas/kubeproxyService"}, "scheduler":{"type":"/v3/schemas/schedulerService"}, "type":"/v3/schemas/rkeConfigServices"}, "sshAgentAuth":false, "type":"/v3/schemas/rancherKubernetesEngineConfig"}, "requested":{"cpu":"1040m", "memory":"140Mi", "pods":"12"}, "state":"active", "transitioning":"no", "transitioningMessage":"", "uuid":"ee87b4b4-690b-11e8-97db-0242ac110002", "version":{"buildDate":"2018-04-12T14:14:26Z", "compiler":"gc", "gitCommit":"d4ab47518836c750f9949b9e0d387f20fb92260b", "gitTreeState":"clean", "gitVersion":"v1.10.1", "goVersion":"go1.9.3", "major":"1", "minor":"10", "platform":"linux/amd64", "type":"/v3/schemas/info"}}' \
> 'https://jvb.ddns.net/v3/clusters/c-9gc6d'
{"actions":{"generateKubeconfig":"https://jvb.ddns.net/v3/clusters/c-9gc6d?action=generateKubeconfig","importYaml":"https://jvb.ddns.net/v3/clusters/c-9gc6d?action=importYaml"},"agentImage":"rancher/rancher-agent:v2.0.2","allocatable":{"cpu":"3","memory":"2740896Ki","pods":"330"},"annotations":{"lifecycle.cattle.io/create.cluster-agent-controller":"true","lifecycle.cattle.io/create.cluster-provisioner-controller":"true","lifecycle.cattle.io/create.cluster-scoped-gc":"true","lifecycle.cattle.io/create.mgmt-cluster-rbac-remove":"true"},"apiEndpoint":"https://104.236.74.116:6443","appliedPodSecurityPolicyTemplateId":"","appliedSpec":{"defaultClusterRoleForProjectMembers":null,"defaultPodSecurityPolicyTemplateId":null,"description":"","desiredAgentImage":"","displayName":"jvb-digitalocean","internal":false,"rancherKubernetesEngineConfig":{"addonJobTimeout":30,"authentication":{"strategy":"x509","type":"/v3/schemas/authnConfig"},"authorization":{"type":"/v3/schemas/authzConfig"},"bastionHost":{"sshAgentAuth":false,"type":"/v3/schemas/bastionHost"},"cloudProvider":{"type":"/v3/schemas/cloudProvider"},"ignoreDockerVersion":true,"ingress":{"provider":"nginx","type":"/v3/schemas/ingressConfig"},"kubernetesVersion":"v1.10.1-rancher2-1","network":{"canalNetworkProvider":{"iface":"","type":"/v3/schemas/canalNetworkProvider"},"plugin":"canal","type":"/v3/schemas/networkConfig"},"nodes":[{"address":"104.236.78.112","hostnameOverride":"jvb-etcd1","nodeId":"c-9gc6d:m-9cg56","port":"22","role":["etcd"],"sshAgentAuth":false,"sshKey":"-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQEAxq2QjeKEwd66Rh0QK9lWCt5lG0h9IOgApSvqGVG30ByFsZp6\nWAS65jU4qvydL1jku1iY8aRw2qlwoSL/p4+SNJMjToPfds/CoLg2pg2LdUdceHvE\ntVQ+xuo7KjjxIq5+HMAyD6XMXzu2pepaZMdYIu0JDBNMDZGWv+EQI9IegYOUeE64\nOdk6ZxWCJXtkb+1rb9wMKm9IMoqUV3kZvORJDTJmYvBqAwEaLr5T7JNAAEWH9EWc\niNCzw6rOgj2THwxqLbXGAkHUr/ROwLHDE4HZtejE6SXTR4371h9adCqcZGdR0PzW\nV/6Y4mj3gH5S9XqBnhjHx+YyqLFzVB0Y1Pw87wIDAQABAoIBADmRpWdXkbxTe4pB\ndaQfD82Kb6WeNZPJZhZ2SRkZErgc66hw6K6EgFugZFE3bZyAl5apr3elfIX/vwES\nER5OfcD6OarQj3nv7aeAeWVYW2aaTPMkefVI+f/Nc2NiqIyyHxzZnCfolPWradtd\nst+s258TLCHxnYH5A8OJxZF7mxdPs9NxGES9he31NyyTN7KN11Q1xKTgK8nlgUPE\nVeXDlxXUez5gvLTOw3Fm8HcNroPVv7onqlHcCnz2dc/G/K6bBvpm+/Mp2/ffsnxn\nVBK+/ejqB1GHf0am5BCfGRbvSy8f/7xWyASbzO5gbdM10PpaXnnhOdAiEKg2anrx\ntx0jVeECgYEA+o9QxhacrbNE6XlUNMtL4KYrnhMFs+3n4dS9P25TDMh66Pxo4CFM\nKaKOrtF4qiPVYk1mUhgYyQlZLyAqvXVV62EaO/kQStncFsQf9KSgwZKq5Khl4S55\n5IMgC0VMztAp4xU2SSYP3mlhEIgLbqXlsXgxpZqUc1yAY/tyVlI4uhkCgYEAyv3f\n8QjRJxdeUWp4LMnKT0GLpJlbLDj4gBNOm+qgsy09NGdrYEk/D3F9MCx3EcGG4a7c\nbSxWy/N91EBjSB7IrEqEwH8LiMnU5A3M7TjjS9noN0px6Hh+QY12FDDUVoLcW/2b\n1vFxY+x9J8OU/uSGl9lWB9E4ZIqQ8+9hJ2qZoEcCgYAFZAP23xGr4wl+hG+9Wwpc\n4PZbRkl/H4U3pJ4VuUfsJHwQey2+dLjwxGCRfkw8TN0Qal3jMkWGPb8/dk+1F8sr\nb0FjPO6OzbUggEO3vbr/ryoiFUVr3NYehnuwAmOup2ffTNuII1uAC4BdHLAyRztz\n4AQhbfv3p2vTutsw5D/4MQKBgEUP1NXzsv0CxVMbzyEl0k/es/zfNBa6l0kzyGXh\nkKUFOjKPEQgkQ3Ekonv2VNhUqGg2DsaR9EHVaLUC5wRNfOapP7siPjFewbCJRXRc\nC3DuBURyboiyHfp9f3PlHQkF4qTlhtS9Nl/RwM77isChzWzyQ4PLWc1TG3fytHIt\noXDXAoGBAKPTv3bwML7D48LWUuNbTT0z+PhTgNpkU4QY5AZJFws0/Bdz/tifAiER\nuQZAQY0ukr11DiCgjX4ZXtNvNVcclMIYOM9Zn/1vg+NxeuPj+MqdMjfOUO1kYOOW\n5CfYGoX+N2gTHLCZCxII8tGleflP7PjPmLsKfytzoU9C+O50xidz\n-----END RSA PRIVATE KEY-----","type":"/v3/schemas/rkeConfigNode","user":"root"},{"address":"104.236.97.28","hostnameOverride":"jvb-worker1","nodeId":"c-9gc6d:m-tjm9d","port":"22","role":["worker"],"sshAgentAuth":false,"sshKey":"-----BEGIN RSA PRIVATE KEY-----\nMIIEpQIBAAKCAQEArw58CogEAF4UUW0I0IXeUjfZZ9Jx1LrI7CJeBPg+EKoUbUnV\nsLHpIDnq5bb/yNzLj8kCqaQjfP9+3m0HBIFJ1fp7rnl19oOET09IPIlK/GMurzM6\nUQSQG3bJOazuIroU3kQ2ED246ZCtZw1lSujUkLJRRXOOG08W+ypBLE+eoC7DhhP4\n3zLc6JsHMmDWGb3tEu4cdGfQUVxb9OLAZW1QZS1yJqgKySAy/Xkuvq/OsSyPGD9q\no1wyCP6K5/pr7cMzDtnfzx0QtIkfEmfJZYii6Mtg7SkIlJNjPUXjdJOFrYJPWmDI\n21Y6rU+oT3C5KeOIj9SEiJywVvJHXj7vOPMx7QIDAQABAoIBAQCgOKKhRPJcn776\n88/AgvYDzGgolT1oVfj131KsO8UanYCKsqxpx7w2qavryj18S1Ho7TKJr9hnXLVy\nwNlOb0L0HQk8nKCyGpdru4XDGmnObLza0AlNA98ds+/i4PsoP6Y/9LCC1us6oxeC\nnYkbMdT0BrFJ3sSvDAldosP52Xm24vKxZ5Gog6jwakj8gYnjD055pQa+HNZfejT4\nh1y4capMD2lrYnKfuRbDmwCDLdYcih6oALrQwqKtqn1Bn2ZHFcazMIbdBGOndB0W\nqp4SFAoJWuPASW2yl7AGNm1lqQzO0TcFaxB6FMdYPQArsE6xajogw32lcNbu0AyR\nRB6kkuiFAoGBANdmnfoldlB/hTU00YfHxeYBJpuamq+U0G8nJ8YSOrYh1zryFfmz\nURS7rjriw28f9o7oXIzv78P36fhiysfbouuIjxJDu1NtnFGvj5yNPFOL63X4CPWc\ngIu2nmyFObdxHMzCbPV6CDShsLXamSE4NU/i4ngT0UqQrpAXlEA0vYDbAoGBANAN\nM6ruAPtf/DPow7NzAFsxmkvcORPUQI1q8MCDXy6BLfaYSBH9P6feD+MFgyXIaSEJ\nT72E3QXu+8wzO0aK4ZKJcw/CG5F/cyK5JXr3JVS9Fwl2qtgaJYy0zy7gMqvklKL/\nMSa4+XH/Obp4xD3JT6FQvVsP8DKmrHyDu5suQA7XAoGAQCIsnUM0NAmT5uCgFlZz\n5OSo0yG4NtkPKlk1LXNWCc6Pn9xekKdEV/D9kkjcOaUBq1Eh5VC3Sdjxz+PSTqll\nm81gE+cYymFTbrrf/pPKx3FcnRZJzJ9ZYr/v8Y3FPoouMgUdngVEli8iRzMb7tkm\nfm+mUGxOH25umzZ6DJfnaIsCgYEAl1GizKWCjVJIqrCW9gOktK8lHBu2X9e2g6TO\nKhbiuX43Ie2sLXlWPQ/j2ih+NJLI88BEAQGLh+9DsEdqy2njmqLhzwpM5swdv5mN\nyxvKSRGmmbl3lOmfefFf5yYldKQx/0NASnuq4eDR6fBAOvgaNKf2sS6scwqK1Z0y\nbDF9Yc0CgYEAipy/GSgLzIxj0mOKtbcdhlq+X1g4Wd117ssu+SWhutOWQ+7Be2sr\ngizB2QNRa25r0Wt7w6Ho/RQvS10iRiGhYh5XvxjQdvtOecA//CfZQSECoqk4MbA4\nLXAw6e2XIXn6wJ/tXR6sxVqrXyJXVfHZRaaCq41wPix3xiGL/Q0Y8ms=\n-----END RSA PRIVATE KEY-----","type":"/v3/schemas/rkeConfigNode","user":"root"},{"address":"104.236.74.116","hostnameOverride":"jvb-control1","nodeId":"c-9gc6d:m-vzgt8","port":"22","role":["controlplane"],"sshAgentAuth":false,"sshKey":"-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAzz4TVS9rBS4StWZiKGfops/Z5aOjmrY11WVfCQNQMgvpu9MO\nHn44A5DJOmEnHgZBEJNqJ7CrmyogVRlFZxUZpUlSIZS+zvqv5O/XIgK9B1NQ1O9y\nwPmiuH7If1BbivABe11d3dg/iytkWEsZcQLmpSBk4IQqFqVkPo+89icYUTaYE+N9\ntPygbvUaJRyrLKGe/2WIbBFEgvYU7LpgxzRoRsvuWS0FZXu6v7sCwlmwllwCR5Bh\nRDLlG5z7e9SRALRuqqEBwPuX2nSgFZXSXdd99khRqDMtC5aP+9vHz2yKtNNXB0Cg\na33GNvF6rW3WuH2n+B7dFT8/WXXgF7k1yM8p+wIDAQABAoIBAApEihKrYcME8Fpa\nczyLQWSaYbJV9DIjGvGyrw20Q1HnczIiF8389OuATef9jkGXj7uxgg3rW+U5Jhbx\nRJYB3p5ozo/pXbbFhtdx5pfByPnG7/dM0cOr2psHdQmRwqdUks445wCpySQtBzC0\nr9MAkMrXmwfgpHEwypGq73LGpxbG5cbbIW/UPoOBLtJV1qEgpr/e7A148Co6HBD9\nL8/3xjhqDxqjpYF3NXotUKxy+1PGierSxklCOFGpDgZEGku3NlAF5xqsTILJR5oc\n614LAqi6SzYYiBIy+NURKyNWMoEowwMeFxDWBvyzwZloQs4kXYhSiBvPngIjbgPw\nUH9TfrECgYEA1NxMdHWi0MU8IP0P47fn4e2oBNc9BJZjAsCM50KaOOfYcXvkI17Z\nWQO3Be7lORzgzDiGTlEmwVQM8Xhfrv3Cfat9HoVhmZDECZmrCSSSHB130GiJ9xV1\noB/Lfx26UT79BE/niVg+y/iGY6JRyv2L0EqhWMnqiXSAhUC0eBzRVikCgYEA+T5M\ngeEAiPYk9gpun/Vz0L0XAXYAqpWXEZzm2Q/y3hHmhEyhGH80y78bViwt67TwUNYj\nxJPsjrLgZ2p+OIP4Ku1ODt2rPsYedoWnBdsuxy33sbSuy5/zcVCrvoXplYUIX9BB\nB2Q1m/hZFxv+Jd/AHmxwygrcGLPiV3/fQa6D24MCgYAhYDmMuTVCTurSsVtgdWGi\nh0UlHSqwIXR85fwx9rt/2LqTcL2g3ipuTqrP/1lpzyFgZCyK8yaEgc1Vv24hXiC0\n28LOmlNnZM+YQcQTtuxCBEzZHMslNvQum02pLNweCSZbGrxPSMrv6JQJ41W34nGd\nw1XxOhfQXbgZLMAfZUVdoQKBgQDNp/YBBOMoH9IHojv9iM5zYlLlRbTvDFeGlOx2\nFo7TZfnLGF+kkvHBXIL166ZIYxgX21nQDi6W4EoBiwFzKJFQ+lX6NjXiqh7gtQ9H\n6b8DYo6K9wdzxRteWK4dmulOEmMCjBfcPXduptD8HIDsR1+Cuv+l9qf9ShBtGnng\n/FqYeQKBgQCqnK6M7WJ2Ju459sybPtGGCbvXdkme5CMatI/uqyRCp6xmpiCp+/zr\nQr0xaGXD2pgIy6betbon7SZohe7BaJM865pVAeW4QIU1QIZAaVA7O0ZI4XrKljgN\nheLf1izYb5/SKBlNKi5YhOawEKIfE97HyVvUg5WTv//t+s7m6wxLnA==\n-----END RSA PRIVATE KEY-----","type":"/v3/schemas/rkeConfigNode","user":"root"}],"services":{"etcd":{"extraArgs":{"election-timeout":"5000","heartbeat-interval":"500"},"snapshot":false,"type":"/v3/schemas/etcdService"},"kubeApi":{"podSecurityPolicy":false,"type":"/v3/schemas/kubeAPIService"},"kubeController":{"type":"/v3/schemas/kubeControllerService"},"kubelet":{"failSwapOn":false,"type":"/v3/schemas/kubeletService"},"kubeproxy":{"type":"/v3/schemas/kubeproxyService"},"scheduler":{"type":"/v3/schemas/schedulerService"},"type":"/v3/schemas/rkeConfigServices"},"sshAgentAuth":false,"type":"/v3/schemas/rancherKubernetesEngineConfig"},"type":"/v3/schemas/clusterSpec"},"baseType":"cluster","caCert":"LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN3akNDQWFxZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFTTVJBd0RnWURWUVFERXdkcmRXSmwKTFdOaE1CNFhEVEU0TURZd05USXlNRE14TkZvWERUSTRNRFl3TWpJeU1ETXhORm93RWpFUU1BNEdBMVVFQXhNSAphM1ZpWlMxallUQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQUxSaElPZEhFZm40CmFVMHozem9xZGNxZ3pwa2NVMTJnNlBBcis5Qk93S3lHRndDUWRNT1VmNndXcStOZGtlSitpaXRJd2NML3Y2ZjMKVHhET0pYQ0J5TnZWUUs4a0NteXdIdW9zREpPdGxXekdCb3BhSktpMURzVUU0Si9ORUNWeFlCZjluN3RMTWEwdAptMXNJWmF2QS9SZlBOcDRDelM5V2lNcmE3S05jUG96Q1BPS01xNzFBaHo0MG1LbFhlcXNxOFVhV0ZSbGwwdXVkClF3MG91dkxYdmxhUHB4ZmYya0FkZGthSUVKdUEreU5lMDU5OWordnY1aHlmTGNhSFZrb3JvL1JUd09BK0hvT3YKenVlUHJabEx3Q05nMTVqcGc3Qm9OU2xQT3A4d2hudkJYUEtVTE1PMVRWRGUxc0NuaGdnTlRqdVdqS3d2dHp3KwpzVmNMaHo5RDNjRUNBd0VBQWFNak1DRXdEZ1lEVlIwUEFRSC9CQVFEQWdLa01BOEdBMVVkRXdFQi93UUZNQU1CCkFmOHdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBSytmUTA3UGRrTFBHMlZOS0VMT3lwQVhUb2xxa3BTWVlucHEKcEY0eDVTZXVHMDZyTlltSmRCK2gyMTN4R0h5VFRRZUwzdGo0WlV1cmdkTFppdTl6RVA0Q05FTzlPMk5RZnRrNApCa09GdkdrV0lWM1hUNkk0YUxBTDhvdC9hVXBPYmpZa3JHZXBETDNlR0l5aGVzOHZQR1cxSlkwUWk5OVppT2h5Ckd4MTFaL0ZKWFp3MFBqT0s2OHRsWHpSNFdBSGtYeDZRVGxtc09xVzJNbS9kaUt5U3UveWdWd2R3N3BxblZSRWkKeG5vNFVsNlIrSlRSZzdVaVRocGNNY2JGTWRhaHNxV0VadlpET09DNlI3MTFsSC8zNVVXa0lSWk4za3IxbFR1VgpOWHowY2tsYXpMdzl3a0NFMUhoRFV5c1Q2U2lkc21Zb3UyaTBCSm5kbnBoMFptMXM5cHM9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K","capacity":{"cpu":"3","memory":"3048096Ki","pods":"330"},"componentStatuses":[{"conditions":[{"message":"ok","status":"True","type":"Healthy"}],"name":"controller-manager","type":"/v3/schemas/clusterComponentStatus"},{"conditions":[{"message":"{\"health\": \"true\"}","status":"True","type":"Healthy"}],"name":"etcd-0","type":"/v3/schemas/clusterComponentStatus"},{"conditions":[{"message":"ok","status":"True","type":"Healthy"}],"name":"scheduler","type":"/v3/schemas/clusterComponentStatus"}],"conditions":[{"lastUpdateTime":"2018-06-05T22:01:00Z","status":"True","type":"BackingNamespaceCreated"},{"lastUpdateTime":"2018-06-05T22:01:00Z","status":"True","type":"DefaultProjectCreated"},{"lastUpdateTime":"2018-06-05T22:01:00Z","status":"True","type":"CreatorMadeOwner"},{"lastUpdateTime":"2018-06-05T22:01:00Z","status":"True","type":"Pending"},{"lastUpdateTime":"2018-06-05T22:05:39Z","status":"True","type":"Provisioned"},{"lastUpdateTime":"2018-06-05T22:05:55Z","status":"True","type":"Waiting"},{"lastUpdateTime":"2018-06-05T22:01:00Z","status":"True","type":"InitialRolesPopulated"},{"lastUpdateTime":"2018-06-05T22:01:00Z","status":"True","type":"NoDiskPressure"},{"lastUpdateTime":"2018-06-05T22:01:00Z","status":"True","type":"NoMemoryPressure"},{"lastUpdateTime":"2018-06-05T22:05:41Z","status":"True","type":"GlobalAdminsSynced"},{"lastUpdateTime":"2018-06-05T22:05:42Z","status":"True","type":"DefaultNamespaceAssigned"},{"lastUpdateTime":"2018-06-05T22:05:44Z","status":"True","type":"Updated"},{"lastUpdateTime":"2018-06-05T22:05:44Z","status":"True","type":"SystemAccountCreated"},{"lastUpdateTime":"2018-06-05T22:05:45Z","status":"True","type":"AgentDeployed"},{"lastUpdateTime":"2018-06-06T01:16:20Z","status":"True","type":"Ready"}],"created":"2018-06-05T22:01:00Z","createdTS":1528236060000,"creatorId":"user-hcj65","defaultClusterRoleForProjectMembers":null,"defaultPodSecurityPolicyTemplateId":null,"description":"","desiredAgentImage":"","driver":"rancherKubernetesEngine","id":"c-9gc6d","internal":false,"labels":null,"limits":{"cpu":"10m","memory":"190Mi","pods":"0"},"links":{"clusterAlerts":"https://jvb.ddns.net/v3/clusters/c-9gc6d/clusteralerts","clusterComposeConfigs":"https://jvb.ddns.net/v3/clusters/c-9gc6d/clustercomposeconfigs","clusterEvents":"https://jvb.ddns.net/v3/clusters/c-9gc6d/clusterevents","clusterLoggings":"https://jvb.ddns.net/v3/clusters/c-9gc6d/clusterloggings","clusterPipelines":"https://jvb.ddns.net/v3/clusters/c-9gc6d/clusterpipelines","clusterRegistrationTokens":"https://jvb.ddns.net/v3/clusters/c-9gc6d/clusterregistrationtokens","clusterRoleTemplateBindings":"https://jvb.ddns.net/v3/clusters/c-9gc6d/clusterroletemplatebindings","namespaces":"https://jvb.ddns.net/v3/clusters/c-9gc6d/namespaces","nodePools":"https://jvb.ddns.net/v3/clusters/c-9gc6d/nodepools","nodes":"https://jvb.ddns.net/v3/clusters/c-9gc6d/nodes","notifiers":"https://jvb.ddns.net/v3/clusters/c-9gc6d/notifiers","persistentVolumes":"https://jvb.ddns.net/v3/clusters/c-9gc6d/persistentvolumes","projects":"https://jvb.ddns.net/v3/clusters/c-9gc6d/projects","remove":"https://jvb.ddns.net/v3/clusters/c-9gc6d","self":"https://jvb.ddns.net/v3/clusters/c-9gc6d","shell":"wss://jvb.ddns.net/v3/clusters/c-9gc6d?shell=true","sourceCodeCredentials":"https://jvb.ddns.net/v3/clusters/c-9gc6d/sourcecodecredentials","sourceCodeRepositories":"https://jvb.ddns.net/v3/clusters/c-9gc6d/sourcecoderepositories","storageClasses":"https://jvb.ddns.net/v3/clusters/c-9gc6d/storageclasses","subscribe":"https://jvb.ddns.net/v3/clusters/c-9gc6d/subscribe","update":"https://jvb.ddns.net/v3/clusters/c-9gc6d"},"name":"jvb-digitalocean","rancherKubernetesEngineConfig":{"addonJobTimeout":30,"authentication":{"strategy":"x509","type":"/v3/schemas/authnConfig"},"authorization":{"type":"/v3/schemas/authzConfig"},"bastionHost":{"sshAgentAuth":false,"type":"/v3/schemas/bastionHost"},"cloudProvider":{"type":"/v3/schemas/cloudProvider"},"ignoreDockerVersion":true,"ingress":{"provider":"nginx","type":"/v3/schemas/ingressConfig"},"kubernetesVersion":"v1.10.1-rancher2-1","network":{"canalNetworkProvider":{"iface":"","type":"/v3/schemas/canalNetworkProvider"},"plugin":"canal","type":"/v3/schemas/networkConfig"},"services":{"etcd":{"extraArgs":{"election-timeout":"5000","heartbeat-interval":"500"},"snapshot":false,"type":"/v3/schemas/etcdService"},"kubeApi":{"extraArgs":{"proxy-client-cert-file":"/etc/kubernetes/ssl/kube-proxy.pem","proxy-client-key-file":"/etc/kubernetes/ssl/kube-proxy-key.pem","requestheader-client-ca-file":"/etc/kubernetes/ssl/kube-ca.pem","requestheader-extra-headers-prefix":"X-Remote-Extra-","requestheader-group-headers":"X-Remote-Group","requestheader-username-headers":"X-Remote-User"},"podSecurityPolicy":false,"type":"/v3/schemas/kubeAPIService"},"kubeController":{"type":"/v3/schemas/kubeControllerService"},"kubelet":{"extraArgs":{"authentication-token-webhook":"true"},"failSwapOn":false,"type":"/v3/schemas/kubeletService"},"kubeproxy":{"type":"/v3/schemas/kubeproxyService"},"scheduler":{"type":"/v3/schemas/schedulerService"},"type":"/v3/schemas/rkeConfigServices"},"sshAgentAuth":false,"type":"/v3/schemas/rancherKubernetesEngineConfig"},"requested":{"cpu":"1040m","memory":"140Mi","pods":"12"},"state":"active","transitioning":"no","transitioningMessage":"","type":"cluster","uuid":"ee87b4b4-690b-11e8-97db-0242ac110002","version":{"buildDate":"2018-04-12T14:14:26Z","compiler":"gc","gitCommit":"d4ab47518836c750f9949b9e0d387f20fb92260b","gitTreeState":"clean","gitVersion":"v1.10.1","goVersion":"go1.9.3","major":"1","minor":"10","platform":"linux/amd64","type":"/v3/schemas/info"}}
```

* You can confirm the changes by checking the cluster again under View in API as noted before.

**At this point kubelet and kubeapi should be restarted with the appropriate extra args**. 

------------------------------------------------------------------------------------------------------------------------------

### Modify running instances

**Take Note:** Below is a general guide for clusters that were created in a Rancher-like fashion, but not via an IaaS backed Node Driver.  This mean any Kubernetes Cluster that is leveraging Docker Engine and the hyperkube container. 

This will not work on IaaS-Backed Rancher Clusters.  You will have a bad time as Rancher will fix the changes you make to the docker container configuration

* ssh into each control-plane nodes
```sh
[jason@vblinux metrics-server-on-rancher-2.0.2 ]$ rancher nodes
ID                NAME           STATE     POOL          DESCRIPTION
c-nng62:m-7ldm4   jvb-etcd1      active    jvb-etcd      
c-nng62:m-qswks   jvb-worker1    active    jvb-worker    
c-nng62:m-vhnqg   jvb-control1   active    jvb-control   
[jason@vblinux metrics-server-on-rancher-2.0.2 ]$ rancher ssh jvb-control1
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-127-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

9 packages can be updated.
0 updates are security updates.


Last login: Tue Jun  5 16:27:37 2018 from 74.109.3.176
```
#### Updating kube-apiserver and kubelet
We're going to modify the kube-apiserver and kubelet.  Let's do kube-apiserver first. Remember for multiple nodes tmux in synchronize-panes mode is your friend.


* Stop the kube-apiserver
```sh
root@jvb-control1:~# docker ps --filter "Name=kube-apiserver"
CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS               NAMES
860ece4f0d0f        rancher/hyperkube:v1.10.1-rancher2   "/opt/rke/entrypoi..."   26 hours ago        Up 26 hours                             kube-apiserver
root@jvb-control1:~# docker stop kube-apiserver
kube-apiserver

```
* Rename the kube-apiserver container
```sh
root@jvb-control1:~# docker rename kube-apiserver kube-apiserverold
```

* Download [a docker template file](https://gist.githubusercontent.com/JasonvanBrackel/81b5501ecb62618fb67377a5ebe601b3) to generate the command used to run kube-apiserver
```sh
root@jvb-control1:~# curl -o run.tpl https://gist.githubusercontent.com/JasonvanBrackel/81b5501ecb62618fb67377a5ebe601b3/raw/cb77f6d12aa9ce2ff1e2fd425710131d6a6cdfcb/run.tpl
```

* Copy the output of the template.  Paste it into a text editor.
```sh
root@jvb-control1:~# docker inspect --format "$(<run.tpl)" kube-apiserverold
docker run \
    --name=/kube-apiserverold \
    --env="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
    --network "host" \
     \
    --volumes-from="service-sidekick" \
    --volume="/etc/kubernetes:/etc/kubernetes:z" \
    --label "io.rancher.rke.container.name"="kube-apiserver" \
    --label "org.label-schema.build-date"="2018-04-25T05:52:34Z" \
    --label "org.label-schema.vcs-ref"="1e8a3dc715cbc00a561e1d84645a6378302b7c8c" \
    --label "org.label-schema.vcs-url"="https://github.com/rancher/hyperkube.git" \
    --entrypoint=/opt/rke/entrypoint.sh \
    --log-driver="json-file" \
    --restart="always" \
    --detach=true \
    "rancher/hyperkube:v1.10.1-rancher2" \
    "kube-apiserver"  "--bind-address=0.0.0.0"  "--tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem"  "--kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem"  "--service-account-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem"  "--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"  "--kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem"  "--apiserver-count=1"  "--feature-gates=MountPropagation=false"  "--insecure-port=0"  "--secure-port=6443"  "--storage-backend=etcd3"  "--service-cluster-ip-range=10.43.0.0/16"  "--client-ca-file=/etc/kubernetes/ssl/kube-ca.pem"  "--authorization-mode=Node,RBAC"  "--insecure-bind-address=127.0.0.1"  "--admission-control=ServiceAccount,NamespaceLifecycle,LimitRanger,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds"  "--cloud-provider="  "--allow-privileged=true"  "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname"  "--tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem"  "--etcd-cafile=/etc/kubernetes/ssl/kube-ca.pem" "--etcd-certfile=/etc/kubernetes/ssl/kube-node.pem" "--etcd-keyfile=/etc/kubernetes/ssl/kube-node-key.pem" "--etcd-servers=https://104.131.114.197:2379" "--etcd-prefix=/registry"
```

* Add these to the end and rename the container kube-apiserver.  
```sh
"--requestheader-client-ca-file=/etc/kubernetes/ssl/kube-ca.pem" "--requestheader-extra-headers-prefix=X-Remote-Extra-" "--requestheader-group-headers=X-Remote-Group" "--requestheader-username-headers=X-Remote-User" "--proxy-client-cert-file=/etc/kubernetes/ssl/kube-proxy.pem" "--proxy-client-key-file=/etc/kubernetes/ssl/kube-proxy-key.pem"
```


* The finished product should look something like this.
```sh
docker run \
    --name=/kube-apiserver \
    --env="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
    --network "host" \
     \
    --volumes-from="service-sidekick" \
    --volume="/etc/kubernetes:/etc/kubernetes:z" \
    --label "io.rancher.rke.container.name"="kube-apiserver" \
    --label "org.label-schema.build-date"="2018-04-25T05:52:34Z" \
    --label "org.label-schema.vcs-ref"="1e8a3dc715cbc00a561e1d84645a6378302b7c8c" \
    --label "org.label-schema.vcs-url"="https://github.com/rancher/hyperkube.git" \
    --entrypoint=/opt/rke/entrypoint.sh \
    --log-driver="json-file" \
    --restart="always" \
    --detach=true \
    "rancher/hyperkube:v1.10.1-rancher2" \
    "kube-apiserver"  "--bind-address=0.0.0.0"  "--tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem"  "--kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem"  "--service-account-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem"  "--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"  "--kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem"  "--apiserver-count=1"  "--feature-gates=MountPropagation=false"  "--insecure-port=0"  "--secure-port=6443"  "--storage-backend=etcd3"  "--service-cluster-ip-range=10.43.0.0/16"  "--client-ca-file=/etc/kubernetes/ssl/kube-ca.pem"  "--authorization-mode=Node,RBAC"  "--insecure-bind-address=127.0.0.1"  "--admission-control=ServiceAccount,NamespaceLifecycle,LimitRanger,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds"  "--cloud-provider="  "--allow-privileged=true"  "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname"  "--tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem"  "--etcd-cafile=/etc/kubernetes/ssl/kube-ca.pem" "--etcd-certfile=/etc/kubernetes/ssl/kube-node.pem" "--etcd-keyfile=/etc/kubernetes/ssl/kube-node-key.pem" "--etcd-servers=https://104.131.114.197:2379" "--etcd-prefix=/registry" "--requestheader-client-ca-file=/etc/kubernetes/ssl/kube-ca.pem" "--requestheader-extra-headers-prefix=X-Remote-Extra-" "--requestheader-group-headers=X-Remote-Group" "--requestheader-username-headers=X-Remote-User" "--proxy-client-cert-file=/etc/kubernetes/ssl/kube-proxy.pem" "--proxy-client-key-file=/etc/kubernetes/ssl/kube-proxy-key.pem"
```

* run the updated command
```sh
root@jvb-control1:~# docker run \
>     --name=/kube-apiserver \
>     --env="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
>     --network "host" \
>      \
>     --volumes-from="service-sidekick" \
>     --volume="/etc/kubernetes:/etc/kubernetes:z" \
>     --label "io.rancher.rke.container.name"="kube-apiserver" \
>     --label "org.label-schema.build-date"="2018-04-25T05:52:34Z" \
>     --label "org.label-schema.vcs-ref"="1e8a3dc715cbc00a561e1d84645a6378302b7c8c" \
>     --label "org.label-schema.vcs-url"="https://github.com/rancher/hyperkube.git" \
>     --entrypoint=/opt/rke/entrypoint.sh \
>     --log-driver="json-file" \
>     --restart="always" \
>     --detach=true \
>     "rancher/hyperkube:v1.10.1-rancher2" \
>     "kube-apiserver"  "--bind-address=0.0.0.0"  "--tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem"  "--kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem"  "--service-account-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem"  "--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"  "--kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem"  "--apiserver-count=1"  "--feature-gates=MountPropagation=false"  "--insecure-port=0"  "--secure-port=6443"  "--storage-backend=etcd3"  "--service-cluster-ip-range=10.43.0.0/16"  "--client-ca-file=/etc/kubernetes/ssl/kube-ca.pem"  "--authorization-mode=Node,RBAC"  "--insecure-bind-address=127.0.0.1"  "--admission-control=ServiceAccount,NamespaceLifecycle,LimitRanger,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds"  "--cloud-provider="  "--allow-privileged=true"  "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname"  "--tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem"  "--etcd-cafile=/etc/kubernetes/ssl/kube-ca.pem" "--etcd-certfile=/etc/kubernetes/ssl/kube-node.pem" "--etcd-keyfile=/etc/kubernetes/ssl/kube-node-key.pem" "--etcd-servers=https://104.131.114.197:2379" "--etcd-prefix=/registry"  "--requestheader-client-ca-file=/etc/kubernetes/ssl/kube-ca.pem" "--requestheader-extra-headers-prefix=X-Remote-Extra-" "--requestheader-group-headers=X-Remote-Group" "--requestheader-username-headers=X-Remote-User" "--proxy-client-cert-file=/etc/kubernetes/ssl/kube-proxy.pem" "--proxy-client-key-file=/etc/kubernetes/ssl/kube-proxy-key.pem"
968a060d0165300ef9d8ef7cd6fe7c3dee43d588d425b506a02fa8deb548642a
```
* Update the old container's restart policy so it will not start again.
```sh
root@jvb-control1:~# docker update kube-apiserverold --restart "no"
```

Let's do the same for kubelet.  We'll need to do this on every kublet not just the control-plane nodes.  tmux synchronize-panes is your friend.

* Shutdown kubelet.
```sh
root@jvb-control1:~# docker stop kubelet
kubelet
```
* Rename the kubelet docker container.
```sh
root@jvb-control1:~# docker rename kubelet kubeletold
```

* Run the template against the docker inspect for the kubelet.
```sh
docker run \
    --name=/kubeletold \
    --env="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
    --network "host" \
     \
    --volumes-from="service-sidekick" \
    --volume="/etc/kubernetes:/etc/kubernetes:z" \
    --volume="/etc/cni:/etc/cni:rw,z" \
    --volume="/opt/cni:/opt/cni:rw,z" \
    --volume="/var/lib/cni:/var/lib/cni:z" \
    --volume="/var/lib/calico:/var/lib/calico:z" \
    --volume="/etc/resolv.conf:/etc/resolv.conf" \
    --volume="/sys:/sys:rprivate" \
    --volume="/var/lib/docker:/var/lib/docker:rw,rslave,z" \
    --volume="/var/lib/kubelet:/var/lib/kubelet:shared,z" \
    --volume="/var/lib/rancher:/var/lib/rancher:shared,z" \
    --volume="/var/run:/var/run:rw,rprivate" \
    --volume="/run:/run:rprivate" \
    --volume="/etc/ceph:/etc/ceph" \
    --volume="/dev:/host/dev:rprivate" \
    --volume="/var/log/containers:/var/log/containers:z" \
    --volume="/var/log/pods:/var/log/pods:z" \
    --volume="/usr:/host/usr:ro" \
    --volume="/etc:/host/etc:ro" \
    --label "io.rancher.rke.container.name"="kubelet" \
    --label "org.label-schema.build-date"="2018-04-25T05:52:34Z" \
    --label "org.label-schema.vcs-ref"="1e8a3dc715cbc00a561e1d84645a6378302b7c8c" \
    --label "org.label-schema.vcs-url"="https://github.com/rancher/hyperkube.git" \
    --entrypoint=/opt/rke/entrypoint.sh \
    --log-driver="json-file" \
    --restart="no" \
    --detach=true \
    --privileged \
    "rancher/hyperkube:v1.10.1-rancher2" \
    "kubelet"  "--enforce-node-allocatable="  "--v=2"  "--cgroups-per-qos=True"  "--volume-plugin-dir=/var/lib/kubelet/volumeplugins"  "--root-dir=/var/lib/kubelet"  "--read-only-port=0"  "--cni-bin-dir=/opt/cni/bin"  "--network-plugin=cni"  "--cni-conf-dir=/etc/cni/net.d"  "--resolv-conf=/etc/resolv.conf"  "--cluster-domain=cluster.local"  "--hostname-override=jvb-control1"  "--cluster-dns=10.43.0.10"  "--address=0.0.0.0"  "--cadvisor-port=0"  "--client-ca-file=/etc/kubernetes/ssl/kube-ca.pem"  "--fail-swap-on=false"  "--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"  "--feature-gates=MountPropagation=false"  "--cloud-provider="  "--kubeconfig=/etc/kubernetes/ssl/kubecfg-kube-node.yaml"  "--pod-infra-container-image=rancher/pause-amd64:3.1"  "--allow-privileged=true"  "--anonymous-auth=false"
```

* Modify the command to change the name of the container and add this line.
```sh
"--authentication-token-webhook=true"
```

* It should look something like this.
```sh
docker run \
    --name=/kubelet \
    --env="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
    --network "host" \
     \
    --volumes-from="service-sidekick" \
    --volume="/etc/kubernetes:/etc/kubernetes:z" \
    --volume="/etc/cni:/etc/cni:rw,z" \
    --volume="/opt/cni:/opt/cni:rw,z" \
    --volume="/var/lib/cni:/var/lib/cni:z" \
    --volume="/var/lib/calico:/var/lib/calico:z" \
    --volume="/etc/resolv.conf:/etc/resolv.conf" \
    --volume="/sys:/sys:rprivate" \
    --volume="/var/lib/docker:/var/lib/docker:rw,rslave,z" \
    --volume="/var/lib/kubelet:/var/lib/kubelet:shared,z" \
    --volume="/var/lib/rancher:/var/lib/rancher:shared,z" \
    --volume="/var/run:/var/run:rw,rprivate" \
    --volume="/run:/run:rprivate" \
    --volume="/etc/ceph:/etc/ceph" \
    --volume="/dev:/host/dev:rprivate" \
    --volume="/var/log/containers:/var/log/containers:z" \
    --volume="/var/log/pods:/var/log/pods:z" \
    --volume="/usr:/host/usr:ro" \
    --volume="/etc:/host/etc:ro" \
    --label "io.rancher.rke.container.name"="kubelet" \
    --label "org.label-schema.build-date"="2018-04-25T05:52:34Z" \
    --label "org.label-schema.vcs-ref"="1e8a3dc715cbc00a561e1d84645a6378302b7c8c" \
    --label "org.label-schema.vcs-url"="https://github.com/rancher/hyperkube.git" \
    --entrypoint=/opt/rke/entrypoint.sh \
    --log-driver="json-file" \
    --restart="no" \
    --detach=true \
    --privileged \
    "rancher/hyperkube:v1.10.1-rancher2" \
    "kubelet"  "--enforce-node-allocatable="  "--v=2"  "--cgroups-per-qos=True"  "--volume-plugin-dir=/var/lib/kubelet/volumeplugins"  "--root-dir=/var/lib/kubelet"  "--read-only-port=0"  "--cni-bin-dir=/opt/cni/bin"  "--network-plugin=cni"  "--cni-conf-dir=/etc/cni/net.d"  "--resolv-conf=/etc/resolv.conf"  "--cluster-domain=cluster.local"  "--hostname-override=jvb-control1"  "--cluster-dns=10.43.0.10"  "--address=0.0.0.0"  "--cadvisor-port=0"  "--client-ca-file=/etc/kubernetes/ssl/kube-ca.pem"  "--fail-swap-on=false"  "--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"  "--feature-gates=MountPropagation=false"  "--cloud-provider="  "--kubeconfig=/etc/kubernetes/ssl/kubecfg-kube-node.yaml"  "--pod-infra-container-image=rancher/pause-amd64:3.1"  "--allow-privileged=true"  "--anonymous-auth=false" "--authentication-token-webhook=true"
```

* Run the new docker command
```sh
root@jvb-control1:~# docker run \
>     --name=/kubelet \
>     --env="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
>     --network "host" \
>      \
>     --volumes-from="service-sidekick" \
>     --volume="/etc/kubernetes:/etc/kubernetes:z" \
>     --volume="/etc/cni:/etc/cni:rw,z" \
>     --volume="/opt/cni:/opt/cni:rw,z" \
>     --volume="/var/lib/cni:/var/lib/cni:z" \
>     --volume="/var/lib/calico:/var/lib/calico:z" \
>     --volume="/etc/resolv.conf:/etc/resolv.conf" \
>     --volume="/sys:/sys:rprivate" \
>     --volume="/var/lib/docker:/var/lib/docker:rw,rslave,z" \
>     --volume="/var/lib/kubelet:/var/lib/kubelet:shared,z" \
>     --volume="/var/lib/rancher:/var/lib/rancher:shared,z" \
>     --volume="/var/run:/var/run:rw,rprivate" \
>     --volume="/run:/run:rprivate" \
>     --volume="/etc/ceph:/etc/ceph" \
>     --volume="/dev:/host/dev:rprivate" \
>     --volume="/var/log/containers:/var/log/containers:z" \
>     --volume="/var/log/pods:/var/log/pods:z" \
>     --volume="/usr:/host/usr:ro" \
>     --volume="/etc:/host/etc:ro" \
>     --label "io.rancher.rke.container.name"="kubelet" \
>     --label "org.label-schema.build-date"="2018-04-25T05:52:34Z" \
>     --label "org.label-schema.vcs-ref"="1e8a3dc715cbc00a561e1d84645a6378302b7c8c" \
>     --label "org.label-schema.vcs-url"="https://github.com/rancher/hyperkube.git" \
>     --entrypoint=/opt/rke/entrypoint.sh \
>     --log-driver="json-file" \
>     --restart="no" \
>     --detach=true \
>     --privileged \
>     "rancher/hyperkube:v1.10.1-rancher2" \
>     "kubelet"  "--enforce-node-allocatable="  "--v=2"  "--cgroups-per-qos=True"  "--volume-plugin-dir=/var/lib/kubelet/volumeplugins"  "--root-dir=/var/lib/kubelet"  "--read-only-port=0"  "--cni-bin-dir=/opt/cni/bin"  "--network-plugin=cni"  "--cni-conf-dir=/etc/cni/net.d"  "--resolv-conf=/etc/resolv.conf"  "--cluster-domain=cluster.local"  "--hostname-override=jvb-control1"  "--cluster-dns=10.43.0.10"  "--address=0.0.0.0"  "--cadvisor-port=0"  "--client-ca-file=/etc/kubernetes/ssl/kube-ca.pem"  "--fail-swap-on=false"  "--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"  "--feature-gates=MountPropagation=false"  "--cloud-provider="  "--kubeconfig=/etc/kubernetes/ssl/kubecfg-kube-node.yaml"  "--pod-infra-container-image=rancher/pause-amd64:3.1"  "--allow-privileged=true"  "--anonymous-auth=false" "--authentication-token-webhook=true"
62a2ecbc94fce561ad6ea49d47f81d72df9f74ce3b8544167a498ef918b4bf42

```
* Update the old container's restart policy so it will not start again.
```sh
root@jvb-control1:~# docker update kubeletold --restart="no"
kubeletold
```

## Validation

If you're still connected you may exit your ssh sessions.

* Clone the metrics-server repository
```sh
git clone -b rancher https://github.com/superseb/metrics-server
```

* Deploy it to your kubernetes cluster
```sh
[jason@vblinux metrics-server-on-rancher-2.0.2 ]$ rancher kubectl create -f metrics-server/deploy/1.8\+/

clusterrolebinding "metrics-server:system:auth-delegator" created
rolebinding "metrics-server-auth-reader" created
apiservice "v1beta1.metrics.k8s.io" created
serviceaccount "metrics-server" created
deployment "metrics-server" created
service "metrics-server" created
clusterrole "system:metrics-server" created
clusterrolebinding "system:metrics-server" created
```

* Check the logs
```sh
[jason@vblinux metrics-server-on-rancher-2.0.2 ]$ rancher kubectl logs -n kube-system -l k8s-app=metrics-server
I0605 19:39:20.463543       1 heapster.go:71] /metrics-server --source=kubernetes.summary_api:''?useServiceAccount=true&kubeletHttps=true&kubeletPort=10250&insecure=true
I0605 19:39:20.463791       1 heapster.go:72] Metrics Server version v0.2.1
I0605 19:39:20.464292       1 configs.go:61] Using Kubernetes client with master "https://10.43.0.1:443" and version 
I0605 19:39:20.464370       1 configs.go:62] Using kubelet port 10250
I0605 19:39:20.464925       1 heapster.go:128] Starting with Metric Sink
I0605 19:39:20.628845       1 serving.go:308] Generated self-signed cert (apiserver.local.config/certificates/apiserver.crt, apiserver.local.config/certificates/apiserver.key)
I0605 19:39:21.742134       1 heapster.go:101] Starting Heapster API server...
[restful] 2018/06/05 19:39:21 log.go:33: [restful/swagger] listing is available at https:///swaggerapi
[restful] 2018/06/05 19:39:21 log.go:33: [restful/swagger] https:///swaggerui/ is mapped to folder /swagger-ui/
I0605 19:39:21.744422       1 serve.go:85] Serving securely on 0.0.0.0:443
```

* Check the metrics
**Note** Make sure your kubectl is the latest version otherwise, you'll see some weird behavior consistent with this behavior
[https://github.com/kubernetes/kubernetes/issues/59438](https://github.com/kubernetes/kubernetes/issues/59438)

```sh
[jason@vblinux metrics-server-on-rancher-2.0.2 ]$ rancher kubectl top nodes
NAME          CPU(cores)   CPU%      MEMORY(bytes)   MEMORY%   
jvb-etcd1     51m          5%        583Mi           65%       
jvb-worker1   57m          5%        639Mi           71% 
```
