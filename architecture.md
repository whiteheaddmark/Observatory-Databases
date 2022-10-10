<h1 style="text-align: center;">NRAO Data Operations Interfaces</h1>

# Overview
NRAO uses internal and external data sources to manage and process radio telescope measurements. An increase over time in the number of supported telescopes and variety of data sources drives the need to provide consistent and manageable interfaces to operational data for various stakeholders. This document outlines the general requirements for such interfaces and offers a high-level architectural solution. The proposed high-level architectural solution is intended to support detailed design activities prior to development.

# Approach
DMS utilizes three architecture designations:
- Conceptual - The most abstract architecture, primarily focused on structures, highlights relationships between key concepts - not how they work - and contains no implementation details.
- Logical - This architecture is broad in scope, can model high and low levels of detail, and captures dynamic behavior. Logical architectures do not identify any particular technology or infrastructure unless it is advantageous to do so.
- Physical - This is the least abstract architecture, is detail-oriented, and includes entities that point to real life software, services, servers, systems, networks, etc.

This document proposes a conceptual architectural style, two related logical architecture patterns, and additional design considerations. The intent is to define boundaries for the system based on the general requirements and support the development team with a clear starting point for detailed design.

# Requirements
This section outlines the general requirements. Additional requirements analysis may be needed to support detailed design.

- The interface and supporting infrastructure will be created and maintained by DMS.
- The interface and supporting infrastructure must provide remote access to NRAO and non-NRAO data sources.
- The interface must support access to versioned data sources.
- Access mechanisms to the interface must support identity and access management (IAM) best practices and utilize DMS IAM technologies.
- The interface must support JSON-formatted responses.
- The system is not required to support "Big Data"-scale volumes or velocities.
- The infrastructure must support standard reliability mechanisms to provide and maintain reliable access to data sources. 
- Data consumers are responsible for caching strategies to mitigate problems related to temporary loss of access to the interface.

# Conceptual Architecture
The Representational state transfer (REST) software architectural style satisfies the general requirements for abstracting access to NRAO and non-NRAO data sources. REST is based in part on the client-server model where clients access resources, in this case objects, data, or services, via Uniform Resource Identifiers. Although REST is independent of any protocol, it commonly uses HTTP and this document assumes its use. REST used with HTTP to provide access to resources is generally referred to as REST APIs.
REST APIs express the following design principles:
- **Uniform Interface** This principle consists of the following set of constraints:
    - An interface must uniquely identify each resource.
    - Clients interact with an API by exchanging representations of resources; many APIs use JSON as the exchange format.
    - Each resource representation should carry enough information to describe how to process the message.
    - REST APIs are driven by hypermedia links that are contained in the representation.
    - For REST APIs built on HTTP, the uniform interface uses standard HTTP verbs to perform operations on resources.
- **Client-server** The client-server model enforces separation of concerns, which helps client and server components evolve independently.
- **Stateless** Each request from the client to the server must contain all of the information necessary to understand and complete the request. The server cannot take advantage of any previously stored context information on the server. The client application must maintain session state.
- **Cacheable** Responses should label themselves as cacheable or non-cacheable. If a response is cacheable, the client gets the right to reuse the response data later for equivalent requests and a specified period. This is accomplished via HTTP headers.
- **Layered** This principle supports enforcing separation of concerns in deployment between levels of abstraction, e.g. API servers, persistence servers, authentication servers, etc.

REST frameworks help developers construct REST APIs and usually offer the following relevant features:
- Modules that support common security specifications for authentication and authorization.
- Support for organizing interfaces into domains and subdomains to help manage complex services.
- Generate API documentation automatically.

# Logical Architecture
## REST Data Services
The general strategy with REST APIs is to organize the API design around resources and provide access to the resources via services. Services model entities and operations clients can perform on those entities while encapsulating the internal implementation. Initially for this system, there will likely be a one-to-one mapping between a service and a data source. However, a resource does not have to be based on a single data item. A service could internally aggregate data items from a variety of sources and present the aggregate entity to the client.

<p align="center">
  <img src="https://github.com/whiteheaddmark/Observatory-Databases/blob/master/images/Services.png?raw=true">
</p>

<div align="center">Figure 1 Generic REST services backed by a database, a file system, or a non-NRAO resource.</div>
</br>
This general strategy raises the question: how should REST API clients access individual services?

## Direct Client-to-Service
A possible approach is to use a direct client-to-service communication architecture. In this approach, a client makes requests directly to services via a URL. In a production environment the URL would map to some entity that provides load balancing and IAM but that is transparent from the logical architecture point of view.

<p align="center">
  <img src="https://github.com/whiteheaddmark/Observatory-Databases/blob/master/images/Client-to-Service.png?raw=true">
</p>

<div align="center">Figure 2 Client-to-service communication architecture.</div>
</br>

Depending on how complex the system becomes, direct communication could face difficulties in the future:
- This approach could increase complexity and latency on the client.
- It could become increasingly difficult to handle cross-cutting concerns.
- Clients may need to communicate with services that don't use internet-friendly protocols.
- It may be difficult to optimize the services' API for a variety of clients. 

## API Gateway Pattern
An alternative approach, called the API Gateway Pattern, is a more flexible solution to the problem of how clients can access services. An API gateway is the single entry point for all clients and handles requests by either routing requests to the appropriate service or by fanning out a request to multiple services. 

<p align="center">
  <img src="https://github.com/whiteheaddmark/Observatory-Databases/blob/master/images/SingleAPIGateway.png?raw=true">
</p>

<div align="center">Figure 3 Single API Gateway Pattern.</div>
</br>

This pattern introduces more flexibility into the design by:
- Insulating clients from how the system is partitioned into services.
- Insulating clients from the problem of determining service locations.
- Providing the optimal API for each client, if needed.
- Reducing the number of requests and round trips, i.e. this pattern enables clients to retrieve data from multiple services with a single round-trip.
- Simplifying clients by moving logic for calling multiple services from the client to the API gateway.
- Translating from web-friendly protocols to any internally-used protocol.
- Providing compatibility with the Microservice Architecture Pattern if the system should evolve to that level of complexity in the future.
- Providing a single point to consolidate cross-cutting concerns like IAM.

API Gateway Pattern drawbacks include:
- **Increased complexity** The API gateway is yet another part of the system that must be developed, deployed and managed.
- **Increased response time** Adding the API gateway adds an additional network step.
- **Increased development time** Developers have to include the gateway in the design, select supporting technologies, and learn how to use them for implementation. 

Multiple gateways, each dedicated to certain types of clients, could be added in the future if requirements warrant.

# API Design Considerations
The current best practice for building REST APIs is called “API-first development” which includes identifying API stakeholders, identifying key services, and designing API contracts. This section is intended to serve as a checklist of API design elements that should be considered during the API contract detailed design phase. A clear understanding of the various stakeholders and the similarities and differences between their service requirements is assumed to be an input into the detailed design phase and should include an analysis based on the idea presented in [REST Data Services](#REST-Data-Services).

## API Operations
HTTP defines a number of methods that assign semantic meaning to a request. Common HTTP methods used by most RESTful web APIs include:
- **GET** retrieves a representation of the resource at the specified URI. The body of the response message contains the details of the requested resource.
- **POST** creates a new resource at the specified URI. The body of the request message provides the details of the new resource. Note that POST can also be used to trigger operations that don't actually create resources.
- **PUT** either creates or replaces the resource at the specified URI. The body of the request message specifies the resource to be created or updated.
- **PATCH** performs a partial update of a resource. The request body specifies the set of changes to apply to the resource.
- **DELETE** removes the resource at the specified URI.

Table 1 suggests an API contract analysis structure for a hypothetical resource based on common HTTP methods.

| Resource | Post | Get | Put | Delete |
| -------- | ---- | --- | --- | ------ |
| /calmodels | Create new model | Retrieve all models | Bulk update of models | Remove all models |
| /calmodels/1 | Error | Retrieve details for model 1 | Update details of model 1 | Remove model 1 |
| /calmodels/1/measurements | Create new measure. for model 1 | Retrieve all measure. for model 1 | Bulk update of measure. for model 1 | Remove all measure. for model 1 |
<div align="center">Table 1 API contract design structure.</div>   

## API Organization
The following best practices should guide API organization and design:
- Adopt a consistent naming convention in URIs.
- Entities are often grouped together into collections; a collection is a separate resource from the item within the collection and should have its own URI; organize URIs for collections and items into a hierarchy; use plural nouns for URIs that reference collections. 
- Consider the relationships between different types of resources and how these associations should be exposed.
- Avoid introducing dependencies between the API and the underlying data sources.
- Avoid creating APIs that simply mirror the internal structure of a database.

## API Versions
Versioning enables an API to indicate the features and resources that it exposes and clients can submit requests that are directed to a specific version of a feature or resource. The API contract design should include a versioning mechanism to handle service or data source changes that could break client logic. 

Included below are the current versioning options to consider:
- **No versioning** This is the simplest approach and may be acceptable for some internal APIs. Significant changes could be represented as new resources or new links. Adding content to existing resources might not present a breaking change as client applications that are not expecting to see this content will ignore it.
- **URI versioning** Each API modification or schema change results in a version number update to the URI for each resource. The previously existing URIs should continue to operate as before, returning resources that conform to their original schema.
- **Query String versioning** Rather than providing multiple URIs, resource versions can be specified via a parameter within the query string appended to the HTTP request. This approach has the semantic advantage that the same resource is always retrieved from the same URI but it relies on the code that handles the request to parse the query string and send back the appropriate HTTP response.
- **Header versioning** Rather than appending the version number as a query string parameter, custom headers indicate the version of the resource. This approach requires that the client application adds the appropriate header to any requests. The code handling the client request could use a default value (version 1) if the version header is omitted.

## Authentication and Authorization
DMS plans to adopt a zero trust architecture to satisfy evolving IAM requirements. In the intermediate term, this involves the use of Red Hat Identity Management (based on FreeIPA) and Red Hat SSO (based on Keycloak) which support standard protocols including OpenId Connect, OAuth2, SAML, etc. NRAO data operations interfaces must utilize DMS IAM technologies. DMS maintains a strong preference for leveraging existing IAM libraries when possible and custom development in this area should only be a last resort. Additionally, the API contract detailed design phase should include an analysis of the authentication and authorization requirements for people and programs to access data sources. Finally, the logical design should be updated to include the use of DMS IAM technologies and the IAM parts of the API contract detailed design.

# References
The content of this document is completely derived from the following sources.
- [Representational state transfer](https://en.wikipedia.org/wiki/Representational_state_transfer)
- [List of architecture styles and patterns](https://en.wikipedia.org/wiki/List_of_software_architecture_styles_and_patterns)
- [Microsoft RESTful web API Design](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [Microsoft API Gateway Pattern](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/direct-client-to-microservice-communication-versus-the-api-gateway-pattern)
- [API Gateway Pattern](https://microservices.io/patterns/apigateway.html)
- [Web Service](https://en.wikipedia.org/wiki/Web_service)
- [REST API Security](https://stackoverflow.blog/2021/10/06/best-practices-for-authentication-and-authorization-for-rest-apis/)
- [REST API Design](https://stackoverflow.blog/2020/03/02/best-practices-for-rest-api-design/)