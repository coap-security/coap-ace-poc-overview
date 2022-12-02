# CoAP/ACE PoC: Project overview

The CoAP/ACE Proof-of-Concept project
demonstrates the use of CoAP[^1], the IETF's standard application protocol for the Internet of Things,
and ACE[^2], the corresponding authorization framework.
It transports CoAP over GATT (Bluetooth Low Energy),
allowing local access from off-the-shelf cell phones
even when connectivity to the Internet is limited.

[^1]: Constrained Application Protocol; [RFC7252] and related documents
[RFC7252]: https://www.rfc-editor.org/rfc/rfc7252.html
[^2]: Authentication and Authorization for Constrained Environments; [RFC9200] and related documents
[RFC9200]: https://www.rfc-editor.org/rfc/rfc9200.html

The demo application is executed across three major components:

* An embedded development kit, the Nordic Semiconductors' nRF52-DK,
  the "device",
  stands in for a sensor or actuator device in the field.

  In the context of the ACE framework, it implements the Resource Server (RS) role.

* An authorization server (abbreviated AS in the context of ACE), running in a data center,
  provides security tokens.
  It is configured with symmetric keys for all devices.

* The user's cell phone running a web application provides the user interface.
  It communicates with the authorization server through the Internet (using WebSockets),
  and with the device using Bluetooth Low Energy (using Web Bluetooth, available on modern Chrome-based browsers).

  In the ACE framework, it implements the Client (C) role.

## Components

*This section describes roles and components of a generalized setup
of which the Proof-of-Concept implements a concrete simple example.*

@@@ high-level diagram, split down to components (who stores what)

### Resource Server (Device)

The role of the Resource Server (RS) is implemented on any device that provides functionality accessed through a network.
ACE is designed to allow this role to be played by highly constrained devices:
with the concrete profile used here, there is no need even for asymmetric cryptography.

The device houses several losely coupled components:

* The actual application:
  reading sensor values, driving actuators or any control componet built on their combination.
* A CoAP stack:
  mapping request coming through the network through the security layer to application requests.
  Note that the security layers used here are "on top" of CoAP.
  This allows applying them consistently on any transport of CoAP.
  For example, a device might offer the same interfaces over Bluetooth and over UDP when connected through a cellular network.
* [OSCORE], CoAP's symmetric security layer.
  It provides message authentication, encryption and integrity protection as well as replay protection.
  In the present setup, it relies on the symmetric key material being provisioned by the Authorization Server (AS)
  through the [ACE OSCORE Profile].
* A shared key with the AS.
  While the RS does not communicate directly with the AS,
  it receives tokens encrypted with this key.
* A pool of recently used tokens.
  These tokens are submitted to the RS by the Client,
  which obtains them from the AS.
  The RS decrypts them, and uses the key material contained therein to set up its OSCORE contexts.
  The tokens contain authorizations for particular operations in the application's model,
  which can be as coarse or fine-grained as the application demands,
  and need not be concerned with the roles or identities backing them.
  The RS stores these on a best-effort base;
  when too many clients attempt communication, old tokens can be discarded,
  and will be provided by the clients again.
  
[OSCORE]: https://www.rfc-editor.org/rfc/rfc8613
[ACE OSCORE Profile]: https://www.rfc-editor.org/rfc/rfc9203

  @@@ stack diagram next to other components

### Authorization Server

### Mobile application

## Implementation

All components that were developed within this project are implemented in the Rust programming language.
This allows easy sharing of code between the mobile application and the embedded device,
eases experimentation[^exp].
Its rigorous type system lends itself to building high quality implementations
even in a prototyping project (see [Maturities]).

[^exp]: Bluntly speaking, a careless change in a Rust program is just as likely to work as in any other language, but if it does break the application subtly, this is often detected at build time.

### Resource Server (Device)

The demo application for the nRF52-DK development kit
is called [coap-ace-poc-firmware and hosted on GitLab],
along with its documentation@@@.

[coap-ace-poc-firmware and hosted on GitLab]: https://gitlab.com/oscore/coap-ace-poc-firmware

When built, the application produces a set of firmware images,
which can be programmed onto the development kit by means of drag-and-drop.
These different images are pre-provisioned with keys shared with the Authorization Server,
as well as distinct identities.

### Authorization Server

### Mobile application

@@@ which does which

### Implementation caveats

@@@ fill from offer list (might still merge with Further and related work)

## Learnings & notes for future development

### Choices and trade-offs

* Time vs cnonce (incl time sources and token lifetimes)

## Further and related work

### Discovery and identification

The demo's discovery model is built on two assumptions:

* Users have no prior knowledge of the precise device they connect to:
  they use Bluetooth proximity to discover any device,
  pick one by name if multiple are around,
  and obtain proof of the device being one under the AS's control when a connection is established.
* There is only one "application" around, and it is known by all involved what it is.

Matching this with deployment realities will need further exploration.
Concrete mechanisms are available for some aspects,
while others may profit from further development on the ACE and CoAP-over-GATT side:

* @@@

### Commissioning (firmware updates)
### C-AS integrations (OAuth, cookies, kerberos)
### Proximity

### Various implementation caveats

* Randomness:
  <del>The devices lack a good source of entropy.</del> @@@ will use softdevice's randomness

* Various quality-of-implementation things (overwriting the same token rather than oldest)
  @@@

### Maturities


### Demo

@@@

See [the firmware's README file](../coap-ace-poc-firmware/README.md)
for a description of how to install the demo,
and how to connect from the browser.

Documentation on how to build and configure the individual components
is provided with the respective components.
