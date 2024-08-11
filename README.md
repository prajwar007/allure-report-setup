# ALLURE-DOCKER-SERVICE KUSTOMIZE EXAMPLE

## USAGE
These files would typically be in a `/base` directory that is patched for a specific environment.  

For example in a directory structure like this where `base` is the dirctory containing this readme.:

```bash
├── base
│   ├── api
│   ├── ui
│   ├── tls-secret.yaml
│   ├── allure-ingress-service-node-port.yaml
│   ├── kustomization.yaml 
│   └── namespace.yaml
│  
└── environments
    ├── dev
    │   ├── config
    │   ├── kustomization.yaml
    │   └── patches
    └── prod
        ├── config
        ├── kustomization.yaml
        └── patches
```

Then in an environments kustomization.yaml:  

`/environments/dev/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  # your other env specific resources here

patches:
  #- path: patches/api-secret.yaml
  # your other patches here

```
### Creating a SSL certificate and key
If you have an existing certificate/key ready to use you can skip this section.

- Create a certificate/key
For this example, we are going to create a certificate/key for domain `allure-report.local.com`
```sh
openssl req -x509 -newkey rsa:4096 -sha256 -nodes -keyout tls.key -out tls.crt -subj "/CN=allure-report.local.com"
```
Note: If you use `Git Bash` use double slashes at the beginning `-subj "//CN=allure-report.local.com"`


Output
```sh
Generating a 4096 bit RSA private key
......++
......................................................................................................................................................++
writing new private key to 'tls.key'
-----
```

### Creating an Ingress
- As pre-requisite you need to have an Ingress Controller to be able to use ingress. 
For example, you can install `ingress-nginx` --> https://kubernetes.github.io/ingress-nginx/deploy/
If you are testing in a `windows` run the yaml for Mac https://kubernetes.github.io/ingress-nginx/deploy/#docker-for-mac


- If you are using kubernetes locally, as you don't have a DNS you can edit the file `/etc/hosts`
```sh
sudo vi /etc/hosts
```
Note: If you use `Windows` the path is this one `C:\Windows\System32\drivers\etc\hosts`

then add the next line at the end
```sh
127.0.0.1   allure-report.local.com
```
On that way, when you request `allure-report.local.com` you will be re-directed to `localhost`

- Remove `TLS` section in the ingress yaml definition in case you don't want to use `https` protocol.

Reference:
- https://kubernetes.io/docs/concepts/services-networking/ingress/

### Create all allure objects via kustomization
```sh
kubectl apply -k ./environments/dev/
```

#### Check ingress resource
- Get ingress
```sh
kubectl get ingress --namespace allure
```
Output:
```sh
NAME                               HOSTS           ADDRESS     PORTS     AGE
allure-ingress-service-node-port   allure-report.local.com   localhost   80, 443   8s
```

- Check if ingress for API is working
```sh
curl https://allure-report.local.com/allure-api -ik
```
Output:
```sh
HTTP/2 200
date: Sun, 11 Aug 2024 10:18:02 GMT
content-type: text/html; charset=utf-8
content-length: 1581
access-control-allow-credentials: true
strict-transport-security: max-age=31536000; includeSubDomains

<!-- HTML for static distribution bundle build -->
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <title>Allure Docker Service</title>
  ...
</head>

<body>
...
</body>
</html>%
```

- Check if ingress for UI is working
```sh
curl https://allure-report.local.com/allure-ui/allure-docker-service-ui -ik
```
Output:
```sh
HTTP/2 200 
date: Sun, 11 Aug 2024 10:18:47 GMT
content-type: text/html; charset=UTF-8
content-length: 2055
x-powered-by: Express
accept-ranges: bytes
cache-control: public, max-age=0
last-modified: Mon, 18 Jan 2021 21:01:25 GMT
etag: W/"807-177174d6888"
strict-transport-security: max-age=31536000; includeSubDomains

<!doctype html><html lang="en"><head><base href="/"/><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><link rel="shortcut icon" href="./favicon.ico"><title>Allure Docker 
Service UI</title><script src="./env-config.js"></script><link href="./static/css/main.a16b884e.chunk.css" rel="stylesheet"></head><body>...</body></html>%
```

### Using Application Deployed
You can start using `Allure Docker Service` in Kubernetes like this:
```sh
curl https://allure-report.local.com/allure-api/version -ik
```
Output:
```sh
HTTP/2 200
date: Sun, 11 Aug 2024 10:20:08 GMT
content-type: application/json
content-length: 86
access-control-allow-credentials: true
strict-transport-security: max-age=31536000; includeSubDomains

{"data":{"version":"2.27.0"},"meta_data":{"message":"Version successfully obtained"}}
```
Check the Swagger Documentation in a browser with the url: https://allure-report.local.com/allure-api

Check scripts https://github.com/fescobar/allure-docker-service/tree/beta#send-results-through-api where the `allure server url` would be `https://allure-report.local.com` and the prefix `/allure-api`


Also, you can start using `Allure Docker Service UI` opening a browser this url: https://allure-report.local.com/allure-ui/allure-docker-service-ui


### Removing all allure objects created via kustomization
```sh
kubectl delete -k ./environments/dev/
```