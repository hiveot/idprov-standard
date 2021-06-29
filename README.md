# 'idprov' IoT Device Provisioning protocol standard

The IoT Device Provisioning protocol aims provide a secure, simple and easy to use means to  provisioning IoT device with a signed certificate.

The protocol definition can be found here: [idprov-protocol.md](docs/idprov-protocol.md)

## Status

This standard is currently a draft version. The standard is currently being validate within the WoST project, where it serves to provision 'Thing devices' with the WoST Hub.

## Audience

This project is aimed at IoT developers that need a method of provisioning IoT devices with support for out-of-band verification. 

## Description

The primary objective of provisioning is for the client to discover the server and establish a trusted relationship between client and server. 

IDProv is a simple and easy to implement standard for M2M (machine to machine) service discovery and provisioning of IoT devices and services to enable secure authenticated connections from an IoT device to IoT services. 

This standard focuses on provisioning over the local network only. Internet based trust is out of scope as it is already covered with the ACME protocol using DNS and certificate providers. Internet access to local devices can also facilitated by WoST through the use of an Intermediary service. This too is out of scope of provisioning.

Provisioned devices use their signed certificate for establishing a mutual authenticated and secured connection over SSL/TLS with a service provider. 

The ['idprov-protocol'](docs/idprov-protocol.md) document describes the protocol in detail.


# Provisioning Protocol

The scope of provisioning by this standard addresses:
1. Discovery of the provisioning server by the client
2. Establishing trust of the server by the client
3. Establishing trust of the client by the server

## Discovery

For automated M2M provisioning the standard defines the use of DNS-SD/Zeroconf for discovering the IDProv server on the local network using the '_idprov._tcp' service type and a TXT record with the path "path=/path/to/directory" endpoint.

The server publishes this discovery record on the local network for discovery by clients on the same network.

Additional methods of discovery are possible. For example using NFC to transfer the server address to the IoT device and at the same time receive the Device ID and OOB secret. These methods are currently out of scope of this standard but might be included in the future.

## Establish trust of the server by the client

Establishing trust of the server means that the client can be guaranteed that the server is who it claims to be and not a malicious actor. 

If a CA certificate is already pre-provisioned on the client then full trust is automatically established by the TLS protocol used to connect to the server. 

If no CA certificate is available at the time of provisioning then the shared out-of-band secret can be used to establish trust for the client as it does for the server, without the need to take a leap of faith. The protocol provides the means of verifying that the server indeed does possess the shared secret obtained through the out-of-band method.


## Establising trust of the client by the server

Establishing trust of the client means that the server can verify that the client is who it claims to be to allow it to publish and subscribe messages.

The objectives are to be secure and ease of use, with support for bulk provision.


## Other Protocols

Several other protocols exist that can be used for M2M authentication. Most of them have the drawback that they are either not intended for M2M authentication, are more complicated, or difficult to use.

Example protocols that can be used for provisioning:
This list is not complete but will be updated as other suitable protocols are found.

* EAP-NOOB 

The [EAP-NOOB protocol](https://tools.ietf.org/html/draft-aura-eap-noob-08) defines an EAP method where the authentication is based on a user-assisted out-of-band (OOB) channel between the server and peer. Many out of band channels can be supported

Unfortunately it looks like the draft of this protocol has expired. As it is quite suited for M2M authentication support might be added in the future.

* EAP-TLS

EAP-TLS is a certificate based mutual authentication protocol

* LWM2M
 
Light-weight M2M (LwM2M) which requires pre-provisioned credentials on the device



# Contributing

Contributions to WoST projects are always welcome. There are many areas where help is needed, especially with documentation and building plugins for IoT and other devices. See [CONTRIBUTING](https://github.com/wostzone/hub/docs/CONTRIBUTING.md) for guidelines.


# Credits

This project is inspired by the [IETF bootstrapping draft](https://tools.ietf.org/id/draft-sarikaya-t2trg-sbootstrapping-08.html).


This project is part of the WoST project which builds on the Web of Things (WoT) standardization by the W3C.org standards organization. For more information https://www.w3.org/WoT/
