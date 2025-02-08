# High Level Design

## Assumptions

The following assumptions are made about the environment and the requirements:

* The Cassandra cluster and the Postgres Database are not deployed and hosted as part of this solution.
  * _Cassandra_:
  	* In order to maintain Scalability, high availability and security requirements the cassandra cluster will have suitable multi region and/or multi DC network topology, LOCAL_QUORUM consistency (to allow for DC failures) and a suitable monitoring strategy in place for node scaling and optimising.
  	* Suitable least privilege access credentials should be configured for the Cleanup Job and Main App.
  	* The multiDC contact points should be loadbalanced (outside this solution) providing a single contact point. The advantage being that the list of contact points does not need to be supplied to the app solution and the application will not have the overhead of using one of the various load balancing policies.
  * _Postgres_:
  	* A suitable distributed Postgres or Postgres compatible solution is implemented for high availability. Cloud provider solutions such as Azure PostgreSQL HA could be used.
  	* Suitable least privilege access credentials should be configured for the Auth Service.
  	* The Postgres implementation should provide a single load balanced access point.
* HTTP is specified as the protocol for communications - this raises immediate security concerns. If HTTP is used then security could be increased if this solution was only used in an internal vnet/vpc but intruders who compromise the network or simply internal users without authorised access could still observe the traffic. The specification mentions a mobile app so this implies external access. **I would recommend HTTPS for this solution to protect against MITM or replay attacks**.
* What is the external endpoint fo this solution? The specification describes the Authentication service checking credentials and forwarding requests to the Main App and Main app responding to period connections from a mobile app. This could be implemented in two ways
  1. ALL traffic is authenticated by the authentication service and requests from the mobile app are assumed to be traffic forwarded by the Authentication Service.  This means that Authentication Service is acting as a reverse proxy for external traffic and the only External endpoint is for Authentication Service.
    ![Authentication service reverse proxy](1_auth_service_reverse_proxy.svg 'Option 1 Reverse Proxy')
  2. Requests to the authentication service are used to obtain an authentication token that is used for subsequest requests to the Main App. This means that the external endpoints are provided for both authentication service and Main app.
    ![Authentication service token generator](2_auth_service_token_generator.svg 'Option 2 Token Generator')

  **I will assume option 1 - reverse proxy**

## Platform Architecture in Azure

![Platform Architecture](Highlevelsolution.png 'Platform Architecture')

* Azure Front door with NSG firewall will be used to provide the access point for the mobile application
* Azure Kubernetes Service (AKS) will be used to host the solution applications.
* Azure Container Registry (ACR) will be used to store the Docker images.
* When a production deployment is triggered the CICD platform will:
  * Create a docker image for Authentication service using the jar file artifact.
  * Create a docker image for Main app using the NodeJS tar artifact.
  * Create a docker image for the Cronjob which will simply execute the Python cleanup script artifact
  * Push all images to the ACR
  * Deploy the application to AKS using Helm
* Azure Key Vault will be used to securely store and manage sensitive information, such as database credentials and tls certs.
* Azure Active Directory (AAD) is used for authentication and authorization to KubeAPI.
* AKS KubeAPI end point will be private and private endpoints access will be provided to the solution vnet for ACR, keyvault and database endpoints.

## Kubernetes Deployment

* A helm chart will be produced for the application to deploy all of the following manifests.
* The "Auth Service" can be deployed as a Kubernetes deployment and service. The deployment manifest would specify the Docker image to use, the environment variables (such as database credentials), TLS certs for HTTPs and the config map to define reverse proxy routing to the Main App.
* The "Main App" can also be deployed as a Kubernetes deployment. The manifest would specify the Docker image, environment variables, and any other configuration.
* The "Cleanup Job" can be deployed as a Kubernetes cron job. The manifest would specify the Docker image, the schedule for running the job, and any other configuration.

## Considerations

* Security: Implement authentication and authorization mechanisms using Entra ID AAD. Ensure that the database credentials are securely stored and accessed.
* Scalability: Use Kubernetes' built-in scaling mechanisms to scale the deployments and services as needed.
* Maintainability: Use Kubernetes' rolling updates and rollbacks to deploy latest Kubernetes versions and node images.
* High Availability: Ensure that the services are deployed with multiple replicas and that the Kubernetes cluster is configured for high availability. This can include using multiple Availability Zones, configuring load balancers, and implementing health checks.
* Cost effectiveness: Monitor and optimize resource usage to minimize costs.

Here is an example of what the Kubernetes deployment manifests might look like:
```yml
# Auth Service deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-service
        image: <ACR_URL>/auth-service:latest
        ports:
        - containerPort: 443
        volumeMounts:
        - name: tls-certs
          mountPath: /etc/tls
        - name: config
          mountPath: /etc/config
        env:
        - name: POSTGRESS_CREDENTIALS
          valueFrom:
            secretKeyRef:
              name: postgress-credentials
              key: credentials
      volumes:
      - name: tls-certs
        secret:
          secretName: auth-service-tls
      - name: config
        configMap:
          name: auth-service-config

# Auth Service service
apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  selector:
    app: auth-service
  ports:
  - name: https
    port: 443
    targetPort: 443
  type: LoadBalancer

# Main App deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: main-app
  template:
    metadata:
      labels:
        app: main-app
    spec:
      containers:
      - name: main-app
        image: <ACR_URL>/main-app:latest
        env:
        - name: CASSANDRA_CREDENTIALS
          valueFrom:
            secretKeyRef:
              name: cassandra-credentials
              key: credentials        
        ports:
        - containerPort: 443
        volumeMounts:
        - name: tls-certs
          mountPath: /etc/tls
      volumes:
      - name: tls-certs
        secret:
          secretName: main-app-tls        

# Auth Service config map
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-service-config
data:
  reverse-proxy.conf: |
    server {
      listen 443 ssl;
      server_name example.com;

      ssl_certificate /etc/tls/tls.crt;
      ssl_certificate_key /etc/tls/tls.key;

      location / {
        proxy_pass http://main-app:443;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
    }

# Cleanup Job
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-job
spec:
  schedule:
    - cron: 0 0 * * *
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup-job
            image: <ACR_URL>/cleanup-job:latest
        env:
        - name: CASSANDRA_CREDENTIALS
          valueFrom:
            secretKeyRef:
              name: cassandra-credentials
              key: credentials
