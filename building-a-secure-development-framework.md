# Building a secure development framework

## Introduction

If you are reading this post, it is probable you hold the position of developer security champion in your team, serve as software factory manager or are responsible of the software supply chain security.

Regardless of your role, this post will guide you through the process of establishing a secure development framework, incorporating elements of software supply chain security and DevSecOps, both of which are integral components of the secure development framework.

## What is a secure development framework

A secure development framework outlines the steps and requirements that each team involved in the software development process within a company must adhere. While it typically does not prescribe specific tools, it focuses on requirements due the changing nature of software development, This implies that a framework cannot mandate the use of a particular tool, as such tools are often language-specific, and enforcing their use could hinder the company's growth and adaptability.

There are many existing frameworks out there, but none specific to your business needs. How you build our own framework is what makes the difference of just passing regulations or really addressing software security seriously.

Existing frameworks such as SSDF defines basic requirements, but you need to adjust to specific languages, culture, business logic, legal regulations or standards, etc

This guide will be a resume of the basics every framework should contain along with a few more advanced cases to improve the supply chain security as well.

I'm going to wrap the phases for this framework in the following list, not necessarily in any order.

* Education and culture
* Business requirements
* Design
* Developer environment
* Source code management
* CI/CD
* Artifact management
* Vulnerability management, monitoring and maintenance

Any of this phases should and must be decomposed into many other inner phases, they're typically the steps almost any modern SDLC have.

Let's review the phases and their specific focus on the S-SDLC

### Education and culture

The primary factor in developing secure software is whether developers are aware of potential vulnerabilities and, more importantly, if they are committed to prioritizing security over convenience in the development process.

Even the most skilled and efficient developer cannot be considered a competent asset to any reputable software company if they lack knowledge about SQL injection attacks, do not know how to address them, or, crucially, lack the willingness to learn how to resolve such issues.

This does not mean, every developer should be an expert penetration tester. They just needs basic understanding of common vulnerabilities such as OWASP top ten.

The role of the company and security champions is to provide resources and teach devs from the beginning of their software developer career.

There are many free resources as well as paid training, I suggest first start with free courses from The Linux Foundation, reading OWASP documentation, blogs specific for the language in use, YouTube videos are good source of knowledge and permit/encourage senior developers teach graduates and mid-seniority roles.

Typically you will encounter some people that do not want to apply security first and will attempt to ignore the rules, this can be from the more seniors to the recent graduates. Usually both think they know everything and do every piece of code perfect (Spoiler: Don't), don't let this type of people mark the security about your software and by result the business reliability.

How to deal with this profile? Let them know there are priorities, don't rush too much on the release cadence times so they can start applying security, show them how their insecurity code could be exploited, etc.

### Business requirements

Next area is business requirements, talk to the product owners, understand what are their needs and make a clear understanding what are the project's requirements in terms of security and risk management.

There may be some projects that require minimal level of security requirements while other will need high levels of security.

If the product owner or manager doesn't know what they need, talk to them, analyze the business logic and go deeper until all of you find what are the business needs.&#x20;

Now we're going into the technical aspects, this is the part everybody want to start, but if you don't know the business requirements you cannot ever design a secure application.

### Design

At this step, you should know what are the business requirements. Now you start thinking about languages, APIs, databases, etc.

First step in the design phase is:

#### Secure by design

Start from the beginning with security, don't apply patches once everything is working. This will reduce money expenses as well as avoid keeping insecure stuff in the SDLC.&#x20;

Ensure TLS is enforced early on development, strong authentication is used in place, proper network segregation is implemented and try to implement zero trust architecture unless impossible.

#### Threat modeling

Along with the software design, threat modeling should be applied early. Knowing what the boundaries and risks are from the start will allow to focus on the most important ones.

Threat modeling also serves as method to better understand the application logic and think on improved software features, not just security wise.&#x20;

### Developer environment

During this phase, software developers will have reticence for some changes.

We will define if developers should have full permissions on their development environments or limited permissions, we need to define if external artifacts should be reachable or limited to certain packages in an internal repository.

In the most critical environments, laptops will have no extra privileges and packages should come from an internal registry very very limited where packages are reviewed, tested and validated security wise.

Find the best for your needs, do not allow free will to hinder your decision; make a choice in a intermediate spot.

Limiting what an user can do, also will improve security of their laptops and by extension the software security. Having a common registry with static packages, but also a proxied artifact repository to store cache and do automatic testing of packages will also help improve software security. As example of this could be a Jfrog artifactory proxying npm official repository with xray configured to analyze and block malicious packages, only used in development environment while stable packages are promoted to the static repository.

Next part for developers is define a company or team standard for software styling, this means keep a common code style. Every developer should adhere to this internal standard and must be verified during Pull/Merge request pipelines.

To make it easier and less hardware intensive in CI/CD, enforce usage of `pre-commit-hooks`, so changes and failures are raised early before committing to remote repository.

In this pre-commit-hooks, apply security as well. Add the language-specific SAST tool to verify insecure code is never pushed.

In the same manner, add secret detection checks to avoid passwords and tokens are stored in git history.

### Source Code Management

Source code management takes an important role to secure software, not just by remote shared development but avoiding insecure or malicious code ever get to main branches.

To protect SCM, first enforce usage of commit signing, this will ensure committed code was by the expected author.

Apply strict branch rules:

* No one is allowed to directly push to main branches
* Disable force push even for admins

Ensure merge requests rules:

* Code should be reviewed by a minimum of 2 unless is a small team
* Cannot approve self merges
* CI/CD pipelines should pass correctly
* If repository code if shared between teams, ensure CODEOWNERS rules apply correctly
* Pipeline code cannot be modified in same merge (avoid leaking or introducing unexpected data)

### CI/CD

An important part of the supply chain security are CI/CD pipelines, this need to be secured and logged as production/critical services, since they serve the purpose of building the important software artifacts.

For this purpose, CI/CD should have the following rules:

* Every CI job should be isolated from each other, an example is use containers for each job
* Runners should be hardened with minimal permissions and network access
* No privileged process can be executed from a CI/CD job
* No cache is stored, to avoid cache poisoning
* Git strategy should be clone instead of fetch, this also avoid cache poisoning on the git side
* Ensure container image policy is always, this avoids usage of cached images without permissions.

In the more strict environments the additional rules could be added:

* Network isolation
* Duplicated builds on different systems

Along with this general rules, one of the key rule is **ensure no secrets or sensitive data is executed in unverified (not protected branches) pipelines.**

This means that in pipelines from merge requests or push to feature branches, no secrets, no sensitive data and no artifact build is done without passing the verification on the merge request or without repository owner manual approval, this will avoid malicious actors to run a pipelines exposing secrets on forks or in development branches.

For source code security we will see 3 methods:

#### Static testing

This is where a specific OSS or commercial tool executes heuristic analysis over the source code to verify its logic is not insecure. It does not execute the application.

Examples are validating that no user defined unprocessed variable is passed into a SQL query.

During static testing usually are executed unit and linting testing too.

#### Dynamic testing

During dynamic tests, the application is executed and is validated its real behavior.

Examples of this are DAST to verify security testing, API testing, FUZZ testing, functional testing, etc.

#### Dependencies testing

We must ensure no insecure package is provided or used by our software.

SCA or software composition analysis is made during CI to ensure packages with vulnerabilities are not used, or update to newer version.

Typically use a bot to automatically update dependencies by MR when new releases are available, this help keep software up to date and avoid large headaches when upgrading all the project dependencies.

Keep software dependencies pinned by version or hash, to avoid dependency attacks or unexpected behaviors with new versions.

### Artifact management

Once the application is tested, verified and approved, software artifacts are usually created.

This is an critical part to secure, since the artifacts are the applications that will be running in our production or customer servers. They should have the code it is expected to have.

How do we ensure this?

* Artifact signing
* Build attestations
* Provenance
* SBOM and VEX creation

With artifact signing we ensure the application is created by the expected process (CI/CD runner), with attestations we can verify the CI/CD build is created by the expected source code, expected input data and expected output binaries.

With SBOMS and VEX, if we sell or release software outside the company we can notify consumers what software is included and its dependencies, as well as what are the vulnerabilities and if they can be exploited, not applicable or fixed by other methods. This gains value as software transparency and provides consumers the view we take security seriously.

SBOMs also serve as a method to easily identify components where a specific package with a vulnerability is present, easing vulnerability patching.

### Vulnerability management, monitoring and maintenance

Once we have a product ready to be released (actually this work should be done since the beginning), we must ensure that no vulnerability is exploitable, and if so, have a logging record to be able to trace where,when and how happened.

For this reason, logging should be applied to all requests with user ID, IPs and request data (except sensitive). If its a web application a WAF service should be added in place to limit common attack vectors.

From software security perspective, general rules should be added:

* Vulnerability management
* Vulnerability disclosure programs
* Risk management
* Pen testing
* Runtime analysis for unexpected behavior
* Threat intelligence

## Existing frameworks

Existing frameworks and standards are all over the net, the most notorious are:

* NIST's Secure Software Development Framework (SSDF)
* BSA Framework for Secure Software
* Microsoft’s Security Development Life cycle

## Example pipeline for a python project

Here is an example of a logical pipeline for a python project, with example tooling.

```yaml
Test:
  Linting:
    - pep8
    - pylint
  SAST:
    - bandit
    - semgrep
  secrets:
    - git-secrets
    - trufflehof
  unit tests:
    - unittest
    - pytest
Build:
  Functional test
  API testing
  UI testing
  DAST:
    - OWASP ZAP
    - burpsuit
    - Custom implementation
Release:
  Package build:
    - python -m build
  Package sign:
    - cosing
    - notary
  Signature verification
  Artifact upload
Promote:
  Artifact release:
    - pypi
    - git artifacts
```
