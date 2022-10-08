
# Overview

NRAO uses internal and external data sources to manage and process radio telescope measurements. An increase over time in the number of supported telescopes and varieties of data sources is driving the need to provide a consistent and manageable interface to operational data for a variety of stakeholders. This document outlines the known requirements for such an interface and offers a high-level architectural solution. The proposed architecture is intended to support additional detailed design activities prior to development.

# General Requirements
This section outlines the general requirements. Additional requirements analysis would be needed to support detailed design.

- The interface and supporting infrastructure will be created and maintained by DMS.
- The interface and supporting infrastructure must provide remote access to NRAO and non-NRAO data sources.
- The interface must support access to versioned data sources.
- Access to the interface must support identity and access management (IAM) best practices.
- The interface must support JSON-formatted responses.
- The system is not required to support "Big Data"-scale volumes or velocities.
- The infrastructure must support standard reliability mechanisms to support reliable access to data sources. 
- Data consumers are responsible for caching strategies to mitigate problems related to temporary loss of access to the interface.

# Proposed Conceptual Architecture
The Representational state transfer (REST) software architectural style satisfies the general requirements for abstracting access to NRAO and non-NRAO data sources. REST is based in part on the client-server model where clients access resources (i.e. object, data, or service) via Uniform Resource Identifiers. Although REST is independent of any protocol, it commonly uses HTTP and this document assumes its use. REST used with HTTP to provide access to resources is generally referred to as REST APIs.
REST APIs express the following design principles:
- **Uniform Interface** This principle consists of the following set of constraints:
    - An interface must uniquely identify each resource.
    - Clients interact with an API by exchanging representations of resources; many APIs use JSON as the exchange format.
    - Each resource representation should carry enough information to describe how to process the message.
    - REST APIs are driven by hypermedia links that are contained in the representation.
    - For REST APIs built on HTTP, the uniform interface includes using standard HTTP verbs to perform operations on resources. The most common operations are GET, POST, PUT, PATCH, and DELETE.
- **Client-server** The client-server model enforces separation of concerns, which helps client and server components evolve independently.
- **Stateless** Each request from the client to the server must contain all of the information necessary to understand and complete the request. The server cannot take advantage of any previously stored context information on the server. The client application must keep the session state.
- **Cacheable** Responses should label themselves as cacheable or non-cacheable.If a response is cacheable, the client gets the right to reuse the response data later for equivalent requests and a specified period. This is accomplished via HTTP headers.
- **Layered** This principle supports enforcing separation of concerns in deployment between, for example, API servers, persistence servers, authentication servers, etc.

REST frameworks help developers construct REST APIs and usually offer the following relevant features:
- Modules that support common security specifications for authentication and authorization.
- Support for organizing interfaces into domains and subdomains to help manage complex services.
- Generate API documentation automatically.

# Proposed Logical Architecture
## REST Data Services
The general strategy with REST APIs is to organize the API design around resources and provide access to the resources via services. Services model entities and the operations that a client can perform on those entities while not exposing the client to the internal implementation. Initially, there will likely be a one-to-one mapping between a service and a data source. However, a resource does not have to be based on a single data item. A service could internally aggregate data items from a variety of sources or via multiple mechanisms and present to the client a single entity.

<p align="center">
  <img src="https://github.com/whiteheaddmark/Observatory-Databases/blob/master/images/Services.png?raw=true">
</p>

<figcaption style="text-align: center;">Figure 1 Generic REST services backed by database, file system or non-NRAO resource.</figcaption>
<br>
This general strategy raises the question: how should REST API clients access individual services?

## Direct Client-to-Service
A possible approach is to use a direct client-to-service communication architecture. In this approach, a client makes requests directly to services via a URL. In a production environment the URL would map to some entity that provides load balancing and IAM but that is transparent from the logical architecture point of view.

<div style="text-align: center;">

![REST Services](/images/Client-to-Service.png)

</div>

Depending on how complex the system becomes, direct communication could face difficulties in the future:
- This approach could increase complexity and latency on the client.
- It could become increasingly difficult to handle cross-cutting concerns.
- Clients may need to communicate with services that don't use internet-friendly protocols.
- It may be difficult to optimize the services' API for a variety of clients. 

## API Gateway Pattern
An alternative approach called the API Gateway Pattern is a more general solution to the problem of how clients can access services. An API gateway is the single entry point for all clients and handles requests in one of two ways: some requests are simply routed to the appropriate service while other requests are fanned out to multiple services. 

<div style="text-align: center;">

![REST Services](/images/SingleAPIGateway.png)

</div>

This pattern introduces more flexibility into the design by:
- Insulating clients from how the system is partitioned into services.
- Insulating clients from the problem of determining service locations.
- Providing the optimal API for each client.
- Reducing the number of requests and round trips, i.e. this pattern enables clients to retrieve data from multiple services with a single round-trip.
- Simplifying clients by moving logic for calling multiple services from the client to the API gateway.
- Translating from web-friendly protocols to any internally-used protocol.
- Providing compatibility with the Microservice Architecture Pattern if the system should evolve to that level of complexity in the future.
- Providing a single point to consolidate cross-cutting concerns like IAM.

API Gateway Pattern drawbacks include:
- **Increased complexity** The API gateway is yet another moving part that must be developed, deployed and managed.
- **Increased response time** Adding the API gateway adds an additional network step
- **Increased development** time Developers have to include the gateway in the design, select supporting technologies, and learn how to use them for implementation. 

Multiple gateways can be added in the future if requirements warrant.

# API Development Guidelines
The current best practice for building REST APIs is called “API-first development” which includes identifying key services, identifying API stakeholders, and designing API contracts. Key detailed design considerations are listed below.
## API Design
Avoid creating APIs that simply mirror the internal structure of a database. The purpose of REST is to model entities and the operations that an application can perform on those entities. A client should not be exposed to the internal implementation.
Entities are often grouped together into collections (orders, customers). A collection is a separate resource from the item within the collection, and should have its own URI.
Adopt a consistent naming convention in URIs. In general, it helps to use plural nouns for URIs that reference collections. It's a good practice to organize URIs for collections and items into a hierarchy.
Also consider the relationships between different types of resources and how you might expose these associations.
Avoid introducing dependencies between the web API and the underlying data sources.


## API Operations
The HTTP protocol defines a number of methods that assign semantic meaning to a request. The common HTTP methods used by most RESTful web APIs are:
- GET retrieves a representation of the resource at the specified URI. The body of the response message contains the details of the requested resource.
- POST creates a new resource at the specified URI. The body of the request message provides the details of the new resource. Note that POST can also be used to trigger operations that don't actually create resources.
- PUT either creates or replaces the resource at the specified URI. The body of the request message specifies the resource to be created or updated.
- PATCH performs a partial update of a resource. The request body specifies the set of changes to apply to the resource.
- DELETE removes the resource at the specified URI.

| Resource | Post | Get | Put | Delete |
| -------- | ---- | --- | --- | ------ |
| /calmodels | Create new model | Retrieve all models | Bulk update of models | Remove all models |
| /calmodels/1 | Error | Retrieve details for model 1 | Update details of model 1 | Remove model 1 |
| /calmodels/1/measurements | Create new measure. for model 1 | Retrieve all measure. for model 1 | Bulk update of measure. for model 1 | Remove all measure. for model 1 |

## API Versions
How to support data source versioning with API versioning.(?)
Versioning enables a web API to indicate the features and resources that it exposes, and a client application can submit requests that are directed to a specific version of a feature or resource.
- No versioning: This is the simplest approach, and may be acceptable for some internal APIs. Significant changes could be represented as new resources or new links. Adding content to existing resources might not present a breaking change as client applications that are not expecting to see this content will ignore it.
- URI versioning: Each time you modify the web API or change the schema of resources, you add a version number to the URI for each resource. The previously existing URIs should continue to operate as before, returning resources that conform to their original schema.
- Query String versioning: Rather than providing multiple URIs, you can specify the version of the resource by using a parameter within the query string appended to the HTTP request. This approach has the semantic advantage that the same resource is always retrieved from the same URI, but it depends on the code that handles the request to parse the query string and send back the appropriate HTTP response.
- Header versioning: Rather than appending the version number as a query string parameter, you could implement a custom header that indicates the version of the resource. This approach requires that the client application adds the appropriate header to any requests, although the code handling the client request could use a default value (version 1) if the version header is omitted.
# References