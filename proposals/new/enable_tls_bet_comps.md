Proposal to Enable TLS Between Harbor Components



# Proposal: `Enable TLS Between Components`

Author: `<Qian Deng / ninjadq>`

## Abstract

Https is more secure then http, So we propose to give user an option to enable TLS between every components of harbor to improve the security of Harbor.

## Background

Everyone knows that http is insecure. But currently Harbor components are still using http to communicate with each other. Even it is inside the container network. The potential problems are still exist. 

For example, some sompanies has policy to disallow their software use http for security concern. In addition, there are some advanced user who use harbor in a large scale enviroment in which harbor components do not deployed in one machine and they are not use docker-compose network. In this situation, some senstive infomation will expose to the intranet.

Besides the potention issue. Move http to https is also an industry trend. We can find so many articles about deprecate http.

## Proposal

[A precise statement of the proposed change.]

This PR propose to give user a option to use https when communicating between components if they all maintained by Harbor maintainer. Which are highlighted by red line.

![Communication digram of harbor components](/Users/dengq/Work/GitHub/goharbor/community/proposals/images/enable-tls-comp-communication.png)



Include these component pairs

* Core <----> Proxy
* Core <----> Job Service
* Core <----> RegistryCtl
* Core <----> Notary Server
* Core <----> Clair Adapter
* Job Service <----> RegistryCtl
* Job Service <----> Clair Adapter

To achieve this goal, we need to 

1. Provide certs and private key for each components

2. Provide the CA certs that signed the certs in 1

3. Let the application inside the container to trust CA provided by 2

4. Implement the logic in these components to use tls to communicate

5. Change the prepare script to deploy and mount these certs to correct place

6. Migrate script to handle config migrate

7. (Optional) Enable mutual TLS authentication

    

## Non-Goals

1. Manage certs for user
2. Generate cert or key for user
3. Enable TLS between Harbor with third party component like DB, Redis, Chartmuseum, etc.
4. Enable TLS between Portal and Proxy. Because Portal only contains static file.

## Rationale

Pros:

* More secure
* Possible to remove logic that use the secretkey to communicate

Cons:

* Performence lost
* More complicated to setup Harbor
* May cause failure when migrating from previous version if not conigured correctly

## Compatibility

1. Not compatible if you migrated from previous version and enable this feature. Because it need more cert files.

## Implementation

[A description of the steps in the implementation, who will do them, and when.]

### Provide an config option

Add an option in `harbor.yml` to provide cert files. https config item may look like this

```yaml
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /your/certificate/path
  private_key: /your/private/key/path

  internal:
    trust_ca_dir: /trust/ca/dir
    core_certificate: /your/core/certificate/path
    core_private_key: /your/core/private/key/path
    clair_adapter_certificate: /your/clair/adapter/certificate/path
    clair_adapter_private_key: /your/clair/adapter/private_key/path
    job_service_certificate: /your/job/service/certificate/path
    job_service_private_key: /your/job/service/private_key/path
    registry_ctl_certificate: /your/registry/ctl/certificate/path
    registry_ctl_private_key: /your/registry/ctl/private/key/path
```



### Load Security Related Files 

First of all we should load all the trust CAs into the container.

We can mount `/trust/ca/dir` to `/harbor_cust_cert` in container. The `install_cert.sh` will append them automaticlly.

But two things need concern,

* one is that let prepare failure early if file is illegal
* another one is remove `ca_bundle` in storage settings. Becaus it's duplicate with `trust_ca_dir` . And need to migrate `ca_bundle` if configured it



### Add the Logic to Enable SSL in Codes

#### Enable Server Side SSL

Most of components are written in Go. So we need enable ssl in the server side like this

```go
cer, err := tls.LoadX509KeyPair("server.crt", "server.key")
if err != nil {
    log.Println(err)
    return
}

config := &tls.Config{Certificates: []tls.Certificate{cer}}
ln, err := tls.Listen("tcp", ":443", config) 
if err != nil {
    log.Println(err)
    return
}
defer ln.Close()
```

If compoent is use Beego as web framework, we can set the beego config file like this

```json
beego.BConfig.Listen.EnableHTTPS = false
beego.BConfig.Listen.HTTPSCertFile = "conf/ssl.crt"
beego.BConfig.Listen.HTTPSKeyFile = "conf/ssl.key"
```

#### Enable Client Side SSL

If mutual ssl is required, we also need to enable client side ssl

Golang can enable it like this

```go
cert, err := tls.LoadX509KeyPair("client.crt", "client.key")
if err != nil {
        log.Fatal(err)
}

client := &http.Client{
        Transport: &http.Transport{
                TLSClientConfig: &tls.Config{
                        RootCAs:      caCertPool,
                        Certificates: []tls.Certificate{cert},
                },
        },
}
```

Nginx should add config like this

```nginx
server {
 ...
 ssl_verify_client on;
 ...
}
```



#### Make CI work After Enable SSL

We need generate certs and keys for each components and update ci configuration files.

