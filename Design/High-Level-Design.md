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
* Azure Key Vault will be used to securely store and manage sensitive information, such as database credentials and tls certs.
* Azure Active Directory (AAD) is used for authentication and authorization to KubeAPI.
* AKS KubeAPI end point will be private and private endpoints access will be provided to the solution vnet for ACR, keyvault and database endpoints.
