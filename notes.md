startIstio Ingress Gateway
In this lab, you will deploy a sample application to your Kubernetes cluster, expose the web-api service to the Istio ingress gateway, and configure secure access to the service. The ingress gateway allows traffic into the mesh. If you need more sophisticated edge gateway capabilities (such as request transformation, OIDC, LDAP, OPA, etc.) then you should use a gateway specifically built for those use cases like Gloo Edge.
Prerequisites
Verify you're in the correct folder for this lab: /root/istio-workshops/istio-basics. This lab builds on the first lab where you installed Istio and its add-on components using the demo profile.
shell
copy
cd /root/istio-workshops/istio-basics
Deploy the sample application
You will use the web-api, recommendation, and purchase-history services built using the fake service as your sample application. The web-api service calls the recommendation service via HTTP, and the recommendation service calls the purchase-history service, also via HTTP.
	1. Set up the istioinaction namespace for our services:
shell
copy
kubectl create ns istioinaction
	2. Deploy the web-api, recommendation and purchase-history services along with the sleep service into the istioinaction namespace:
shell
copy
kubectl apply -n istioinaction -f sample-apps/web-api.yaml
kubectl apply -n istioinaction -f sample-apps/recommendation.yaml
kubectl apply -n istioinaction -f sample-apps/purchase-history-v1.yaml
kubectl apply -n istioinaction -f sample-apps/sleep.yaml
	3. After running these commands, you should check that all pods are running in the istioinaction namespace:
shell
copy
kubectl get po -n istioinaction
Wait a few seconds until all of them show a Running status.
Configure the inbound traffic
	1. The Istio ingress gateway will create a Kubernetes Service of type LoadBalancer. Use this GATEWAY_IP address to reach the gateway:
shell
copy
kubectl get svc -n istio-system
	2. Store the ingress gateway IP address in an environment variable.
shell
copy
export GATEWAY_IP=$(kubectl get svc -n istio-system istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
export INGRESS_PORT=80
export SECURE_INGRESS_PORT=443
Expose our apps

Even though you don't have apps defined in the istioinaction namespace in the mesh yet, you can still use the Istio ingress gateway to route traffic to them. Using Istio's Gateway resource, you can configure what ports should be exposed, what protocol to use, etc. Using Istio's VirtualService resource, you can configure how to route traffic from the Istio ingress gateway to your web-api service.
	1. Review the Gateway resource:
shell
copy
cat sample-apps/ingress/web-api-gw.yaml
	2. Review the VirtualService resource:
shell
copy
cat sample-apps/ingress/web-api-gw-vs.yaml
Why is port number 8080 shown in the destination route configuration for the web-api-gw-vs VirtualService resource? Check the service port for the web-api service in the istioinaction namespace:
shell
copy
kubectl get service web-api -n istioinaction
You can see the service listens on port 8080.
	3. Apply the Gateway and VirtualService resources to expose your web-api service outside of the Kubernetes cluster:
shell
copy
kubectl -n istioinaction apply -f sample-apps/ingress/
	4. The Istio ingress gateway will create new routes on the proxy that you should be able to call from outside of the Kubernetes cluster:
shell
copy
curl -H "Host: istioinaction.io" http://$GATEWAY_IP:$INGRESS_PORT
	5. Query the gateway configuration using the istioctl proxy-config command:
shell
copy
istioctl proxy-config routes deploy/istio-ingressgateway.istio-system
	6. If you want to see an individual route, you can ask for its output as json like this:
shell
copy
istioctl proxy-config routes deploy/istio-ingressgateway.istio-system --name http.8080 -o json
Secure the inbound traffic

To secure inbound traffic with HTTPS, you need a certificate with the appropriate SAN and you will need to configure the Istio ingress-gateway to use it.
	1. Create a TLS secret for istioinaction.io in the istio-system namespace:
shell
copy
kubectl create -n istio-system secret tls istioinaction-cert --key labs/02/certs/istioinaction.io.key --cert labs/02/certs/istioinaction.io.crt
	2. Update the Istio ingress-gateway to use this cert:
shell
copy
cat labs/02/web-api-gw-https.yaml
Note, we are pointing to the istioinaction-cert and that the cert must be in the same namespace as the ingress gateway deployment. Even though the Gateway resource is in the istioinaction namespace, the cert must be where the gateway is actually deployed.
	3. Apply the web-api-gw-https.yaml in the istioinaction namespace. Since this gateway resource is also called web-api-gateway, it will replace our prior web-api-gateway configuration for port 80.
shell
copy
kubectl -n istioinaction apply -f labs/02/web-api-gw-https.yaml
	4. Call the web-api service through the Istio ingress-gateway on the secure 443 port:
shell
copy
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
	5. If you call it on the 80 port with http, it will not work as you no longer have the gateway resource configured to be exposed on port 80.
shell
copy
curl -H "Host: istioinaction.io" http://$GATEWAY_IP:$INGRESS_PORT
Mesh Metrics
Now that we have an Istio gateway running and receiving traffic, we can now visualize information about the traffic we just generated.
Click on the Grafana UI tab and we should start to see some mesh metrics appearing.
Next lab
Congratulations, you have exposed the web-api service to Istio ingress gateway securely. You'll explore adding services to the mesh in the next lab.

From <https://play.instruqt.com/embed/soloio/tracks/get-started-istio/challenges/istio-ingress-gateway/assignment> 



Install Istio
One of the quickest ways to get started with Istio is to leverage the demo profile. The demo profile is designed to showcase Istio functionality with modest resource requirements. The demo profile contains an Istio control plane (also called Istiod), Istio ingress-gateway and egress-gateway, and a few add-on components. In this lab, you will install Istio with the demo profile. You will validate the installation is successful and examine the installation artifacts. While you can use either istioctl, Helm, or the Istio operator to install Istio, in this lab you will use istioctl.
Note: If you open the Prometheus/Grafana/Jaeger/Kiali UI tab, you will see the "Please wait, we are trying to connect to the service..." message. This is normal before these services are installed.
Download Istio
	1. Download the Istio release binary:
shell
copy
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} sh -
	2. Add the istioctl client to the PATH:
shell
copy
export PATH=$PWD/istio-${ISTIO_VERSION}/bin:$PATH
	3. Check istioctl version:
shell
copy
istioctl version
	4. Check if your Kubernetes environment meets Istio's platform requirement:
shell
copy
istioctl x precheck
The precheck response should indicate that no issues were found:
shell
copy
✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
To get started, check out https://istio.io/latest/docs/setup/getting-started/
Install Istio
	1. List available installation profiles:
shell
copy
istioctl profile list
	2. Since this is a getting started workshop, you will use the demo profile to install Istio.
shell
copy
istioctl install --set profile=demo -y
	3. You should see output that indicates each Istio component is installed successfully. Check out the resources installed by Istio:
shell
copy
kubectl get all,cm,secrets,envoyfilters -n istio-system
	4. Check out Custom Resource Definitions (CRDs) installed by Istio:
shell
copy
kubectl get crds -n istio-system
	5. Verify the installation using the following command:
shell
copy
istioctl verify-install
You should see the following at the end of the output to indicate that your Istio is installed successfully:
shell
copy
✔ Istio is installed and verified successfully
Install Istio Telemetry Add-ons
	1. Istio telemetry add-ons are shipped as samples, but these add-ons are optimized for quick getting started and demo purposes and not for production usage. They provides a convenient way to install telemetry components that integrate with Istio.
shell
copy
kubectl apply -f istio-${ISTIO_VERSION}/samples/addons
	2. Wait till all pods in the istio-system are running:
shell
copy
kubectl get pods -n istio-system
	3. Enable access to the Prometheus dashboard:
shell
copy
istioctl dashboard prometheus --browser=false --address 0.0.0.0
	4. Click on the Prometheus UI tab, you should be able to view the Prometheus UI from there. Press ctrl+C to end the prior command, and use the command below to enable access to the Grafana dashboard:
shell
copy
istioctl dashboard grafana --browser=false --address 0.0.0.0
	5. Click on the Grafana UI tab, and you should be able to view the Grafana UI. Press ctrl+C to end the prior command, and use the command below to enable access to the Jaeger dashboard:
shell
copy
istioctl dashboard jaeger --browser=false --address 0.0.0.0
	6. Click on the Jaeger UI tab, and you should be able to view the Jaeger UI. Press ctrl+C to end the prior command, and use the command below to enable access to the Kiali dashboard:
shell
copy
istioctl dashboard kiali --browser=false --address 0.0.0.0
Click on the Kiali UI tab, and you should be able to view the Kiali UI. Press ctrl+C to end the prior command. You will not see much telemetry data on any of these dashboards now, as you don't have any services defined in the Istio service mesh yet. You will revisit these dashboards soon.
Next lab
Congratulations, you have installed the Istio control plane (Istiod), Istio ingress-gateway and egress-gateway, and its add-on components successfully. We'll learn how to expose your services to an Istio ingress gateway securely in the next lab.

From <https://play.instruqt.com/embed/soloio/tracks/get-started-istio/challenges/install-istio/assignment> 




Adding Services to the Mesh
In this lab, you will incrementally add services to the mesh. The mesh is actually integrated with the services themselves which makes it mostly transparent to the service implementation.
Sidecar injection
Adding services to the mesh requires that the client-side proxies be associated with the service components and registered with the control plane. With Istio, you have two methods to inject the Envoy Proxy sidecar into the microservice Kubernetes pods:
	• Automatic sidecar injection
	• Manual sidecar injection.
	1. To enable the automatic sidecar injection, use the command below to add the istio-injection label to the istioinaction namespace:
shell
copy
kubectl label namespace istioinaction istio-injection=enabled
	2. Validate the istioinaction namespace is annotated with the istio-injection label:
shell
copy
kubectl get namespace -L istio-injection
Now that you have an istioinaction namespace with automatic sidecar injection enabled, you are ready to start adding services in your istioinaction namespace to the mesh. Since you added the istio-injection label to the istioinaction namespace, the Istio mutating admission controller automatically injects the Envoy Proxy sidecar during the initial deployment or restart of the pod.
Review Service requirements
Before you add Kubernetes services to the mesh, you need to be aware of the application requirements to ensure that your Kubernetes services meet the minimum requirements.
Service descriptors:
	• each service port name must start with the protocol name, for example name: http
Deployment descriptors:
	• The pods must be associated with a Kubernetes service.
	• The pods must not run as a user with UID 1337
	• App and version labels are added to provide contextual information for metrics and tracing
Check the above requirements for each of the Kubernetes services and make adjustments as necessary. If you don't have NET_ADMIN security rights, you would need to explore the Istio CNI plugin to remove the NET_ADMIN requirement for deploying services.
Using the web-api service as an example, you can review its service and deployment descriptor:
shell
copy
cat sample-apps/web-api.yaml
From the service descriptor, the name: http declares the http protocol for the service port 8080:
yaml
copy
  - name: http
    protocol: TCP
    port: 8080
    targetPort: 8081
From the deployment descriptor, the app: web-api label matches the web-api service's selector of app: web-api so this deployment and its pod are associated with the web-api service. Further, the app: web-api label and version: v1 labels provide contextual information for metrics and tracing. The containerPort: 8081 declares the listening port for the container, which matches the targetPort: 8081 in the web-api service descriptor earlier.
yaml
copy
  template:
    metadata:
      labels:
        app: web-api
        version: v1
      annotations:
    spec:
      serviceAccountName: web-api
      containers:
      - name: web-api
        image: nicholasjackson/fake-service:v0.7.8
        ports:
        - containerPort: 8081
Check the purchase-history-v1, recommendation, and sleep services to validate they all meet the above requirements.
Adding services to the mesh
	1. You can add a sidecar to each of the services in the istioinaction namespace, starting with the web-api service:
shell
copy
kubectl rollout restart deployment web-api -n istioinaction
	2. Validate the web-api pod is running with Istio's default sidecar proxy injected:
shell
copy
kubectl get pod -l app=web-api -n istioinaction
You should see 2/2 in the output. This indicates the sidecar proxy is running alongside the web-api application container in the web-api pod:
copy
NAME                       READY   STATUS    RESTARTS   AGE
web-api-7d5ccfd7b4-m7lkj   2/2     Running   0          9m4s
	3. Validate the web-api pod log looks good:
shell
copy
kubectl logs deploy/web-api -c web-api -n istioinaction
	4. Validate you can continue to call the web-api service securely:
shell
copy
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
Understand what happens
Use the command below to get the details of the web-api pod:
shell
copy
kubectl get pod -l app=web-api -n istioinaction -o yaml
From the output, the web-api pod contains 1 init container and 2 normal containers. The Istio mutating admission controller was responsible for injecting the istio-init container and the istio-proxy container.
The istio-init container:
The istio-init container uses the proxyv2 image. The entry point of the container is pilot-agent, which contains the istio-iptables command to set up port forwarding for Istio's sidecar proxy.
copy
    initContainers:
- args:
      - istio-iptables
      - -p
      - "15001"
      - -z
      - "15006"
      - -u
      - "1337"
      - -m
      - REDIRECT
      - -i
      - '*'
      - -x
      - ""
      - -b
      - '*'
      - -d
      - 15090,15021,15020
      image: docker.io/istio/proxyv2:latest
      imagePullPolicy: Always
      name: istio-init
Interested in knowing more about the flags for istio-iptables? Run the following command:
shell
copy
kubectl exec deploy/web-api -c istio-proxy -n istioinaction -- /usr/local/bin/pilot-agent istio-iptables --help
The output explains the flags such as -u, -m, and -i which are used in the istio-init container's args. You will notice that all inbound ports are redirected to the Envoy Proxy container within the pod. You can also see a few ports such as 15021 which are excluded from redirection (you'll soon learn why this is the case). You may also notice the following securityContext for the istio-init container. This means that a service deployer must have the NET_ADMIN and NET_RAW security capabilities to run the istio-init container for the web-api service or other services in the Istio service mesh. If the service deployer can't have these security capabilities, you can use the Istio CNI plugin which removes the NET_ADMIN and NET_RAW requirement for users deploying pods into Istio service mesh.
The istio-proxy container:
When you continue looking through the list of containers in the pod, you will see the istio-proxy container. The istio-proxy container also uses the proxyv2 image. You'll notice the istio-proxy container has requested 0.01 CPU and 40 MB memory to start with as well as 2 CPU and 1 GB memory for limits. You will need to budget for these settings when managing the capacity for the cluster. These resources can be customized during the Istio installation thus may vary per your installation profile.
When you reviewed the istio-init container configuration earlier, you may have noticed that ports 15021, 15090, and 15020 are on the list of inbound ports to be excluded from redirection to Envoy. The reason is that port 15021 is for health check, and port 15090 is for the Envoy Proxy to emit its metrics to Prometheus, and port 15020 is for the merged Prometheus metrics colllected from the Istio agent, the Envoy Proxy, and the application container. Thus it is not necessary to redirect inbound traffic for these ports since they are being used by the istio-proxy container.
Also notice that the istiod-ca-cert and istio-token volumes are mounted on the pod for the purpose of implementing mutual TLS (mTLS), which will be covered in the lab 04.
Add more services to the Istio service mesh
	1. Next, you can add the istio-proxy sidecar to the other services in the istioinaction namespace
shell
copy
kubectl rollout restart deployment purchase-history-v1 -n istioinaction
kubectl rollout restart deployment recommendation -n istioinaction
kubectl rollout restart deployment sleep -n istioinaction
	2. Validate that all the pods in the istioinaction namespace are running with Istio's default sidecar proxy injected:
shell
copy
kubectl get pods -n istioinaction
	3. Validate that you can continue to call the web-api service securely:
shell
copy
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
What have you gained?
Congratulations on adding services in the istioinaction namespace to the Istio service mesh. One of the values of using a service mesh is that you will gain immediate insight into the behavior and interactions between your services. Istio delivers a set of dashboards as add-on components that give you access to important telemetry data, just by adding services to the mesh.
Distributed tracing
You can view distributed tracing information using the Jaeger UI tab.
	• Navigate to the Jaeger UI tab. On the "Service" dropdown, select "istio-ingressgateway". Click on the "Find Traces" button at the bottom. You should see some traces, which show every request to the web-api service through the Istio's ingress gateway.
	• Click on one of the traces to view the details of the distributed traces for that request. You can click on each trace span to learn more about it. You may notice all trace spans have the same value for the x-request-id header. Why? This is how Jaeger knows these trace spans are part of the same request. In order for your services' distributed tracing to work properly in your Istio service mesh, the B-3 trace headers including x-request-id have to be propagated between your services.
Next lab
Congratulations, you have added the sample application successfully to your Istio service mesh and observed the services' communications. You saw the secure communications between your services and saw how much time each request took. We'll explore securing these services with declarative policies in the Istio service mesh in the next lab.

From <https://play.instruqt.com/embed/soloio/tracks/get-started-istio/challenges/adding-services-to-mesh/assignment> 


Securing Communication Within Istio
In the previous lab, you explored adding services into a mesh. When you installed Istio using the demo profile, it is using a permissive security mode. The Istio permissive security setting is useful when you have services that are being moved into the service mesh incrementally, as it allows both plain text and mTLS traffic. In this lab, you will explore how Istio manages secure communication between services and how to require strict security between services in the sample application.
Prerequisites
Verify you're in the correct folder for this lab: /root/istio-workshops/istio-basics. This lab builds on earlier work where you added your services to the mesh.
shell
copy
cd /root/istio-workshops/istio-basics
Permissive mode
By default, Istio automatically upgrades the connection securely from the source service's sidecar proxy to the target service's sidecar proxy. This is why you saw the lock icon in the Kiali graph earlier in the connections between the Istio ingress gateway, the web-api service, the history service, and the recommendation service. While it is probably ok when onboarding your services to your Istio service mesh to have communications between source and target services allowed via plain text if mutual TLS communication fails, you wouldn't want this in a production environment. You need to have a proper security policy in place.
Check if you have a peerauthentication policy in any of your namespaces:
shell
copy
kubectl get peerauthentication --all-namespaces
You should see No resources found in the output, which means no peer authentication has been specified and the default PERMISSIVE mTLS mode is being used.
Enable strict mTLS
	1. You can lock down access to the services in your Istio service mesh to securely require mTLS using a peer authentication policy. Execute this command to define a default policy for the istio-system namespace that updates all of the servers to accept only mTLS traffic:
shell
copy
kubectl apply -n istio-system -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
	2. Verify your peerauthentication policy is installed:
shell
copy
kubectl get peerauthentication --all-namespaces
You should see the the default peerauthentication policy installed in the istio-system namespace with STRICT mTLS enabled:
copy
NAMESPACE      NAME      MODE     AGE
istio-system   default   STRICT   84s
Because the istio-system namespace is also the Istio mesh configuration root namespace in your environment, this peerauthentication policy is the default policy for all of your services in the mesh regardless of which namespaces your services run.
	3. You can send some traffic to web-api from a pod that is not part of the Istio service mesh. Deploy the sleep service and pod in the default namespace:
shell
copy
kubectl apply -n default -f sample-apps/sleep.yaml
	4. Access the web-api service from the sleep pod in the default namespace:
shell
copy
kubectl exec deploy/sleep -n default -- curl http://web-api.istioinaction:8080/
The request will fail because the web-api service can only be accessed with mutual TLS connections. The sleep pod in the default namespace doesn't have the sidecar proxy, so it doesn't have the needed keys and certificates to communicate to the web-api service via mutual TLS.
	5. Run the same command from the sleep pod in the istioinaction namespace:
shell
copy
kubectl exec deploy/sleep -n istioinaction -- curl http://web-api.istioinaction:8080/
You should now see the request succeed.
Visualize mTLS enforcement in Kiali
You can visualize the services in the mesh in Kiali.
	1. Generate some load to the data plane (by calling our web-api service) so that you can observe interactions among your services:
copy
for i in {1..200};
do curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT  --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP;
  sleep 3;
done
	2. Navigate to the Kiali tab and select the Graph tab.
On the "Namespace" dropdown, select "istioinaction". On the "Display" drop down, select "Traffic Animation" and "Security".
	3. You should observe the service interaction graph with some traffic animation and security badges.
	4. Return to the Terminal tab and enter ctrl+c to end the load generation.
Understand how mTLS works in Istio service mesh
	1. Inspect the key and/or certificates used by Istio for the web-api service in the istioinaction namespace:
shell
copy
istioctl proxy-config secret deploy/web-api -n istioinaction
From the output, you'll see the default secret and your Istio service mesh's root CA public certificate.
The default secret containers the public certificate information for the web-api service. You can analyze the contents of the default secret using openssl.
	2. Check the issuer of the public certificate:
shell
copy
istioctl proxy-config secret deploy/web-api -n istioinaction -o json | jq '[.dynamicActiveSecrets[] | select(.name == "default")][0].secret.tlsCertificate.certificateChain.inlineBytes' -r | base64 -d | openssl x509 -noout -text | grep 'Issuer'
	3. Check if the public certificate in the default secret is valid:
shell
copy
istioctl proxy-config secret deploy/web-api -n istioinaction -o json | jq '[.dynamicActiveSecrets[] | select(.name == "default")][0].secret.tlsCertificate.certificateChain.inlineBytes' -r | base64 -d | openssl x509 -noout -text | grep 'Validity' -A 2
You should see the public certificate is valid and expires in 24 hours.
	4. Validate the identity of the client certificate is correct:
shell
copy
istioctl proxy-config secret deploy/web-api -n istioinaction -o json | jq '[.dynamicActiveSecrets[] | select(.name == "default")][0].secret.tlsCertificate.certificateChain.inlineBytes' -r | base64 -d | openssl x509 -noout -text | grep 'Subject Alternative Name' -A 1
You should see the identity of the web-api service. Note it is using the SPIFFE format, e.g. spiffe://{my-trust-domain}/ns/{namespace}/sa/{service-account}.
Understand the SPIFFE format used by Istio
Where do the cluster.local and web-api values come from? Check the istio configmap in the istio-system namespace:
shell
copy
kubectl get cm istio -n istio-system -o yaml | grep trustDomain -m 1
You'll see cluster.local returned as the trustDomain value, per the installation of your Istio using the demo profile.
copy
    trustDomain: cluster.local
If you review the sample-apps/web-api.yaml file, you will see the web-api service account in there.
shell
copy
cat sample-apps/web-api.yaml | grep ServiceAccount -A 3
copy
kind: ServiceAccount
metadata:
  name: web-api
---
How does the web-api service obtain the needed key and/or certificates?
Earlier, you reviewed the injected istio-proxy container for the web-api pod. Recall there are a few volumes mounted to the istio-proxy container.
copy
      volumeMounts`
- mountPath: /var/run/secrets/istio
        name: istiod-ca-cert
      - mountPath: /var/lib/istio/data
        name: istio-data
      - mountPath: /etc/istio/proxy
        name: istio-envoy
      - mountPath: /var/run/secrets/tokens
        name: istio-token
      - mountPath: /etc/istio/pod
        name: istio-podinfo
      - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
        name: web-api-token-ztk5d
        readOnly: true
...
    - name: istio-token
      projected:
        defaultMode: 420
        sources:
        - serviceAccountToken:
            audience: istio-ca
            expirationSeconds: 43200
            path: istio-token
    - configMap:
        defaultMode: 420
        name: istio-ca-root-cert
      name: istiod-ca-cert
    - name: web-api-token-ztk5d
      secret:
        defaultMode: 420
        secretName: web-api-token-ztk5d
The istio-ca-cert mounted volume is from the istio-ca-root-cert configmap in the istioinaction namespace. During start up time, Istio agent (also called pilot-agent) creates the private key for the web-api service and then sends the certificate signing request (CSR) for the Istio CA (Istio control plane is the Istio CA in your installation) to sign the private key, using the istio-token and the web-api service's service account token web-api-token-ztk5d as credentials. The Istio agent sends the certificates received from the Istio CA along with the private key to the Envoy Proxy via the Envoy SDS API.
You noticed earlier that the certificate expires in 24 hours. What happens when the certificate expires? The Istio agent monitors the web-api certificate for expiration and repeats the CSR request process described above periodically to ensure each of your workload's certificate is still valid.
How is mTLS strict enforced?
When mTLS strict is enabled, you will find the Envoy Proxy configuration for the istio-proxy container actually has fewer lines of configurations. This is because when mTLS strict is enabled, you would only allow mTLS traffic and therefore don't need or want filter chain configurations to allow plain text traffic to services in the mesh (hint: search for "transport_protocol": "raw_buffer" in your Envoy configuration when PERMISSIVE mode is applied). If you are curious to explore Envoy configuration for any of your pod, you can use the command below to view the configuration for the istio-proxy container of the web-api pod.
shell
copy
istioctl proxy-config all deploy/web-api -n istioinaction -o json
Question
If you have only deployed a few services, why is there so much Envoy configuration detail for the pod? The reason is that Istio listens to everything in your Kubernetes cluster by default. You can explore using discovery selectors, and exportTo and Sidecar resources for operating Istio in a production environment in the Istio Essentials and Expert workshops from Solo.io.
Next lab
Congratulations, you have enabled strict mTLS policy for all services in the entire Istio mesh. We'll explore controlling traffic with these services in the next lab.

From <https://play.instruqt.com/embed/soloio/tracks/get-started-istio/challenges/securing-com-with-istio/assignment> 
Control Traffic
You are now ready to take control of how traffic flows between your services. In a Kubernetes environment, there is simple round-robin load balancing between service endpoints. While Kubernetes does support rolling upgrades, this is fairly coarse-grained and is limited to moving to a new version of the service. You may find it necessary to dark launch a new version, then canary test your new version with some portion of requests before shifting all traffic to it. You will now explore many of the Istio features which control the traffic between services while increasing the resiliency.
Prerequisites
Verify you're in the correct folder for this lab: /root/istio-workshops/istio-basics. This lab builds on the earlier lab where you enforced mTLS for your services in the mesh.
shell
copy
cd /root/istio-workshops/istio-basics
Dark Launch

You may find the v1 of the purchase-history service is rather boring as it always return the Hello From Purchase History (v1)! message. You want to make a new version of the purchase-history service so that it returns dynamic messages based on the result from querying an external service, for example the JsonPlaceholder service.
Dark launch allows you to deploy and test a new version of a service while minimizing the impact to your users, e.g. you can keep the new version of the service in the dark. Using a dark launch approach enables you to deliver new functions rapidly with reduced risk. Istio allows you to preceisely control how new versions of services are rolled out without the need to make any code change to your services or redeploy your services.
	1. You have v2 of the purchase-history service ready in the labs/05/purchase-history-v2.yaml file:
shell
copy
cat labs/05/purchase-history-v2.yaml
The main changes are the purchase-history-v2 deployment name and the version:v2 labels, along with the fake-service:v2 image and the newly added EXTERNAL_SERVICE_URL environment variable. The purchase-history-v2 pod establishes the connection to the external service at startup time and obtains a random response from the external service when clients call the v2 of the purchase-history service.
Should you deploy the labs/05/purchase-history-v2.yaml to your Kubernetes cluster next? What percentage of the traffic will visit v1 and v2 of the purchase-history services? Because both of the deployments have replicas: 1, you will see 50% traffic goes to v1 and 50% traffic goes to v2. This is probably not what you would want because you haven't had a chance to test v2 in your Kubernetes cluster yet.
You can use Istio's networking resources to dark launch the v2 of the purchase-history service. Virtual Service gives you the ability to configure a list of routing rules that control how your Envoy Proxy routes requests to services within the service mesh. The client could be Istio's ingress-gateway or any of your other services in the mesh. In a previous lab, when the client is istio-ingressgateway, the virtual service is bound to the web-api-gateway gateway. If you recall the Kiali graph for our application from the prior lab, the client for the purchase-history service in your application is the recommendation service.
A destination rule allows you to define configurations of policies that are applied to a request after the routing rules are enforced as defined in the destination virtual service. In addition, a destination rule is also used to define the set of Kubernetes pods that belong to a specific grouping, for example multiple versions of a service, which are called subsets in Istio.
	2. You can review the virtual service resource for the purchase-history service that configures all traffic to v1 of the purchase-history service:
shell
copy
cat labs/05/purchase-history-vs-all-v1.yaml
	3. Also review the destination rule resource for the purchase-history service that defines the v1 and v2 subsets. Since v2 is dark-launched and no traffic will go to v2, it is not required to have v2 subsets now but you will need them soon.
shell
copy
cat labs/05/purchase-history-dr.yaml
	4. Apply the purchase-history-vs and purchase-history-dr resources in the istioinaction namespace:
shell
copy
kubectl apply -f labs/05/purchase-history-vs-all-v1.yaml -n istioinaction
kubectl apply -f labs/05/purchase-history-dr.yaml -n istioinaction
	5. After you have configured Istio to send 100% of traffic to purchase-history to v1 of the service, you can now deploy the v2 of the purchase-history service:
shell
copy
kubectl apply -f labs/05/purchase-history-v2.yaml -n istioinaction
	6. Confirm the new v2 purchase-history pod is running:
shell
copy
kubectl get pods -n istioinaction -l app=purchase-history
You should see both v1 and v2 of the purchase-history pods are running, each with its own sidecar proxy.
	7. Make some requests to the purchase-history application
shell
copy
for i in {1..10}; do curl -s --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP|grep "Hello From Purchase History"; done
You will see all of the responses from purchase-history are from v1 of the service. This is great! We introduced the v2 service but it is currently not yet routable. Let us verify its health and start sending traffic.
Verify purchase-history-v2
Recall the v2 of the purchase-history service has some code to call the external service and requires the ability for the pod to connect to the external service during initialization. By default in Istio, the istio-proxy starts in parallel with the application container (purchase-history-v2 here in our example) so it is possible that the application container is running before istio-proxy fully starts, and it could be unable to connect to any external services outside of the cluster yet.
	1. Check the purchase-history-v2 pod logs to see if there are any errors:
shell
copy
kubectl logs deploy/purchase-history-v2 -n istioinaction
Note the connection refused error at the beginning of the log during the service initialization.
This is not good, the purchase-history-v2 pod cannot reach the JsonPlaceHolder external service at startup time. Generate some load on the web-api service to ensure your users are not impacted by the newly added v2 of the purchase-history service:
	2. How can we solve this problem and ensure the application container can connect to services outside of the cluster during the container start time? The holdApplicationUntilProxyStarts configuration is introduced in Istio to solve this problem. Let us add this configuration to the pod annotation of the purchase-history-v2 and use it:
shell
copy
cat labs/05/purchase-history-v2-updated.yaml
From the holdApplicationUntilProxyStarts annotation below, you have configured the purchase-history-v2 pod to delay starting until the istio-proxy container reaches its Running status:
yaml
copy
  template:
    metadata:
      labels:
        app: purchase-history
        version: v2
      annotations:
        proxy.istio.io/config: '{ "holdApplicationUntilProxyStarts": true }'
    spec:
	3. Deploy the updated v2 of the purchase-history service:
shell
copy
kubectl apply -f labs/05/purchase-history-v2-updated.yaml -n istioinaction
	4. Check the purchase-history-v2 pod logs to see to ensure there are no errors this time:
shell
copy
kubectl logs deploy/purchase-history-v2 -n istioinaction
You will see we are able to connect to the external service in the log:
copy
2021-06-11T18:13:03.573Z [INFO]  Able to connect to : https://jsonplaceholder.typicode.com/posts=<unknown>
	5. Test the v2 of the purchase-history service from its own sidecar proxy:
shell
copy
kubectl exec deploy/purchase-history-v2 -n istioinaction -c istio-proxy -- curl -s localhost:8080
Awesome! You are getting a valid response this time. If you rerun the above command, you will notice a slightly different body from purchase-history-v2 each time.
copy
  "body": "Hello From Purchase History (v2)! + History: 24 Title: autem hic labore sunt dolores incidunt Body: autem hic labore sunt dolores incidunt",
"code": 200
Selectively Route Requests

	1. You want to test the v2 of the purchase-history service only from a specific test user while all other requests continue to route to the v1 of the purchase-history service. With Istio's virtual service resource, you can specify HTTP routing rules based on HTTP requests such as header information. In the purchase-history-vs-all-v1-header-v2.yaml file shown in the following example, you will see that an HTTP route rule has been defined to route requests from clients using user: Tom header to the v2 of the purchase-history service. All other client requests will continue to use the v1 subset of the purchase-history service:
shell
copy
cat labs/05/purchase-history-vs-all-v1-header-v2.yaml
	2. Review the changes of the purchase-history-vs virtual service resource. Apply the changes to your mesh using this command:
shell
copy
kubectl apply -f labs/05/purchase-history-vs-all-v1-header-v2.yaml -n istioinaction
	3. Send some traffic to the web-api service through the istio-ingressgateway:
shell
copy
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" -H "user: mark" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
	4. You will get Hello From Purchase History (v1)! in the response. Why is that? Recall we configured exact in exact: Tom earlier. Change the command using user: Tom instead:
shell
copy
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" -H "user: Tom" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
You should see Hello From Purchase History (v2)! in the response! Feel free to send a few more requests. Based on your routing rule configuration, Istio made sure that requests with header user: Tom always route to the v2 of the purchase-history service while all other requests continue to route to v1 of the purchase-history service.
Canary Testing
You have dark-launched and done some basic testing of the v2 of the purchase-history service. You want to canary test a small percentage of requests to the new version to determine whether there are problems before routing all traffic to the new version. Canary tests are often performed to ensure the new version of the service not only functions properly but also doesn't cause any degradation in performance or reliability.
Shift 20% Traffic to v2

	1. Review the updated purchase-history virtual service resource that shifts 20% of the traffic to v2 of the purchase-history service:
shell
copy
cat labs/05/purchase-history-vs-20-v2.yaml
You will notice subset: v2 is added which will get 20% of the traffic while subset: v1 will get 80% of the traffic.
	2. Deploy the updated purchase-history virtual service resource:
shell
copy
kubectl apply -f labs/05/purchase-history-vs-20-v2.yaml -n istioinaction
	3. Generate some load on the web-api service to check how many requests are served by v1 and v2 of the purchase-history service. You should see only a few from v2 while the rest from v1. You may be curious why you are not seeing an exactly 80%/20% distribution among v1 and v2. You may need to have over 100 requests to see this 80%/20% weighted version distribution, but it should balance eventually.
shell
copy
for i in {1..20}; do curl -s --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP|grep "Hello From Purchase History"; done
Shift 50% Traffic to v2
	1. Review the updated purchase-history virtual service resource:
shell
copy
cat labs/05/purchase-history-vs-50-v2.yaml
You will notice subset: v2 is updated to get 50% of the traffic while subset: v1 will get 50% of the traffic.
	2. Deploy the updated purchase-history virtual service resource:
shell
copy
kubectl apply -f labs/05/purchase-history-vs-50-v2.yaml -n istioinaction
	3. Generate some load on the web-api service to check how many requests are served by v1 and v2 of the purchase-history service. You should observe roughly 50%/50% distribution among the v1 and v2 of the service.
shell
copy
for i in {1..20}; do curl -s --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP|grep "Hello From Purchase History"; done
Shift All Traffic to v2
Now you haven't observed any ill effects during your test, you can adjust the routing rules to direct all of the traffic to the canary deployment:
	1. Deploy the updated purchase-history virtual service resource:
shell
copy
kubectl apply -f labs/05/purchase-history-vs-all-v2.yaml -n istioinaction
	2. Generate some load on the web-api service, you should only see traffic to the v2 of the purchase-history service.
shell
copy
for i in {1..20}; do curl -s --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP|grep "Hello From Purchase History"; done
Controlling Outbound Traffic
When you use Kubernetes, any application pod can make calls to services that are outside the Kubernetes cluster unless there is a Kubernetes network policy that prevents it. However, network policies are restricted to layer-4 rules which means that they can only allow or restrict access to specific IP addresses. What if you want more control over how applications within the mesh can reach external services using layer-7 policies and a more fine-grained attribute policy evaluation?
By default, Istio allows all outbound traffic to ensure users have a smooth starting experience. If you choose to restrict all outbound traffic across the mesh, you can update your Istio install to enable restricted outbound traffic access so that only registered external services are allowed. This is highly recommended.
	1. Check the default Istio installation configuration for outboundTrafficPolicy:
shell
copy
kubectl get istiooperator installed-state -n istio-system -o jsonpath='{.spec.meshConfig.outboundTrafficPolicy.mode}'
The output should be empty. This means the default mode ALLOW_ANY is used, which allows services in the mesh to access any external service.
	2. Update your Istio installation so that only registered external services are allowed, using the meshConfig.outboundTrafficPolicy.mode configuration:
shell
copy
istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY -y
	3. Confirm the new configuration, you should see REGISTRY_ONLY from the output:
shell
copy
kubectl get istiooperator installed-state -n istio-system -o jsonpath='{.spec.meshConfig.outboundTrafficPolicy.mode}'
	4. Send some traffic to the web-api service (You may need to wait for a minute for it to update):
shell
copy
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
You should see the request to purchase-history to fail because all outbound traffics are blocked by default but the v2 of purchase-history service needs to connect to the jsonplaceholder.typicode.com service.
copy
      "upstream_calls": [
{
          "uri": "http://purchase-history:8080",
          "code": 503,
          "error": "Error processing upstream request: http://purchase-history:8080/"
        }
      ],
	5. Check the pod logs of the purchase-history-v2 pod:
shell
copy
kubectl logs deploy/purchase-history-v2 -n istioinaction | grep "x-envoy-attempt-count: 3" -A 10
You can see envoy attempted three times, including the two retries by default, and the service can't connect to the external service (jsonplaceholder.typicode.com here).
copy
x-envoy-attempt-count: 3
x-envoy-internal: true
x-forwarded-proto: https
x-b3-sampled: 1
accept: */*
x-forwarded-for: 10.42.0.1"
2021-06-15T02:23:09.301Z [INFO]  Sleeping for: duration=-265.972µs
Get "https://jsonplaceholder.typicode.com/posts?id=65": EOF
2021-06-15T02:23:09.304Z [INFO]  Finished handling request: duration=3.393441ms
2021/06/15 02:23:09 http: panic serving 127.0.0.6:41109: json: error calling MarshalJSON for type json.RawMessage: invalid character 'h' after top-level value
Above is the expected behavior of the REGISTRY_ONLY outboundTrafficPolicy mode in Istio. When services in the mesh attempts to access external services, only registered external services are allowed, and you haven't registered any external services yet.
	6. Istio has the ability to selectively access external services using a Service Entry resource. A Service Entry allows you to bring a service that is external to the mesh and make it accessible by services within the mesh. In other words, through service entries, you can bring external services as participants in the mesh. You can create the following service entry resource for the jsonplaceholder.typicode.com service:
shell
copy
cat labs/05/typicode-se.yaml
	7. Apply the service entry resource into the istioinaction namespace:
shell
copy
kubectl apply -f labs/05/typicode-se.yaml -n istioinaction
	8. Send some traffic to the web-api service. You should get a 200 response now.
shell
copy
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
	9. Another important benefit of importing external services through service entries in Istio is that you can use Istio routing rules with external services to define retries, timeouts, and fault injection policies. For example, you can set a timeout rule on calls to the jsonplaceholder.typicode.com service as shown below:
shell
copy
cat labs/05/typicode-vs.yaml
	10. Run the following command to apply the virtual service resource:
shell
copy
kubectl apply -f labs/05/typicode-vs.yaml -n istioinaction
Visualize
If you want a visual representation of what you just observed, head over to the Kiali UI tab. You should now see that v1 and v2 of purchase-history are shown as their own workloads.
Questions
Do you want to securely restrict which pods can access a given external service? Should you send traffic to your external service through the istio-egressgateway? We will cover this in the Istio Expert workshop.
Conclusion
A service mesh like Istio has capabilities that enable you to manage traffic flows within the mesh as well as entering and leaving the mesh. These capabilities allow you to efficiently control rollout and access to new features, and make it possible to build secure and resilient services without having to make complicated changes to your application code.

From <https://play.instruqt.com/embed/soloio/tracks/get-started-istio/challenges/control-traffic/assignment> 
Resiliency and Chaos Testing
You will explore some features provided by Istio to increase the resiliency between the services. When you build a distributed application, it is critical to ensure the services in your application are resilient to failures in the underlying platforms or the dependent services. Istio has support for retries, timeouts, circuit breakers, and even injecting faults into your service calls to help you test and tune your timeouts. Similar to the dark launch and canary testing you explored earlier, you don't need to add these logic rules into your application code or redeploy your application when configuring these Istio features to increase the resiliency of your services.
Retries
	1. Istio supports program retries for your services in the mesh without you specifying any changes to your code. By default, client requests to each of your services in the mesh will be retried twice. What if you want a different number of retries per route for some of your virtual services? You can adjust the number of retries or disable them altogether when automatic retries don't make sense for your services. Display the content of the purchase-history-vs-all-v2-v3-retries.yaml:
shell
copy
cat labs/05/purchase-history-vs-all-v2-v3-retries.yaml
Note the number of retries configuration is for the purchase-history service when the http header user matches exactly to Tom then routes to v3 of the purchase-history service. All other cases, it continues to route to v2 of the purchase-history service.
	2. Apply the virtual service resource to the istioinaction namespace, along with the updated purchase-history destination rule that contains the v3 subset:
shell
copy
kubectl apply -f labs/05/purchase-history-vs-all-v2-v3-retries.yaml -n istioinaction
kubectl apply -f labs/05/purchase-history-dr-v3.yaml -n istioinaction
	3. To see the new retry configuration in action, you can create a new version of the purchase history (v3) which errors 50% of the time with 503 error code and 4 seconds of error delay. You will use it to simulate a bad application deployment and show how retries would help prevent end users from experiencing them.
shell
copy
cat labs/05/purchase-history-v3.yaml
yaml
copy
        - name: "ERROR_CODE"
          value: "503"
        - name: "ERROR_RATE"
          value: "0.5"
        - name: "ERROR_TYPE"
          value: "delay"
        - name: "ERROR_DELAY"
          value: "4s"
	4. Deploy the new version of the purchase history (v3) to the istioinaction namespace:
shell
copy
kubectl apply -f labs/05/purchase-history-v3.yaml -n istioinaction
	5. Generate some load with the user: Tom header to ensure traffic goes 100% to v3 of the purchase-history service. You will quickly see errors from the v3 of the purchase-history service:
shell
copy
for i in {1..6}; do kubectl exec deploy/sleep -n istioinaction -- curl -s -H "user: Tom" http://purchase-history:8080/; done
You will see 50% successful rate and 50% failure with 503 error code. If you remove the retries configuration (see labs/05/purchase-history-vs-all-v2-header-v3.yaml) and use Istio's default retry, you should not see any errors from v3 of the purchase-history service because the service errors 50% of the time and Istio will retry any failed request to the service with 503 error code automatically up to two times:
	6. Lets add back the automatic retries by removing the retries.attempts: 0 in the VirtualService
shell
copy
cat labs/05/purchase-history-vs-all-v2-header-v3.yaml
kubectl apply -f labs/05/purchase-history-vs-all-v2-header-v3.yaml -n istioinaction
	7. Generate some load you should NOT see any errors from the v3 of the purchase-history service:
shell
copy
for i in {1..6}; do kubectl exec deploy/sleep -n istioinaction -- curl -s -H "user: Tom" http://purchase-history:8080/; done
	8. If you check the logs of the purchase-history service, you will see the retries:
shell
copy
kubectl logs deploy/purchase-history-v3 -n istioinaction | grep x-envoy-attempt-count
In the log shown below, the first request was not successful but the automatic retry was successful:
yaml
copy
x-envoy-attempt-count: 1
x-envoy-attempt-count: 2
x-envoy-attempt-count: 1
x-envoy-attempt-count: 2
Note: The error code has to be 503 for Istio to retry the requests. If you change the ERROR_CODE to 500 in the purchase-history-v3.yaml, redeploy the updated purchase-history-v3.yaml file and send some request to the purchase-history service from the sleep pod, you will get errors 50% of the time.
Traffic Monitoring
Lets take a look at the traffic in the Kiali UI
	1. Click on the Kiali UI tab, and you should be able to view the Kiali UI. Navigate to the Graph Tab and select the istioinaction namespace.
	2. Change the time to Last 1h and click on the purchase-history v3 service. On the right hand side you should be able to observe the errors and response times.
Timeouts
	1. Istio has built-in support for timeouts with client requests to services within the mesh. The default timeout for HTTP request in Istio is disabled, which means there is no timeout. You can overwrite the default timeout setting of a service route within the route rule for a virtual service resource. For example, in the route rule within the purchase-history-vs resource below, you can add the following timeout configuration to set the timeout of the route to the purchase-history service on port 8080, along with three retry attempts with each retry timing out after three seconds.
shell
copy
cat labs/05/purchase-history-vs-all-v2-v3-retries-timeout.yaml
	2. Apply the resource in the istioinaction namespace:
shell
copy
kubectl apply -f labs/05/purchase-history-vs-all-v2-v3-retries-timeout.yaml -n istioinaction
	3. Send some traffic to the purchase-history service from the sleep pod:
shell
copy
for i in {1..6}; do kubectl exec deploy/sleep -n istioinaction -- curl -s -H "user: Tom" http://purchase-history:8080/|grep timeout; done
You will see three requests time out:
copy
upstream request timeout
upstream request timeout
upstream request timeout
With the addition of 3 retries, and a 50% failure rate, why is it returning "upstream request timeout" errors? Shouldn't it have retried after a failure with the subsequent attempt succeeding? It didn't retry because retries will be done for only specific retry conditions that you can configure.
The default retryOn conditions don't include a response taking too long, which is what happened with the 3 second preTryTimeout and the delay from purchase-history-v3 taking 4 seconds. You can view the conditions that can be configured with retryOn.
	4. Let's add retryOn conditions to include 5xx which will result in retry attempts being done for a delayed response (readTimeout).
Display the contents of purchase-history-vs-all-v2-v3-retries-timeout-retryon.yaml with the addition of the retryOn line:
shell
copy
cat labs/05/purchase-history-vs-all-v2-v3-retries-timeout-retryon.yaml
Apply the resource:
shell
copy
kubectl apply -f labs/05/purchase-history-vs-all-v2-v3-retries-timeout-retryon.yaml -n istioinaction
Now when you generate load with user: Tom, there will be a delay, but no more upstream request timeout errors due to the retries now being done for delayed responses:
shell
copy
for i in {1..6}; do kubectl exec deploy/sleep -n istioinaction -- curl -s -H "user: Tom" http://purchase-history:8080/|grep timeout; done
Circuit Breakers
Circuit breaking is an important pattern for creating resilient microservice applications. Circuit breaking allows you to limit the impact of failures and network delays, which are often outside of your control when making requests to dependent services. Prior to having your service mesh, you had to add logic directly within your code (or your language specific library) to handle situations when the calling service fails to provide the desirable result. Istio allows you to apply circuit breaking configurations within a destination rule resource, eliminating the need to modify your service code.
	1. Take a look at the updated purchase-history-dr destination rule as shown in the example that follows. It defines the connection pool configuration to indicate the maximum number of TCP connections, show the maximum number of HTTP requests per connection, and set the outlier detection to be three minutes after an error. When any clients access the purchase-history-dr service, these circuit-breaker behavior rules will be followed.
shell
copy
cat labs/05/purchase-history-dr-v3-cb.yaml
	2. Apply the resource in the istioinaction namespace:
shell
copy
kubectl apply -f labs/05/purchase-history-dr-v3-cb.yaml -n istioinaction
	3. Make requests against the purchase-history application with outlier detection enabled.
shell
copy
for i in {1..50}; do kubectl exec deploy/sleep -n istioinaction -- curl -s -H "user: Tom" http://purchase-history:8080/; done
You may have noticed but our current outlier detection configuration tells Istio to remove the purchase-history backend if it returns any 5xx error. It then will not allow routing to it until 5s has elapsed.
yaml
copy
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 5s
      maxEjectionPercent: 100
Questions
	• Can you apply the purchase-history-dr-v3-cb.yaml in the istio-system namespace? You will learn more about this later in the workshop.
	• Note the purchase-history-dr destination rule applies to requests from any client to the purchase-history service in the istioinaction namespace. Do you want to configure the client scope of the destination rule? You can learn how to do that in Solo's Istio Essentials course.
Fault Injection
It can be difficult to configure service timeouts and circuit-breakers properly in a distributed microservice application. Istio makes it easier to get these settings correct by enabling you to inject faults into your application without the need to modify your code. With Istio, you can perform chaos testing of your application easily by adding an HTTP delay fault into the web-api service only for user Amy so that the injected fault doesn't affect any other users.
	1. You can inject a 30-second fault delay for 100% of the client requests when the user HTTP header value exactly matches the value Amy.
shell
copy
cat labs/05/web-api-gw-vs-fault-injection.yaml
	2. Apply the virtual service resource using the following command:
shell
copy
kubectl apply -f labs/05/web-api-gw-vs-fault-injection.yaml -n istioinaction
	3. Send some traffic to the web-api service, you should see 200 response code right away:
shell
copy
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
	4. Send some traffic to the web-api service with the user: Amy header, you should see a 200 response code after a 15 seconds delay:
shell
copy
curl --cacert ./labs/02/certs/ca/root-ca.crt -H "Host: istioinaction.io" -H "user: Amy" https://istioinaction.io:$SECURE_INGRESS_PORT --resolve istioinaction.io:$SECURE_INGRESS_PORT:$GATEWAY_IP
Note that there is no change to the web-api service to inject the delay. Istio injects the delay automatically via programmatically configuring the istio-proxy container.

