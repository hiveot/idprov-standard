# IDProv - Iot Device Provisioning protocol

The IDProv protocol provides a mechanism for IoT device provisioning for machine to machine authentication, such as between IoT device and service providers such as a IoT communication Hub.

## Status

Functional. The service provisions signed certificates to IoT devices when providing a correct out-of-band secret. Certificates can also be provided by administrators or plugins.


## Overview

Aspects of the protocol:
* Provisioning of IoT devices with CA signed certificates. The issued certificate is intended for authentication of the IoT device with services.
* Use of TLS for secure communication. TLS is proven to be secure and many resources are available for client side implementation.
* OOB. Support for out of band verification. OOB provisioning is an optional way to raise the level of trust between device and provisioning service and remove the leap of faith.
* Support for bulk provisioning of OOB secrets.
* Simple. Only four JSON encoded messages in the provisioning protocol.

Out of scope:
* Discovery of the provisioning server. Discovery is a separate concern. A DNS-SD discovery plugin and client library is available for automated LAN based discovery. 
* Bandwidth optimization. Provisioning messages are few and far in between and bandwidth is therefore not a consideration. The additional complexity is not worth it.
* Memory optimization. While memory usage is an important consideration, the protocol uses standard TLS and certificates management similar to what the message communication channel uses. No special memory intensive tasks are used.
* Performance optimization. Similar to the previous points, there is not sufficient reason to make things more complex by optimizing for performance.

## Protocol Steps

The process follows the following steps:
1. Client discovers the Hub/Server address. This can be done through various means such as DNS-SD and manual configuration. This step preceeds the provisioning using this protocol.
2. Client obtains server directory and certificates. The actual provisioning can take place from a different server than that of the provider of the directory.
3. Client issues the provisioning request providing its device ID, OOB information and its public key in PEM format.
4. Server validates the client request using identity and OOB information. 
   * If this is a new request, the OOB information must match. A new certificate is generated and returned along with the certificate chain. 
   * If no OOB confirmation has been received then the status 'waiting is returned' and the client will have to try again at a later time.
   * If this is a certificate refresh, mutual authentication using the old certificate is required and no OOB information is needed. A new certificate will be issued immediately.
5. The client should request a new certificate before the existing certificate expires. 
   * Renewal requests are made using mutual authentication using the existing certificate and do not require an OOB secret.
   * If the renewal grace period has expired, the request follows the same process as a new request and must include an OOB secret and be confirmed by administrator.


## Protocol Detail

All communication between IoT client and IDProv server require a TLS connection. The client is expected to require server authentication for all except the 'directory' call. See 'leap of faith' in the next section.

### Client Get Provisioning Directory

The first action a client takes after discovering the provisioning server is to request the server directory that contains endpoint, certificate and version information. 

If the client does not yet have the server and CA certificates, it has to allow the request to take place with insecureSkipVerify and verify the provided certificate.

```http
> HTTP GET https://server:port/idprov/directory
> 200 OK
> {
>   endpoints {
>    directory: "https://server:port/idprov/directory",
>    status: "https://server:port/idprov/status/{deviceID}",
>    postOOB: "https://server:port/idprov/oobsecret",
>    postProvisionRequest: "https://server:port/idprov/provreq",
>  },
>  caCert: PEM,
>  serverCert: PEM,
>  version: "1" 
> }
```
At this stage the client takes a leap of faith that the server CA certificate is valid. The client MUST use the provided CA and server certificate with the TLS client for any further requests to the provided endpoints.

The path '/idprov/directory' is the default. A discovery service is allowed to advertise a different path. The paths of the directory endpoint MUST follow the listed endpoints.

### Device Sends Provisioning Requests

The provisioning request MUST contain the following information:
* The deviceID of the device being provisioned. This ID MUST be unique within the service.
* The device public key in PEM format. This is used to generate the device certificate. The corresponding private key and generated certificate are needed in mutual authentication so only the owner of the private key can use the certificate to connect to the servers.
* The device IP address at the time of the request. Used for logging and verification.
* The device MAC address at the time of the request. Used for logging and verification.
* The device's OOB secret 


```http
> HTTP POST https://server:port/idprov/provreq
> {
>    deviceID: {deviceID},
>    ip: {IP address of the device at the time of the request},
>    mac: {MAC address of the device},
>    oobSecret: {OOB secret. Empty if OOB is not supported by this device},
>    publicKeyPEM:  {publickey of the device}
> }
> 200 OK
> {
>    deviceID: {deviceID}
>    status: "Approved"
>    caCert: PEM,
>    serverCert: PEM,
>    clientCert: PEM
> }
```

The request returns the status object. See also the GET status method.
* request status:
  * Approved  - the request is approved and certificate is include available for download
  * Waiting - the request is waiting on OOB secret to be provided by a third party
  * Rejected - the request is rejected
* caCert, serverCert and clientCert are only included if the request is approved


Verifications by the server:
* The IP and MAC in the request matches the IP and MAC of the request sender. This can be used for logging and additional verification, for example check if only a single request is made per device.
* The oobSecret must match the oob secret of the deviceID in the device register.  
* Peer certificate:
  * In case of renewing an existing certificate with mutual authentication the peer certificate MUST have the same deviceID as the requested certificate AND must not be expired beyond the renewal grace period. The peer certificate MUST be 'iotdevice'.
  * In case of administrator or plugin requesting a certificate, the certificate OU MUST be 'plugin' or admin'. In this case no OOB secret needs to be provided.


### Server Receives OOB Device Secret

The OOB method provides the server with the deviceID and a shared secret via its out-of-band confirmation API. This API can only be accessed by an authorized user.
Authorization is provided by using a valid client certificate with the OU field set to 'admin' or 'plugin'. 


```http
> HTTP POST https://server:port/idprov/oob
> {
>   deviceID: {deviceID},
>   oobSecret: {out of band secret},
>   validUntil: {iso-date}
> }
> 200 OK
```

Where:
* {deviceID} is replaced by the device-ID when posting to the server. The deviceID must be unique for the server.
* {oobSecret} holds a string with the out-of-band secret that the device will use in its provisioning request
* {validUntil} The OOB secret is kept while the service is running or the validUntil period expires. A restart of the service MUST invalidate existing OOB secrets immediately.


### Device Requests Provisioning Status

Last, clients can obtain the provisioning status and certificate of a device. 

```http
> HTTP GET https://server:port/idprov/status/{deviceID}
> 200 OK
> {
>    deviceID: {unique ID of the device}
>    status: "Approved"
>    caCert: PEM,
>    serverCert: PEM,
>    clientCert: PEM,
> }
```
The term "{deviceID}" MUST be replaced by the device ID when posting to the server.



## Leap Of Faith

Once the device has correctly identified the provisioning server, it is not possible to intercept the traffic without detection. Device to server connections uses TLS with server verification using the CA certificate. Intercepting the traffic will lead to failure of the TLS connection.

However, that initial provisioning step requires a leap of faith, similar to SSH connections. This leap of faith is sensitive to a man in the middle attack as a rogue provisioning server can act as an intermediary server. 

This MiM attack is only possible if the rogue server installs a full intermediary server with its own certificate management, messaging relay and rewrite of Thing TD's for intercepted devices. Without this step, administrators would notice quickly that a device isn't connecting to the right server.

The real provisioning server will still be able to detect a rogue intermediary by matching the device, MAC and IP address. When more than one device has the same IP or MAC address, they are provisioned by the same machine which is unlikely. This detection can be thwarted however if the rogue intermediary can mimick multiple IP and MAC addresses.

If this too is a concern then it is best to eliminate the leap of faith dilemma by configuring the discovery service to use DNS names. At the same time monitor for invalid discovery service announcements that use IP addresses.

