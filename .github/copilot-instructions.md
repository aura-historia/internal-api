# Aura-Historia API

Aura-Historia Api is the repository for the documentation of the REST API of th aura-historia-project.
Aura-Historia is a cloud-native application with its backend being hosted entirely on AWS.
The backend is written in Rust, utilizes event-driven architecture with CQRS and Event-Sourcing and mainly relies on the following managed services:
- AWS ApiGateway
- AWS OpenSearch Service
- AWS DynamoDB
- AWS SQS
- AWS Lambda

The frontend is written in Next.js with TypeScript and ShadCN and is hosted on Vercel.
You will barely need it.

Frontend and backend each have their own repositories. They're referenced below such that you can always consult them.
- Backend: https://github.com/aura-historia/backend
- Frontend: https://github.com/aura-historia/webapp

Always reference below instructions first and fallback to search or bash commands only when you encounter unexpected information that does not match the info here.

## Working Effectively

Your sole task is it to help with documentation.
First and foremost this includes creating and updating OpenApi Specification Docs.

### OpenApi Specification Docs

We need the OpenApi Spec as Yaml inside this repository.
It's intended usage is neither for externals nor users, but for internals - especially communication between frontend- and backend-team.

**Always use OpenAPI 3.0.0** when creating or updating the specification.

Whenever you are tasked with creating, extending or updating OpenApi Specification Docs for our REST-Api, 
make sure to **traverse and inspect** the relevant code in the **repository of the backend**.
Always inspect the branch `develop` unless explicitly stated otherwise.

The OpenApi Specification should always be production-ready and as explicit as possible.
Not only looking at the implementation of the lambda-handler but also taking a look at the unit-tests may help discover desired behavior and structure.
Whenever the amount of values for certain types - be it query, body or path - is limited (e.g. enum or manual string-checking), 
make sure to only include the accepted values in the docs.
For complex objects such as those in response- or request-body, take a look at the respective Rust-Types while carefully observing it's `serde` annotations.
Moreover, always hold onto best-practices for OpenApi-Specs for REST Apis.
Do **never** add anything to the docs that's not clearly provable with evidence.

Always include as much information as possible. This especially includes but is not limited to:
- Descriptions
- Query-Parameters
- Path-Parameters
- Request-Bodies
- Response-Bodies
- Request-Headers
- Response-Headers
- Allowed values

### Changelog

This repository maintains a `CHANGELOG.md` file to document all API changes for internal communication between frontend and backend teams.

**When to Update the Changelog:**
- After any API change is merged to the `develop` branch
- When new endpoints are added
- When existing endpoints are modified
- When endpoints are deprecated or removed
- When new data types or error codes are introduced

**Changelog Format:**
The changelog follows the "Keep a Changelog" format with sections organized by date and feature:
- **Added** - New endpoints, features, or functionality
- **Changed** - Modifications to existing endpoints or behavior
- **Removed** - Deprecated or removed endpoints

**What to Include:**
For each API change, document:
- Complete endpoint specifications (method, path, parameters, body, responses)
- All new data types with property descriptions
- New error codes and their meanings
- Authentication requirements
- Examples of request/response payloads
- The merge date and feature name for tracking

**What NOT to Include:**
- Implementation details (pipeline architecture, database schemas, internal services)
- Backend code examples (Rust types, internal type names, service layers)
- Frontend code examples (TypeScript interfaces, React components, UI logic)
- Infrastructure details (AWS service names, instance types, deployment specifics)

Keep the changelog focused on REST API level documentation only. The changelog is for communicating API contract changes, not implementation or usage patterns.

**Purpose:**
The changelog serves as a communication tool, not a version control mechanism. It helps:
- Frontend developers discover and integrate new features
- Backend developers document their changes comprehensively
- Both teams track the evolution of the API over time
- Everyone understand what changed, when it changed, and why