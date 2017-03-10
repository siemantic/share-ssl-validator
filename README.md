# Storj Share SSL Validation Service

## Purpose
The purpose of the Share SSL Validation Service is to manage the Letsencrypt DNS Challenge Validation process for a Storj Share instance.

## Document Conventions & Terms
+ When referring to LetsEncrypt, we will use the acronym LE
+ The terms Share user and Farmer are used interchangably and mean the same thing
+ Farmer Public Address or farmer_public_address refers to the farmers Storj network address
+ Farmer SSL URL or Share SSL URL refers to a DNS A record created by the Validation Service in the format of [farmer_public_address].storj.farm

## Description
The Validation Service manages DNS Challenge Validation for LE, confirms that a Farmer is serving SSL properly, and manages the state of SSL vs NOSSL for Farmers.

## Scope

### Validation Service
This service should only be responsible for the following...

+ Taking a challenge string from a user on the Storj netowrk
  + The authenticity of the user is validated using the TriggerManager mechanism from Storj Core
+ Creating a DNS A record for the Farmer using their public address as the hostname and the storj.farm domain
+ Creating a DNS TXT record using the gcloud API
  + This portion could be implemented such that we can add other dns providers in the future but that is not the primary focus
+ Wait for confirmation that the user has successfully validated and received their SSL certificates
+ Confirming that the users HTTPS server is reachable, working properly, and using a valid certificate
+ Managing the state of SSL vs NOSSL for the Share user
+ Clean up TXT records either after success confirmation or a set timeout

### Storj Share
The additions to the Storj Share service should only include the following

+ Submitting a request to LE requesting a certificate for the DNS name [farmer_public_addr].storj.farm
  + the storj.farm domain should be a configurable parameter
  + This should use the node.js lib: https://www.npmjs.com/package/letsencrypt
+ Submitting a Trigger to the Storj network requesting SSL using the challenge string received from LE, the Farmers IP address and its public address
+ Saving the certificate provided by LE after successful validation
+ Setting up a local webserver capable of serving over SSL using the certificates retrieved from LE
+ Notifying the Validation Service that it has obtained a certificate and that it is working properly
+ Accepting a request from the Validation Service over the HTTPS server confirming that it's SSL is properly configured


## Process Flow Details

## Validation Service
### Initial SSL Requset (getssl)
+ Brings up a node on the network using the Storj Core lib
  + Node property is wildcard
  + Behavior is the name of the trigger `getssl`
  + Function is how to handle the request
+ Listens for a `getssl` Trigger on the Storj network
  + [TriggerManager](https://storj.github.io/core/TriggerManager.html)
  + Message is already verified by Storj Core if it has gotten to this point
+ Creates a DNS A record
  + Host: [farmer_public_address].storj.farm
  + IP: The farmers publically reachable IP address
+ Here we could check the DNS entry in a loop until DNS has propagated or leave it up to the farmer (prob best to leave it up to the farmer)
+ Takes the challenge string and creates a TXT record
  + There is an implemented example in a bash script [here](https://github.com/spfguru/dehydrated4googlecloud) for clarification
+ Responds to the Farmer letting it know that its DNS records have been created

### SSL Connectivity Confirmation Request (confssl)
+ Listens for the `confssl` trigger message on the network
+ Attempts to reach out to the farmer over it's DNS address via SSL
+ Confirms SSL Success by validating the farmers certificate or Failure
+ Responds to the farmer reporting Confirmation or Failure
+ Captures the state of the Farmers SSL

## Share SSL
### Initial SSL Request (getssl)
+ Farmer comes online (for the first time, or not the first time but has no ssl)
+ Farmer requests certificate from LE using the node.js [Letsencrypt Module](https://www.npmjs.com/package/letsencrypt)
  + The DNS validation method should be used
  + LE responds with a challenge string
+ Farmer receives token from LE to be set in DNS TXT record
+ Farmer reaches out to Validation Service via the `getssl` trigger
  + Provide farmer id
  + Provide IP addr
  + LE challenge string
+ Waits for confirmation from the Validation Service saying that the DNS A record and DNS TXT records have been created
  + This may not be necessary but might give more insight into any issues that may arrise
+ Farmer should receive confirmation and certificate from LE
+ Have some sort of timeout, retry, and failure logic

## Notes
The ordering of things here depends on time that we are given between requesting the certificate from LE and how long LE will continue to check for the DNS TXT record. We should trim down the number of back and forth requests as much as possible as long as it does not affect the certificate retrieval. If we need to buy ourselves more time, we can ensure that the DNS A record is created and propagated prior to sending the request to LE.
