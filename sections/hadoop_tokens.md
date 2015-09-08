<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  
   http://www.apache.org/licenses/LICENSE-2.0
  
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
  
# Introducing Hadoop Tokens

So far we've covered Kerberos and *Kerberos Tickets*. Hadoop complicates
things by adding another form of delegated authentication, *Hadoop Tokens*.


### Why does Hadoop have another layer on top of Kerberos?

That's a good question, one developers ask on a regular basis —at least once
every hour based on our limited experiments.

Hadoop clusters are some of the largest "single" distributed systems on the planet
in terms of numbers of services: a YARN cluster of 10,000 nodes would have
10,000 hdfs principals, 10,000 yarn principals and the principals of the users
running the applications. That's a lot of principals, all talking to each other,
all having to talk to the KDC, having to re-authenticate all the time, and making
calls to the KDC whenever they wish to talk to another principal in the system.

Tokens are wire-serializable objects issued by Hadoop services, which grant access
to services. Some services issue tokens to callers which are then used by those callers
to directly interact with other services *without involving the KDC at all*.

As an example, The HDFS NameNode has to give callers access to the blocks comprising a file.
This isn't done in the DataNodes: all filenames and the permissions are stored in the NN.
All the DNs have is their set of blocks. 

To get at these blocks, HDFS gives an authenticated caller a *Block Tokens* for every block
they need to read in a file. The caller then requests the blocks of any of the datanodes
hosting that block, including the block token in the request.

These HDFS Block Tokens do not contain any specific knowledge of the principal running the
Datanodes, instead they declare that the caller has stated access rights to the specific block, up until
the token expires.


```
public class BlockTokenIdentifier extends TokenIdentifier {
  static final Text KIND_NAME = new Text("HDFS_BLOCK_TOKEN");

  private long expiryDate;
  private int keyId;
  private String userId;
  private String blockPoolId;
  private long blockId;
  private final EnumSet<AccessMode> modes;
  private byte [] cache;

  ...
```

Alongside the fields covering the block and permissions, that `cache` data contains 
 
 ## Tickets vs Tokens
 
 
| Token                | Function                   |
|--------------------------|----------------------------------------------------|
|  Authentication Token | Directly authenticate a caller. |
| Delegation Token | A token which can be passed to another process. |
 
 
 ### Authentication Tokens
 
 Authentication Tokens are explicitly issued by services to allow the caller to
 interact with the service without having to re-request tickets from the TGT.
 
 When an Authentication Tokens expires, the caller must request a new one off the service.
 If the Kerberos ticket to interact with the service has expired, this may include
 re-requesting a ticket off the TGS, or even re-logging in to kerberos to obtain a new TGT.
 
 As such, they are almost equivalent to Kerberos Tickets -except that it is the 
 distributed services themselves issuing the Authentication Token, not the TGS.
 
 ### Delegation Tokens
 
 A delegation token is requested by a client of a service; they can be passed to
 other processes. 
 
 When the token expires, the original client must request a new delegation token
 and pass it on to the other process, again.
 
 What is more important is: *delegation tokens can be renewed before they expire.*
 
 This is a fundamental difference between Kerberos Tickets and Hadoop Delegation Tokens.
 
 Holders of delegation tokens may renew them with a token-specific `TokenRenewer` service,
 so refresh them without needing the Kerberos credentials to log in to kerberos.
 
 More subtly
 
 1. The tokens must be renewed before they expire: once expired, a token is worthless.
 1. Token renewers can be implemented as a Hadoop RPC service, or by other means, *including HTTP*.
 
 For the HDFS Client protocol, the client protocol itself is the token renewer. A client may
 talk to the Namenode using its current token, and request a new one, so refreshing it.
 
 In contrast, the YARN timeline service is a pure REST API, which implements its token renewal over
 HTTP/HTTPS. To refresh the token, the client must issue an HTTP request (a PUT operation, interestingly
 enough), receiving a new token as a response.
 
 Other delegation token renewal mechanisms alongside Hadoop RPC and HTTP could be implemented,
 that is a detail which client applications do not need to care about. All the matters is that
 they have the code to refresh tokens, usually code which lives alongside the RPC/REST client,
 *and keep renewing the tokens on a regularl basis*. Generally this is done by starting
 a thread in the background.
 
 
 
 
 
 ## Token Propagation in YARN Applications
 
 
 
 
 Imagine a user deploying a YARN application in a cluster, one which needs
 access to the user's data stored in HDFS. The user would be required to be authenticated with
 the KDC, and have been granted a *Ticket Granting Ticket*; the ticket needed to work with
 the TGS. 
 
 The client-side launcher of the YARN application would be able to talk to HDFS and the YARN
 resource manager, because the user was logged in to Kerberos. This would be managed in the Hadoop
 RPC layer, requesting tickets to talk to the HDFS NameNode and YARN ResourceManager, if needed.
 
 To give the YARN application the same rights to HDFS, the client-side application must
 request a Delegation Token to talk to HDFS, a key which is then passed to the YARN application in
 the `ContainerLaunchContext` within the `ApplicationSubmissionContext` used to define the
 application to launch: its required container resources, artifacts to download, "localize",
 environment to set up and command to run.
 
 The YARN resource manager finds a location for the Application Master, and requests that
 hosts' Node Manager start the container/application. 
 
 The Node Manager uses the "delegated HDFS token" to download the launch-time resources into
 a local directory space, then executes the application.
 
 *Somehow*, the HDFS token (and any other supplied tokens) are passed to the application that
 has been launched.
 
 The launched application master can use this token to interact with HDFS *as the original user*.
 
 The AM can also pass token(s) on to launched containers, so that they too have access to HDFS.
 
 
 The Hadoop Name Node does not need to care whether the caller is the user themselves, the Node Manager
 localizing the container, the launched application or any launched containers. All it does is verify
 that when a caller requests access to the HDFS filesystem metadata or the contents of a file,
 it must have a ticket/token which declares that they are the specific user, and that the token
 is currently considered valid (based on the expiry time and the clock value of the Name Node)


## Determining the Kerberos Principal for a service

1. Service name is derived from the URI (see `SecurityUtil.buildDTServiceName`)...different
services on the same host have different service names
1. Every service has a protocol (usually defined by the RPC protocol API)
1. To find a token for a service, client enumerates all `SecurityInfo` instances; these
return info about the provider. One class `AnnotatedSecurityInfo`, examines the annotations
on the class to determine these values, including looking in the Hadoop configuration
to determine the kerberos principal declared for that service (see [IPC](ipc.html) for specifics).

## Delegation Token internals