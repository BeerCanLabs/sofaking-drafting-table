# Tiers of Intensity Publishing Source Notes

> Architecture evidence refreshed August 3, 2026. Exact cloud account IDs,
> CIDRs, security-group IDs, and source allowlists are intentionally omitted.
> The confirmed internal Traefik load-balancer DNS names are included because
> they are required to identify the stage and production EKS ingress paths.

## Product Context

- Carolina Retail Agency's 2026-27 Tiers of Intensity model shifts special
  retail funding toward a service-intensity model.
- The calculation service evaluates Assortment & Instruction, Behavior,
  Communication, Independent Functioning, and Personal Care/Health.
- The API centralizes the rules so SE, shopstar, and SOM integrations do not
  maintain separate implementations.
- Tiers of Intensity is modeled as its own Software Deployment Pattern because
  its consumers and deployment boundary extend beyond Project Recliner.

## Reviewed Sources

- Application:
  https://github.com/BeerCanLabs/sk-som-tiers-of-intensity at
  `fc727888dd2bd3be32ccde3363417c20152ac376`
- Kubernetes:
  https://github.com/BeerCanLabs/sk-k8s-deployments/tree/main/manifests/som/tier-calc-api
  at `52cfaaaf667e2e4390cd25004fac6577e736048c`
- Terraform:
  https://github.com/BeerCanLabs/sk-som-tiers-of-intensity-terraform at
  `9a133d2a2ed983aecec2a212e01f55149941b1a9`
- Jira: OPS-964449, Provision SQS and SNS in AWS, marked Done

All sources were reviewed read-only. This DRAFT change does not modify the
application, Kubernetes, or Terraform repositories.

## Development Evidence

- The API and Lambda worker target .NET 10.
- The source includes the containerized Lambda worker, queue domain,
  application and infrastructure projects, partial-batch response handling,
  HTTPS client, and queue tests.
- API routes include synchronous calculation, detailed-result, explanation,
  queue-batch, funding-tool, ruleset, health, readiness, and OpenAPI endpoints.
- The API is stateless and performs no database or remote lookup in its
  calculation path.
- The Lambda adapter invokes the API queue-batch endpoint; the API owns
  product-specific SNS publication.

## Infrastructure Evidence

- Development, QA, and staging use the LE1 non-production EKS context.
- Production uses the Chaise EKS context.
- All checked-in API Deployments start two replicas and use HPA scaling from
  two to four replicas at 70 percent CPU.
- Every deployment has readiness and liveness probes plus CPU and memory
  requests and limits.
- Traefik Ingress routes a ClusterIP Service on port 80 to container port 8080.
- The stage Lambda-to-EKS path enters Traefik on LE1 through
  `dualstack.internal-le1-ilb-traefik-ss-38425194.us-east-1.elb.amazonaws.com`.
- The production Lambda-to-EKS path enters Traefik on Chaise through
  `dualstack.chaise-xalb-ssl-traefik-fe-393192678.us-east-1.elb.amazonaws.com`.
- In both environments, the internal load balancer forwards to Traefik, which
  routes to the calculation API ClusterIP Service and EKS pods.
- ArgoCD is the GitOps deployment method for the Kubernetes resources.
- Terraform provisions SNS, SQS and DLQ resources, a containerized Lambda,
  IAM, a CloudWatch log group, and Cognito resources.

## Data Evidence

- Requests contain restricted FERPA-protected customer information and PII,
  including customer identifiers, CDP metadata, domain evidence, provider
  credentials, services, and source references.
- The API does not persist calculation data.
- SQS message retention is four days.
- DLQ retention is fourteen days.
- The queue and DLQ use SQS-managed server-side encryption and deny insecure
  transport.
- Lambda CloudWatch logs retain records for 365 days.

## Networking Evidence

The documented flow is:

1. Product event SNS topics deliver to the per-environment SQS queue.
2. SQS triggers the Lambda worker through an event-source mapping.
3. The worker calls the calculation API queue-batch endpoint over HTTPS.
4. The API publishes successful result notifications to product SNS topics.
5. Traefik routes environment ingress hosts to the API Service over HTTP
   inside EKS.
6. ArgoCD reconciles the API manifests into LE1 and Chaise.

## Known Implementation Risks

The catalog is complete about the current state, but it does not claim these
implementation gaps are fixed:

- Queue API-key authentication is optional in source and is not enabled by the
  checked-in Kubernetes configuration.
- Synchronous endpoints install no application authentication middleware.
- Terraform's production worker URL does not match the production Kubernetes
  Ingress hostname.
- Terraform emits `PublishTopics` variables while the API expects
  `ProductTopics`.
- Terraform does not explicitly enable API-owned SNS publication.
- Terraform grants SNS publication to Lambda even though source assigns that
  responsibility to the API.
- Cognito is provisioned, but the API does not validate Cognito or JWT tokens.

These gaps are tracked in the Tiers of Intensity Authentication and
Configuration Risks decision record rather than left as silent catalog
unknowns.
