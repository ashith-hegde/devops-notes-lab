# AWS Fundamentals and Global Infrastructure

## AWS Fundamentals

**Amazon Web Services (AWS)** is a cloud provider that offers computing infrastructure and services on demand.

Key characteristics:

- **On-demand provisioning:** Resources can be provisioned when needed, without first purchasing and physically installing servers.
- **Scalability:** Resources can be increased or decreased to match workload requirements.
- **Broad applicability:** AWS supports workloads across many industries and use cases.

### Common AWS Use Cases

AWS can be used for:

- Enterprise IT infrastructure
- Backup and storage
- Big data analytics
- Website hosting
- Mobile and social applications
- Gaming infrastructure

### Cloud vs. On-Premises

- **On-premises:** An organization purchases, installs, and operates physical infrastructure.
- **Cloud:** A provider makes infrastructure and services available on demand.

## Brief History of AWS

| Year | Milestone |
|---|---|
| 2002 | AWS began internally at Amazon. |
| 2003 | Amazon recognized its infrastructure expertise as a potential offering for other organizations. |
| 2004 | Amazon Simple Queue Service (SQS) was launched publicly. |
| 2006 | The public offering expanded with SQS, Amazon S3, and Amazon EC2. |
| 2007 | AWS expanded into Europe. |

Organizations cited as examples of AWS users include Netflix, Dropbox, Airbnb, and NASA.

## AWS Global Infrastructure

The core global infrastructure concepts are:

- **Regions**
- **Availability Zones (AZs)**
- **Data centers**
- **Edge locations / Points of Presence (PoPs)**

These concepts support geographic deployment, resilient architectures, and lower-latency content delivery.

## AWS Regions

A **Region** is a geographic area containing a cluster of AWS data centers. Each Region has a unique code.

Examples:

- `us-east-1` — US East (Northern Virginia)
- `eu-west-3` — Europe (Paris)
- `ap-southeast-2` — Asia Pacific (Sydney)

Most AWS services are **Region-scoped**. Resources and configurations in one Region generally need to be considered separately from those in another, depending on the service. AWS Regions are connected through AWS's global network infrastructure.

### Choosing a Region

| Factor | Why it matters |
|---|---|
| Compliance and data governance | Laws, regulations, and organizational policies may constrain where data can be stored or processed. |
| Proximity to users | Deploying closer to users generally reduces network latency. |
| Service availability | Not every AWS service or feature is available in every Region. |
| Pricing | Service prices can vary between Regions; check the relevant AWS pricing page. |


## AWS Management Console

The **AWS Management Console** is a web interface for accessing and managing AWS services.

- **Region selector:** Choose the Region in which to work with regional services.
- **Services menu:** Browse services alphabetically or by category.
- **Search:** Find services, features, documentation, knowledge articles, and blogs.
- **Console Home:** May show recently visited services, account health, cost and usage information, and tutorials. Its layout can change over time.

You do not need to be physically located in a Region to use it. Keep track of the selected Region during hands-on work: switching Regions changes the regional resources shown and can make existing resources appear to be missing.

### Global vs. Regional Services

- **Global services:** Not managed within one selected Region in the same way as regional services. Examples include IAM and Route 53.
- **Regional services:** Resources are associated with a particular Region. Examples include Amazon EC2 and AWS Lambda.

Check the documentation for a specific service to confirm its scope.

### Checking Service Availability

Before a lab or deployment, check the [AWS regional services list](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/) to confirm that the required services and features are available in your chosen Region.

## Availability Zones (AZs)

An **Availability Zone** is an isolated location within an AWS Region.

- An AZ consists of one or more discrete data centers with redundant power, networking, and connectivity.
- AZs are designed to isolate failures so that an issue in one AZ is less likely to affect others.
- AZs within a Region are connected by high-bandwidth, low-latency networking.
- Deploying application components across multiple AZs can improve availability and resilience.

Example: the Sydney Region is `ap-southeast-2`. The course material illustrates AZ identifiers such as `ap-southeast-2a`, `ap-southeast-2b`, and `ap-southeast-2c`. The number and physical layout of AZs can vary by Region. AWS does not expose every detail of the underlying data-center layout.

## Edge Locations / Points of Presence

**Points of Presence (PoPs)** include edge locations used by AWS services to bring certain capabilities closer to end users.

- They can help deliver content with lower latency by serving it from locations nearer to users.
- They are particularly relevant to content-delivery services such as Amazon CloudFront.
- Edge locations are distinct from Regions and Availability Zones.

Infrastructure counts change over time, so use current AWS documentation for up-to-date numbers.

## Quick Revision

| Concept | Main purpose |
|---|---|
| Region | Geographic area for deploying AWS resources and services. |
| Availability Zone | Isolated infrastructure location within a Region; supports resilience. |
| Data center | Physical facility that hosts computing infrastructure. |
| Edge location / PoP | Helps bring supported services and content closer to end users. |
| Console Region selector | Determines which Region's resources are shown for regional services. |
| Global service | Service not tied to one selected Region in the same way as a regional service. |

**Remember:** Regions help determine geographic placement, multiple AZs help improve resilience, and edge locations help bring content closer to users. For labs, verify the selected Region and service availability.

## Reference

- [AWS Global Infrastructure](https://aws.amazon.com/about-aws/global-infrastructure/)
- [AWS Infrastructure Map](https://infrastructure.aws/)
- [AWS Regional Services List](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/)


