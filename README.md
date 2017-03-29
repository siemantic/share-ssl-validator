# Farmer DNS Service

## Purpose

The purpose of the Farmer DNS Service is to manage the lifecycle of DNS for farmers on the Storj network.

## Document Conventions & Terms

+ When referring to LetsEncrypt, we will use the acronym LE
+ The terms Share user and Farmer are used interchangably and mean the same thing
+ A Farmer's NodeID refers to the farmers Storj network address
+ Farmer SSL URL or Share SSL URL refers to a DNS A record created by the Validation Service in the format of [nodeid].storj.farm

## Description

The Farmer DNS Service handles DNS ACME validation for LE, helps a farmer verify that they are serving SSL properly, and manages DNS entries for farmers.

## Scope

### Farmer DNS Service

This service should only be responsible for the following:

+ Validating a farmer's NodeID
  + Validated by having the farmer sign the value of a TXT record with their private key
+ Creating a DNS A record for the Farmer using their nodeid as the hostname on the storj.farm domain
+ Creating a DNS TXT record using the gcloud API
+ Confirm that a farmer is reachable over HTTPS at the DNS entry we created.

The intention here is to separate all knowledge of SSL and LE from the Farmer DNS Service, making the farmer responsible for it's own certificate. Since we are a TLD, this seems to be the most natural approach to things.

### Storj Share

The additions to the Storj Share service should only include the following:

+ Submitting a request to LE requesting a certificate for the DNS name [nodeid].storj.farm
  + the storj.farm domain should be a configurable parameter
  + This should use the node.js lib: https://www.npmjs.com/package/letsencrypt
+ Sending an HTTP request with the value of the TXT challenge given by LE signed with the farmer's private key
+ Saving the certificate provided by LE after successful validation
+ Setting up a local webserver capable of serving over SSL using the certificates retrieved from LE
+ Request that the Farmer DNS Service validate that it is reachable over HTTPS at it's DNS entry
+ Accept an HTTPS request from the Farmer DNS Service for validation

## Implementation

### Process Overview

+ The Storj Share farmer requests a certifcate from LE
+ LE responds with a set of ACME challenges, one of which is to create a DNS TXT record
+ The Storj Share farmer signs the TXT challenge with its private key
+ The Storj Share farmer sends the signature and the challenge to the Farmer DNS Service
+ The Farmer DNS Service validates the signature matches the farmer's NodeID
+ The Farmer DNS Service registers the TXT record to [nodeid].storj.farm on Google DNS
+ The Farmer DNS Service registers the A record to [nodeid].storj.farm on Google DNS
+ The Farmer DNS Service returns 200 letting the farmer know the DNS registers were created
+ The Farmer tells LE that it has completed one of the ACME challenges
+ LE confirms the challenge was completed and issues a certificate
+ The Farmer asks the Farmer DNS Service to validate that it is configured properly
+ The Farmer DNS Service makes an HTTPS request to the Farmer to validate it's configuration
+ The Farmer DNS Service responds with a 200 to the validation request indicating that the Farmer is properly configured

### Code bases affected

* https://github.com/Storj/storjshare-daemon - Handle LE and Farmer DNS Service registrion
* Implement a new microservice which is responsible for managing the lifecycle of DNS for nodes registering under the `storj.farm` TLD. (Farmer DNS Service)
