# IDProv - Iot Device Provisioning protocol

The IDProv protocol provides a mechanism for IoT device provisioning for machine to machine authentication, such as between IoT device and service providers such as a IoT communication Hub.

## Status

Draft. This standard is still under development. 

A reference implementation in goland is provided by the [idprov-go](https://github.com/wostzone/idprov-go) project. This implementation is used to further test and validate that the protocol meets the intended use-cases.

## Overview

Aspects of the standard:
* Discovery of the provisioning server.
* Provisioning of IoT devices with CA signed certificates.
* The issued certificate is intended for authentication of the IoT device with IoT service providers.
* HTTP over TLS is used to for communication. TLS is proven to be secure and many resources are available for client side implementation.
* Out of band verification. OOB provisioning is used to raise the level of trust between device and provisioning service and remove the leap of faith.
* Support for bulk provisioning of OOB secrets.
* Simple. Only four JSON encoded messages in the provisioning protocol.

Out of scope:
* Bandwidth optimization. Provisioning messages are few and far in between and bandwidth is therefore not a consideration. The additional complexity is not worth it.
* Memory optimization. While memory usage is an important consideration, the protocol uses standard TLS and certificates management similar to what the message communication channel uses. No special memory intensive tasks are used.
* Performance optimization. Similar to the previous points, there is not sufficient reason to make things more complex by optimizing for performance.


### Discovery

The IDProv standard supports several discovery mechanisms that provide the device with the IDProv server connection information. 

1. DNS-SD: 
The provisioning server publishes a DNS-SD service record on the local network. Clients on the same subnet can receive this record and determine how to connect to the server.

Included are:
* service name is "idprov"
* service type is "_idprov._tcp"
* IP address of the server
* TXT record included with the path to the 'get directory' endpoint: TXT: "directory=/idprov/directory"

2. NFC: 
This method is under investigation.
Supporting devices contain an NFC receiver that receive the provisioning server address and CA certificate.

3. Bluetooth/Wifi:
This method is under investigation.
Supporting devices pair with the server when the pairing button is pressed. When a connection is established the provisioning server address is received. This could be extended to include issuing of a certificate. 

4. Manually:
A devices with terminal input can receive a configuration that contains server address, CA certificate, device ID, and a client certificate. To renew the certificate the client still needs to post a provisioning request. The renewal of an existing certificate takes place autonomously.


## Out-Of-Band Secret

If certificates can not be pre-provisioned on the device, an out-of-band secret is used to verify that the client and server are who they claim to be. 

The OOB secret is only needed to establish the inital trust between client and server. It can be omitted for devices that renew a valid client certificate. OOB secrets are treated as one-time-tokens. The server discards the secret after a successful request. Depending on the device hardware, a new secret can be generated on each OOB provisioning request or a fixed secret can be re-used.

To ensure that OOB secrets cannot be intercepted while a leap-of-faith is in progress, the provisioning request only contains the hash of the secret. Without the actual secret a man-in-the middle attacker cannot generate a valid hash and obtain a certificate.

Different methods can be used to retrieve the OOB secret from the device and send it to the server. The server has an administration API that is used to transfer the retrieved deviceID and secret. The retrieval method can vary depending on the device capability. For example:

A. NFC. A scanner scans the NFC tag on the devices and reads the device ID and OOB secret
B. QR Code. A scanner scans a QR code sticker on the devices and reads the device ID and OOB secret
C. Bluetooth. The device pairs with the server and the server requests the device ID and OOB secret.


## IDProv Protocol

### Process Steps

The provisioning process follows the following steps:
1. Client discovers the IDProv server address. This can be done through various means such as DNS-SD (recommended) or manual configuration.
2. Client obtains server directory and CA certificate. 
3. Client issues the provisioning request providing its device ID, OOB information and its public key.
4. Server validates the client request using identity and OOB information. 
   * If this is a new request, the OOB information must match. A new certificate is generated and returned along with the certificate chain. 
   * If no OOB confirmation has been received then the status 'waiting is returned' and the client will have to try again at a later time.
   * If this is a certificate refresh, mutual authentication using the old certificate is required and no OOB information is needed. A new certificate will be issued immediately.
5. The client should request a new certificate before the existing certificate expires. 
   * Renewal requests are made using mutual authentication using the existing certificate and do not require an OOB secret.

All communication between IoT client and IDProv server require a TLS connection. The client is expected to require server authentication for all except the 'directory' call. 


### Get Provisioning Directory

The first action a client takes after discovering the provisioning server is to request the server directory that contains information for using the protocol.
* A list of endpoints by name. directory, status, postOOB, and postProvisionRequest are required. Additional endpoints are optional as long as their names are standardized.
* The CA certificate of the provisioning server.
* The protocol version. Currently "1". All 1.x versions of the protocol are backwards compatible.

If the client does not yet have the server and CA certificates, it has to allow the request to take place with insecureSkipVerify and verify using the OOB secret.

```http
> HTTP GET https://server:port/idprov/directory
> 200 OK
> {
>   endpoints {
>     directory: "https://server:port/idprov/directory",
>     status: "https://server:port/idprov/status/{deviceID}",
>     postOobSecret: "https://server:port/idprov/oobSecret",
>     postProvisionRequest: "https://server:port/idprov/provreq",
>   },
>   services {
>      messageBus: "mqtts://server:port/",
>      thingDirectory: "https://server:port/directory",
>   },
>   caCert: PEM,
>   version: "1" 
> }
```

The client MUST continue using the provided CA certificate with the TLS client for any further requests to the provided endpoints. To ensure the server providing the directory is the same as the server answering the provisioning request. The server is still untrusted until the provisioning request response signature is verified.

The path '/idprov/directory' is the default. A discovery service is allowed to advertise a different path.

The services section list services that can be used by clients with the provided  certificates. The CA will validate the server while an issued client certificate authenticates the client. Without a client certificate a username/password authentication must be used. The names are defined in the appendix.

The default port for the idprov server is 43776 (IDPro)

### Server Receives OOB Device Secret

The out-of-band secret endpoint is used to provide the server with the deviceID and a one-time secret. This endpoint requires a TLS connection with a valid admin or plugin OU certificate. 
Authorization is provided by using a valid client certificate with the OU field set to 'admin' or 'plugin'. The administrator must use a valid client and CA certificate to ensure mutual trust. 

Constraints:
* One time use. Once the secret is used in a successful provisioning request, it is removed from the server. Depending on the client capabilities it is recommended to  generate a new secret each time provisioning is attempted. A device can re-use the same secret when re-provisioning but it will need to be applied to the server again.
* Limited life span. Secrets issued to the server have a limited life span. The default life span is 3 days. When using bulk setting of secrets during pre-provisioning, the lifespan can be enlarged to give the administrator sufficient time to provision all devices.

```http
> HTTP POST https://server:port/idprov/oobsecret
> {
>   deviceID: {deviceID},
>   oobSecret: {out of band one time secret},
>   validUntil: {iso-date}
> }
> 200 OK
```

Where:
* {deviceID} is replaced by the device-ID when posting to the server. The deviceID must be unique for the server.
* {oobSecret} holds a string with the out-of-band secret that the device will use in its provisioning request
* {validUntil} The OOB secret is kept while the service is running or the validUntil period expires. A restart of the service MUST invalidate existing OOB secrets immediately.


### Device Sends Provisioning Requests

The provisioning request MUST contain the following information:
* The deviceID of the device being provisioned. This ID MUST be unique within the service.
* The device public key in PEM format, used to generate the device certificate. 
* The device IP address at the time of the request. Used for logging and verification.
* The device MAC address at the time of the request. Used for logging and verification.
* signature is the base64 of the HMAC function of the message and the OOB secret. This field is not needed if the client uses a valid certificate.

Once the server has accepted the request and issues a certificate, the OOB secret is removed. Any further provisioning attempts will be treated as if no OOB is known. This adds a time window constraints in which provisioning is accepted for the device.

```http
> HTTP POST https://server:port/idprov/provreq
> {
>    deviceID: {deviceID},
>    ip: {IP address of the device at the time of the request},
>    mac: {MAC address of the device},
>    publicKeyPEM:  {publickey of the device},
>    signature: {base64(HMAC(request,oobSecret))},
> }
> 200 OK
> {
>    status: "Approved",
>    retrySec: 3600,
>    caCert: PEM,
>    clientCert: {signature in PEM format signed by the CA},
>    signature: {base64(HMAC(response,oobSecret))},
> }
```

The response:
* deviceID for which the certificate is valid
* status:
  * Approved  - the request is approved and certificate is include available for download
  * Waiting - the request is waiting on OOB secret to be provided by a third party. The device has to repeat request at a later time.
  * Rejected - the request is rejected
* retrySec - time in seconds to retry the request. In case of status Approved this is the recommended certificate renewal interval.
* caCert The CA certificate that should be used for connections to the message bus and other services. Typically this is the same as the CA for the IDProv server although a different certificate can be used.
* clientCert contains the client certificate signed by the CA.
* signature is the base64 encoded HMAC of the response message and the OOB secret.  This field is not needed if a valid client certificate was used by the device.


The retrySec time indicates when the request must be repeated. In case the server has issued a certificate this is the recommended renewal time. If the server responds with status 'waiting' the client should retry after the indicated time has elapsed. This lets the server manage its load. 

Verification methods:

#### Certificate Request With Signature Verification

A new provisioning request by an unauthenticated client (eg no client certificate) must contain a valid signature field created using the out of band secret.

The signatures in the request and response are generated as follows (see appendix A for a code example): Base64(HMAC(message, oob-secret))
1. Create the message with the signature field an empty string
2. Create a HMAC of the message using the SHA256 hash of the out-of-band secret from the sender.
3. Store the base64 encoded result in the signature field

The signature verification:
1. Recreate the received message with the signature field blank
2. Create the HMAC of the received message using the SHA256 hash of the out-of-band secret from the receiver.
3. Base64 decode the HMAC signature of the received message
4. Use HMAC-verify to compare the two HMAC signatures. 

This verification takes places twice, once by the server when receiving a request and once by the client when receiving a response. If both verifications succeed then this guarantees that both sender and receiver share the same out of band secret and the request and response messages have not been tampered with. 

#### Certificate Renewal 

A provisioning request to renew an existing certificate will be approved without a signature. No out of band secret is needed. 

The client (IoT device) MUST use mutual authentication with the server using a previously issued client certificate that has not expired.

The deviceID of the certificate MUST match the device ID of the request certificate renewal. Eg, the client can renew its own certificate.

#### Certificate Request By Administrator 

An administrator that uses a valid client certificate with the admin or plugin organizational unit can request a certificate for a device. No OOB secret is needed.


### Requesting The Provisioning Status

Administrators can obtain the provisioning status and client certificate of a device. This requires the use of a client certificate with an admin or plugin OU.

```http
> HTTP GET https://server:port/idprov/status/{deviceID}
> 200 OK
> {
>    deviceID: {unique ID of the device}
>    status: "Approved"
>    caCert: PEM,
>    clientCert: PEM,
> }
```
The term "{deviceID}" in the URL MUST be replaced by the device ID when posting to the server.


# Appending A: Code examples

## Signature verification example in golang

Creating a message signature:
```golang
func Sign(message string, sharedSecret string) (base64Encoded string, err error) {
	hmac256 := hmac.New(sha256.New, []byte(secret))
	hmac256.Write([]byte(message))
	hmacMessage := hmac256.Sum(nil)
	base64Signature := base64.StdEncoding.EncodeToString(hmacMessage)
	return base64Signature, nil
}
```


Verifying a signature: 
```golang
func Verify(message string, sharedSecret string, base64Signature string) error {
	hmacMessage, err := base64.StdEncoding.DecodeString(base64Signature)
	if err != nil {
		return err
	}
	// generate a new HMAC using our secret
	hmac256 := hmac.New(sha256.New, []byte(secret))
	hmac256.Write([]byte(message))
	myHmac := hmac256.Sum(nil)
  // HMAC must be equal if both sides use the same message and secret
	equal := hmac.Equal(hmacMessage, myHmac)
	if !equal {
		return errors.New("Message verification failed. Secrets used to sign do not match.")
	}
	return nil
}
```

# Appending B: TLS Services

The services section in the idprov directory result list services that work with the provisioned certificates. The names are predefined as follows:

!!! This section is tentative. Are there existing names for webservices like these that can be used instead? 

* mqtt: MQTT service endpoint
* mqttws: MQTT service websocket endpoint
* wostdir: WoST directory service endpoint
* wostportal: http server providing a web client user interface for consumers.
* adminportal: http server providing a web client user interface for administrators
* adminapi: http server providing REST API to adminster users and groups
