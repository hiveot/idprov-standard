# 'idprov' IoT Device Provisioning protocol standard

The IoT Device Provisioning protocol aims provide a secure, simple and easy to use means to  provisioning IoT device with a signed certificate.

This project is inspired by the [IETF bootstrapping draft](https://tools.ietf.org/id/draft-sarikaya-t2trg-sbootstrapping-08.html).

## Status

This standard is currently a draft version. The standard is currently being validate within the WoST project, where it serves to provision 'Thing devices' with the WoST Hub.

## Audience

This project is aimed at IoT developers that need a method of provisioning IoT devices with support for out-of-band verification. 

## Description

IDProv is a simple standard for M2M (machine to machine) provisioning of IoT devices and services to enable secure authenticated connections from an IoT device to IoT services. 

Provisioned devices use their signed certificate for establishing a mutual authenticated and secured connection over SSL/TLS with a service provider. 

This standard defines how to provide an IoT device with such a certificate in a secure manner.

The protocol defines 'organizations' to determine the authorization for requesting certificates and publish of out-of-band secrets. 'iotdevice', 'plugin', 'admin' and the three main organizations that are included in the 'ou' field of the certificate. IoT devices can renew their certificate while administrators in the 'admin' ou can issue new certificates.


The ['idprov-protocol'](docs/idprov-protocol.md) documents describes the protocol in detail.


# Provisioning Protocols

The primary objective of provisioning is to establish a trusted relationship between device and server. The two main use-cases are to establish trust over the local network, and establish trust on the internet network. The second use-case is out of scope for this project and pretty much covered by the ACME protocol. The use of TLS ensures that the connection between client and server is secured, even though we don't know who is at the other end. Future improvements to TLS also benefit this protocol.

Establishing trust over a local network can be broken down into two directions:
1. Establish trust of the server by the client
2. Establish trust of the client by the server

## Establish trust of the server by the client

Establishing trust of the server means that the client can be guaranteed that the server is who it claims to be and not a malicious actor. 

The trust level depends on the method of verification. A preset configuration by a third party of the server address, for example pre-installation of a CA certificate on the device, means that the client can verify the server identity. When the server is discovered through DNS-SD a leap of faith is needed that it is a valid server. 

### DNS-SD

The protocol used for automatic discovery is DNS-SD ,as provided by the discover protocol binding. This lends itself for plug and play devices but requires a leap of faith. 
If a CA certificate is pre-provisioned full trust can be established.

## Establising trust of the client by the server

Establishing trust of the client means that the server can verify that the client is who it claims to be to allow it to publish and subscribe messages. This plugin aims to support various machine to machine authentication protocols. 

Key objectives are security and ease of use with support for bulk provision or simple out of band verification.

### IDProv

IDProv is internal IoT Device Provisioning protocol based on certificate exchange over TLS with support for out of band identity verification. 

See [docs/idprov-protocol.md](docs/idprov-protocol.md) for more detail.

### EAP-NOOB 

The [EAP-NOOB protocol](https://tools.ietf.org/html/draft-aura-eap-noob-08) defines an EAP method where the authentication is based on a user-assisted out-of-band (OOB) channel between the server and peer. Many out of band channels can be supported

Unfortunately it looks like the draft of this protocol has expired. As it is quite suited for M2M authentication support might be added in the future.

### EAP-TLS

EAP-TLS is a certificate based mutual authentication protocol.

### Others

This section is in development. For consideration are 
* Light-weight M2M (LwM2M) which requires pre-provisioned credentials on the device



# Contributing

Contributions to WoST projects are always welcome. There are many areas where help is needed, especially with documentation and building plugins for IoT and other devices. See [CONTRIBUTING](https://github.com/wostzone/hub/docs/CONTRIBUTING.md) for guidelines.


# Credits

This project builds on the Web of Things (WoT) standardization by the W3C.org standards organization. For more information https://www.w3.org/WoT/
