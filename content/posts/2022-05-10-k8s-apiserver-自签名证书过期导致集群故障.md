---
title: k8s apiserver 自签名证书过期导致集群故障
author: 阿辉
date: 2022-05-10T15:46:38+00:00
categories:
- Kubernetes
- ApiServer
tags:
- Kubernetes
- ApiServer
keywords:
- Kubernetes
- ApiServer
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---

{{< toc >}}


# 1. 故障处理过程

今天接到同事反馈发现有一套k8s apiserver集群出现如下报错：

```
Failed to create new replica set "recommend-alg-service-74c6bc97cd": Get https://10.13.96.12:6443/api/v1/namespaces/saas-ec-tomcat-pl/resourcequotas: x509: certificate has expired or is not yet valid
```

随后去api server节点上查询api server日志，发现也有大量报错：

```
I0510 17:43:56.889617  790992 reflector.go:211] Listing and watching *v1.MutatingWebhookConfiguration from k8s.io/client-go/informers/factory.go:135
I0510 17:43:56.892900  790992 log.go:172] http: TLS handshake error from 10.13.96.11:36528: remote error: tls: bad certificate
E0510 17:43:56.892945  790992 reflector.go:178] k8s.io/client-go/informers/factory.go:135: Failed to list *v1.MutatingWebhookConfiguration: Get https://10.13.96.11:6443/apis/admissionregistration.k8s.io/v1/mutatingwebhookconfigurations?resourceVersion=331916875: x509: certificate has expired or is not yet valid
I0510 17:43:57.087301  790992 reflector.go:211] Listing and watching *v1.ClusterRole from k8s.io/client-go/informers/factory.go:135
I0510 17:43:57.090560  790992 log.go:172] http: TLS handshake error from 10.13.96.11:36530: remote error: tls: bad certificate
E0510 17:43:57.090625  790992 reflector.go:178] k8s.io/client-go/informers/factory.go:135: Failed to list *v1.ClusterRole: Get https://10.13.96.11:6443/apis/rbac.authorization.k8s.io/v1/clusterroles?resourceVersion=397554978: x509: certificate has expired or is not yet valid
```

让人奇怪的是，并不是所有请求都报证书错误，通过日志发现，大量的请求到api server都是200的，出现500的为少数。

同时我们通过上面的报错发现，报错的访问来源是`10.13.96.11:36528`，这个IP是api server自己。

最后我们将错误定位到以下范围：

- api server自己访问自己报证书过期的错误，而其它组件访问都是正常的。

同时我们检查了服务器上的所有k8s 集群组件使用的证书，并没有过期。

在无计可施时，怀着试试看的想法，我们将api server服务重启了，发现重启后恢复正常。果然是重启能解决99%的问题。

<!--more-->

# 2. 故障原因排查

故障恢复后，我们一直在排查原因，排查原因时我们发现下面3点：

- api server服务正好启动了一年没有重启
- api server自己访问自己报证书过期的错误
- api server访问自己使用的是IP地址，然而我们api server对其它组件及外部都是使用域名来访问的

下面就只能看源码了，通过api server的源码，我们发现api server启动时会自动生成一个自签名证书：

k8s.io/apiserver/pkg/server/options/serving_with_loopback.go:
```golang
// ApplyTo fills up serving information in the server configuration.
func (s *SecureServingOptionsWithLoopback) ApplyTo(secureServingInfo **server.SecureServingInfo, loopbackClientConfig **rest.Config) error {
	if s == nil || s.SecureServingOptions == nil || secureServingInfo == nil {
		return nil
	}

	if err := s.SecureServingOptions.ApplyTo(secureServingInfo); err != nil {
		return err
	}

	if *secureServingInfo == nil || loopbackClientConfig == nil {
		return nil
	}

	// create self-signed cert+key with the fake server.LoopbackClientServerNameOverride and
	// let the server return it when the loopback client connects.
	certPem, keyPem, err := certutil.GenerateSelfSignedCertKey(server.LoopbackClientServerNameOverride, nil, nil)
	if err != nil {
		return fmt.Errorf("failed to generate self-signed certificate for loopback connection: %v", err)
	}
	certProvider, err := dynamiccertificates.NewStaticSNICertKeyContent("self-signed loopback", certPem, keyPem, server.LoopbackClientServerNameOverride)
	if err != nil {
		return fmt.Errorf("failed to generate self-signed certificate for loopback connection: %v", err)
	}

	secureLoopbackClientConfig, err := (*secureServingInfo).NewLoopbackClientConfig(uuid.New().String(), certPem)
	switch {
	// if we failed and there's no fallback loopback client config, we need to fail
	case err != nil && *loopbackClientConfig == nil:
		return err

	// if we failed, but we already have a fallback loopback client config (usually insecure), allow it
	case err != nil && *loopbackClientConfig != nil:

	default:
		*loopbackClientConfig = secureLoopbackClientConfig
		// Write to the front of SNICerts so that this overrides any other certs with the same name
		(*secureServingInfo).SNICerts = append([]dynamiccertificates.SNICertKeyContentProvider{certProvider}, (*secureServingInfo).SNICerts...)
	}

	return nil
}
```

而且这个自签名证书的时间为一年。

k8s.io/client-go/util/cert/cert.go:

```golang
// GenerateSelfSignedCertKey creates a self-signed certificate and key for the given host.
// Host may be an IP or a DNS name
// You may also specify additional subject alt names (either ip or dns names) for the certificate.
func GenerateSelfSignedCertKey(host string, alternateIPs []net.IP, alternateDNS []string) ([]byte, []byte, error) {
	return GenerateSelfSignedCertKeyWithFixtures(host, alternateIPs, alternateDNS, "")
}

// GenerateSelfSignedCertKeyWithFixtures creates a self-signed certificate and key for the given host.
// Host may be an IP or a DNS name. You may also specify additional subject alt names (either ip or dns names)
// for the certificate.
//
// If fixtureDirectory is non-empty, it is a directory path which can contain pre-generated certs. The format is:
// <host>_<ip>-<ip>_<alternateDNS>-<alternateDNS>.crt
// <host>_<ip>-<ip>_<alternateDNS>-<alternateDNS>.key
// Certs/keys not existing in that directory are created.
func GenerateSelfSignedCertKeyWithFixtures(host string, alternateIPs []net.IP, alternateDNS []string, fixtureDirectory string) ([]byte, []byte, error) {
	validFrom := time.Now().Add(-time.Hour) // valid an hour earlier to avoid flakes due to clock skew
	maxAge := time.Hour * 24 * 365          // one year self-signed certs

	baseName := fmt.Sprintf("%s_%s_%s", host, strings.Join(ipsToStrings(alternateIPs), "-"), strings.Join(alternateDNS, "-"))
	certFixturePath := path.Join(fixtureDirectory, baseName+".crt")
	keyFixturePath := path.Join(fixtureDirectory, baseName+".key")
	if len(fixtureDirectory) > 0 {
		cert, err := ioutil.ReadFile(certFixturePath)
		if err == nil {
			key, err := ioutil.ReadFile(keyFixturePath)
			if err == nil {
				return cert, key, nil
			}
			return nil, nil, fmt.Errorf("cert %s can be read, but key %s cannot: %v", certFixturePath, keyFixturePath, err)
		}
		maxAge = 100 * time.Hour * 24 * 365 // 100 years fixtures
	}

	caKey, err := rsa.GenerateKey(cryptorand.Reader, 2048)
	if err != nil {
		return nil, nil, err
	}
```


可以用如下方法查看自签名证书：`curl --resolve apiserver-loopback-client:6443:{Master_IP}-k -v https://apiserver-loopback-client:6443/healthz`，如：

```
 admin@deMBP16  ~/Downloads  curl --resolve apiserver-loopback-client:6443:10.13.96.11 -k -v https://apiserver-loopback-client:6443/healthz
* Added apiserver-loopback-client:6443:10.13.96.11 to DNS cache
* Hostname apiserver-loopback-client was found in DNS cache
*   Trying 10.13.96.11:6443...
* Connected to apiserver-loopback-client (10.13.96.11) port 6443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (OUT), TLS handshake, Client hello (1):
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Request CERT (13):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Certificate (11):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=apiserver-loopback-client@1652180293
*  start date: May 10 09:58:13 2022 GMT
*  expire date: May 10 09:58:13 2023 GMT
*  issuer: CN=apiserver-loopback-client-ca@1652180293
*  SSL certificate verify result: self signed certificate in certificate chain (19), continuing anyway.
* Using HTTP2, server supports multiplexing
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x137014e00)
> GET /healthz HTTP/2
> Host: apiserver-loopback-client:6443
> user-agent: curl/7.79.1
> accept: */*
>
* Connection state changed (MAX_CONCURRENT_STREAMS == 250)!
< HTTP/2 401
< cache-control: no-cache, private
< content-type: application/json
< content-length: 165
< date: Tue, 10 May 2022 16:03:17 GMT
<
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
* Connection #0 to host apiserver-loopback-client left intact
}%
```

# 3. 故障结论

故障原因很明显：api server启动时会自动生成一个自签名证书，而且这个自签名证书的时间为一年。

故障处理方法：重启api server（建议在一年内重启）

github上有人提过issue，官方以k8s本来就应该一年内升级且会重启为由，不认为这是一个问题：https://github.com/kubernetes/kubernetes/issues/86552

