---
layout: post
title: Authorization
date: 2025-07-01
description: General Info on Authorization
tags:
categories: elixir
giscus_comments: true
---

Permission checking or authorization is one of the common things (along with database design) that startups have troubles with (if they are lucky enough to move past the startup phase). I wanted to gather together some general information for engineers working on an existing authorization system or implementing a new one.

## Basics

- Authorization is not authentication. They are related but should not be considered to be one and the same thing. Authentication is verifying that a subject is who they are purporting to be. That has to happen before making authorization decisions. Discussions of authorization can get confusing because you can hear Auth and think that the discussion is on authorization but it's instead on authentication. A good way of differentiating these activities is to use AuthN for authentication and AuthZ for authorization.
- Authorization is deciding whether or not a subject (user / program as identified by authentication) can perform an action on an object (resource / user)

## Acronyms

Like any technical area authorization has it's own set of acronyms.

The three most common approaches to authorization:

- Role Based Access Control (RBAC). A set of roles are associated with permissions to perform some action on some resource. Users (an actual user or a program) are associated with these roles. This is a popular way of managing authorizations. It's advantages are that it is not too hard to understand and is fairly easy to administer.
- Attribute Based Control (ABAC). Organizations can find that RBAC is too limiting for complex data models. In ABAC you consider one or more security attributes to decide on authorization. There are four main categories of attributes:
  - subject (user) attributes. For example, username, age, id, job title, organization, job role
  - resource (object) attributes. Indicate what would be acted upon.
  - action. Indicates what action the subject wishes to take (view, modify, delete)
  - environmental. This is a common consideration in ABAC systems that you do not see in RBAC systems. Time, device, location are all environmental attributes.
- Relationship-based Access Control (ReBac). The concept behind ReBAC authorization systems is that by following a chain of relationships one can determine access. Both Facebook and Google are using a form of ReBac. Google's ReBAC system is implemented in a tool called Zanzibar. Zanzibar implements not just a ReBAC system but it covers infrastructure that is particular to Google and utilizes "in house" tools. Google published a paper describing the system and a number of companies have built implementations of the system described in the paper using tools like Cockroach DB.

Other common authorization acronyms:

- AuthN - authentication
- AuthZ - authorization
- CIAM - Customer Identity and Access Management. This is used to describe the combination of AuthN and AuthZ.
- Discretionary Access Control (DAC) - files and directories had read, write and execute permissions. This was the development of the earliest Unix system. The core concept of permissions involves three categories: owner, group, and others, each with read, write, and execute permissions. Access control lists (ACL) was used as a mechanism to hand out permissions.
- Mandatory Access Control (MAC) - Honeywell's SCOMP and NSA's Blacker, focused on achieving multilevel security (MLS) for military applications. This system applied security labels to subjects (users/processes) and objects (files/data). For example, labels could be "Top Secret," "Secret," "Confidential". An administrator places objects into these labeled categories. It was meant to protect classified information.
- Access Control List (ACL). This is a means of controlling access to files, directories and network endpoints. The cloud service providers use ACL's in their cloud networking. For example, in Azure there are Network Security Groups (NSG) and in AWS there are Security Groups and Network Access Control List (NACL). These are both implementations of an ACL.
- Identity and Access Management (IAM) is a security framework that manages digital identities and controls user access to resources. It ensures that the right individuals have appropriate access to the necessary resources when they need them, enhancing security and operational efficiency. IAM involves establishing user identities, authenticating them, and authorizing access to specific resources, like applications, data, and devices.

## Problem Areas

- Authorization ends up being using throughout a full stack application: backend services, Javascript/Typescript Web UI, smart phone applications, However, each is on it's own release cycle and, generally, use their own tools and languages.
- Subject authorizations can change over time
- Resources can change over time
- Allowed actions on resources can change over time
- Authorization has to be speedy enough to not be noticeable
- Customers need to understand how to use what you provide in order to safeguard their system
- Developers need to know how to add new permissions as new functionality is added
- Marketing and sales need to know that the authorizations provided align with the sales goals for the company

## Feature Flags and Authorization

Feature flags are not authorization. But the topics are related. Feature flags are (traditionally) used to hide a new feature from the majority of users in a system. A few users are allowed access to the new feature to ensure that it works as expected. A rollout of some kind occurs to provide the feature to a larger and larger percentage of users. Ultimately, the new feature is completely released and the feature flag goes away.

That's one of the key differences between feature flags and authorization. A created feature flag will ultimately not exist once it's served its purpose. Authorization, on the other hand, is fundamental to business processes. It's not a short-lived thing.

Generally, in a code path, a feature flag is checked before an authorization check occurs. This is not required but is the general practice since it provides an optimization. There's no sense checking authorization if a user should not access a code path.

It's important to review feature flags to ensure that they are not used as a replacement for authorizations. A feature flag that exists for an extended period of time is a sign that your authorization system is potentially subverted.

## Scrolling

If an end user is looking at a large set of data does the code have to authorize each and every item that they are examining. If this is the case it presents scaling problems that make it impossible to present an efficient system.

For example, if an end user is examining a spreadsheet with hundreds of thousands of cells a system would be unusable if a view authorization was needed for every single cell in the spreadsheet. Authorization systems must be configured to authorize at a higher level. If your system must authorize at an extreme level of granularity then it most likely will fail in various ways.

## Authorization and Customer Charges

If your system is sold to customers and charges are based on "level of access" then you must continuously be aware of authorization overlap. That is, you may have an RBAC system that authorizes a "user" to perform certain actions and a "manager" to perform more actions on resources. However, as new resources and actions are introduced in the system or a monolith is split into microservices the original intent can easily be lost. A "user" may be able to do as much (if not more) than a "manager". Customers can discover this and "game the system" so that they are charged less than what was planned when the system was first designed.

It's important to review authorizations as new resources or actions are added to an existing system to ensure that authorizations align with a billing model.

## Authorizations and Microservices

Microservices provide additional authorization challenges. An end user wants to perform an action on some resource. Is this allowed? The problem is that the original receiver of the request may turn around and make calls to one or more microservices in order to accomplish the request. Do each of these microservices perform authorization checks? Are there conflicts that can arise where an operation is "half done" because one microservice authorizes but another does not? Are each of the microservices "in sync" on what is authorized and what is not?

It's important to look at the resources and actions on those resources and determine exactly what occurs to accomplish the action. You need to determine whether you have any instances in your system where an action can be "half done". These scenarios must be removed from your system.

## References

- [NIST Model for Role Based Access Control (RBAC)](https://tsapps.nist.gov/publication/get_pdf.cfm?pub_id=916402)
- [Guide to Attribute Based Access Control (ABAC) Definition and Consideration](https://nvlpubs.nist.gov/nistpubs/specialpublications/nist.sp.800-162.pdf)
- [AuthZed Annotated Google Zanzibar Paper (ReBAC)](https://zanzibar.tech/)
- [Access Control Requirements for Web 2.0 Security and Privacy (ReBAC)](https://www.researchgate.net/profile/Carrie-Gates-2/publication/240787391_Access_Control_Requirements_for_Web_20_Security_and_Privacy/links/540e6f670cf2d8daaacd4adf/Access-Control-Requirements-for-Web-20-Security-and-Privacy.pdf)
