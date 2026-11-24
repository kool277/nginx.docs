---
f5-product: Miscellaneous
f5-files:
- content/nginx/admin-guide/installing-nginx/installing-nginx-docker.md
- content/nic/install/images/registry-download.md
---

In order to obtain a container image, you will need the JSON Web Token file or SSL certificate and private key files provided with your NGINX Plus subscription. 

These files grant access to the F5 Container Registry, `private-registry.nginx.com`, which has the images. To download the files:

{{< tabs name="product_keys" >}}

{{% tab name="JSON Web Token" %}}

{{< include "licensing-and-reporting/download-jwt-from-myf5.md" >}}

{{% /tab %}}

{{% tab name="SSL" %}}

{{< include "licensing-and-reporting/download-certificates-from-myf5.md" >}}

{{% /tab %}}

{{< /tabs >}}