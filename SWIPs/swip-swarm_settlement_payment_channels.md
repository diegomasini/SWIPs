---
SWIP: <to be assigned>
title: Introduce support for multiple payment processing options in Swarm
author: Diego Masini (@diegomasini)
discussions-to: <URL>
status: Draft
type: Standards Track
category: Core
created: 2019-07-22
---

<!--You can leave these HTML comments in your merged SWIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new SWIPs. Note that a SWIP number will be assigned by an editor. When opening a pull request to submit your SWIP, please use an abbreviated title in the filename, `SWIP-draft_title_abbrev.md`. The title should be 44 characters or less.-->

## Simple Summary
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the SWIP.-->
In the current Swarm design, accounting of the data exchanged between peers and the payment for such data is coupled. To promote widespread adoption of Swarm it is best to abstract the actual payment mechanism and let nodes participating in the network to decide what payment system better adapts to their needs.

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->
Defining abstractions for payment processing will make easier to integrate different payment methods into Swarm. These abstractions should specify the minimum requirements to allow different implementations to be supported by Swarm, thus decoupling the distributed storage service from the actual payment system used by participants to pay for it. Additionally, a new component needs to be included to keep track of the payment methods available for each peer a node is exchanging data with. The payment method or methods to use should be negotiated when stablishing the connection with such peers. Incorporating this abstractions requires modifications in the handshake protocol, the message handling, and the accounting and payment strategies implemented at the moment.

## Motivation
<!--The motivation is critical for SWIPs that want to change the Swarm protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the SWIP solves. SWIP submissions without sufficient motivation may be rejected outright.-->
Nodes consuming and providing services in the Swarm network may benefit of settling their debts using different payment systems they might already be participating in, like payment channel networks (e.g. Lumino, Raiden or Lightning network). Storage providers offering other paid services will find appealing not to be forced to support multiple payment systems, but being able to consolidate the payments they receive using a single technology. As a side effect, new nodes can bootstrap its participation in a payment channel network by providing storage services in Swarm with zero cost of entry, as described in Generalised Swap Swear and Swindle games, 2019, Tron and Fischer. 

Defining and implementing the required APIs to achieve this decoupling enables Swarm to interoperate accross Blockchains without being tied to a specific Blockchain or settlement technology. Additionally, it enable multicurrency support for users to pay for the storage they consume.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for the current Swarm platform and future client implementations.-->

Swarm defines the ```Balance``` interface  in p2p/protocols/accounting.go as an abstraction for the accounting:

```golang
// Balance is the actual accounting instance
// Balance defines the operations needed for accounting
// Implementations internally maintain the balance for every peer
type Balance interface {
	// Adds amount to the local balance with remote node `peer`;
	// positive amount = credit local node
	// negative amount = debit local node
	Add(amount int64, peer *Peer) error
}
```

Swap (defined in swap/protocol.go) is an implementation of this interface and its ```Spec``` defines the ```EmitChequeMsg``` message:

```golang
// Spec is the swap protocol specification
var Spec = &protocols.Spec{
	Name:       "swap",
	Version:    1,
	MaxMsgSize: 10 * 1024 * 1024,
	Messages: []interface{}{
		HandshakeMsg{},
		EmitChequeMsg{},
		ErrorMsg{},
	},
}
```

Which is handled by the devp2p Peer defined for the Swap protocol in swap/peer.go in the ```handleEmitChequeMsg``` function:

```golang
func (sp *Peer) handleEmitChequeMsg(ctx context.Context, msg interface{}) error 
```

This function performs accounting and payment tasks which are tightly coupled to Swap, making it difficult to support different settlement strategies. 

The ```handleMsg``` function defined in swap/peer.go should delegate the processing of ```EmitChequeMsg``` to a service (from now on ```SwarmPayments```) provinding access to implementations of the Payments API (from now on ```PaymentProcessor```), thus decoupling Swarm from payments processing. The existing code for ```handleEmitChequeMsg``` will become part of the ```PaymentProcessor``` SWAP implementation. The ```handleMsg``` function could be redefined as:

```golang
// handleMsg is for handling messages when receiving messages
func (sp *Peer) handleMsg(ctx context.Context, msg interface{}) error {
	switch msg := msg.(type) {

	case *EmitPaymentMsg:
		return sp.payments.emitPayment(ctx, msg)

	case *ErrorMsg:
		return sp.handleErrorMsg(ctx, msg)

	default:
		return fmt.Errorf("unknown message type: %T", msg)
	}
}
```

where the ```sp.payments``` member of the ```Peer``` struct holds the ```SwarmPayments``` service. This service is responsible to hold the particular ```PaymentProcessor``` implementations available to the node and a mapping between peer (beneficiary) addressess and the ```PaymentProcessor``` negotiated during the handshake. 

The ```Cheque```and ```ChequeParams``` defined in swap/types.go should be more general to allow implementors of the payments API to generate the required data structures for the specific payments implementation (e.g. Balance Proof, in the case of payment channels). The current implementations of ```Cheque``` and ```ChequeParams``` should be part of ```PaymentProcessor``` SWAP implementation:

```golang
// ChequeParams encapsulate all cheque parameters
type ChequeParams struct {
	Contract    common.Address // address of chequebook, needed to avoid cross-contract submission
	Beneficiary common.Address // address of the beneficiary, the contract which will redeem the cheque
	Serial      uint64         // monotonically increasing serial number
	Amount      uint64         // cumulative amount of the cheque in currency
	Honey       uint64         // amount of honey which resulted in the cumulative currency difference
	Timeout     uint64         // timeout for cashing in
}

// Cheque encapsulates the parameters and the signature
type Cheque struct {
	ChequeParams
	Sig []byte // signature Sign(Keccak256(contract, beneficiary, amount), prvKey)
}
```

Upon connection and during the handshake, each peer should indicate its supported ```PaymentProcessor```s, being the SWAP implementation the default and fallback payment method to use. The ```PaymentProcessor``` implementation negotiated during the handshake with a given Peer will be registered in the  ```SwarmPayments``` service. To support multiple payment methods this information could be stored in a map where for each payment method a list of benefiries supporting such method is stored.

```golang 
map[PaymentProcessorDescriptor][]common.Address
```

This structure should be revised if at some point enabling nodes to use a combination of payment methods (in a particular order of preference expressed during the handshake), becomes a nice to have feature for Swarm. 

If no indication of supported payment methods is sent, or if there is no match between the payment methods supported by the two nodes then all settlements will be done by using SWAP cheques and the SWAP chequebook smart contract.

## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

The current Swap implementation uses Ether to settle debts and requires interactions with the SWAP chequebook smart contract. The settlement process is tightly coupled with the Swarm node, making hard to support other currencies besides Ether or other settlement methods such as payment channels. Moreover, this coupling tights Swarm to Ethereum-like Blockchains, impeding other Blockchain solutions to benefit from the integration of Swarm as a distributed storage solution. Several options were considered to decouple the payments technology to use from Swarm:

* Introduce ERC20 support directly into the SWAP chequebook smart contract: It seems feasible to follow this path, however for each new token to be supported a new chequebook needs to be deployed or multiple token support needs to be introduced to the chequebook. While this is possible, it might introduce unwanted complexity to the SWAP chequebook and not enough flexibility to support other means of payment.
* Introduce support for payment channels directly into the SWAP chequebook smart contract: This idea requires an additional level of abstraction for the cheques and the chequebook. The SWAP smart contract should be modified to directly interact with different on-chain payment mechanisms. In the case of payment channel networks cheques should be generalized to allow modeling Balance Proof. The interaction between the chequebook and the payment channel network will occur during the on-chain settlement, when the SWAP smart contract should send the Balance Proof to the payement channel smart contract(s) being use. As with the previous approach, this requires several changes to the SWAP chequebook smart contract. For every payment system to be supported a different chequebook should be designed. Having a single chequebook to handle multiple payment systems will result in an smart contract too difficult to maintain and keep secure.
* Completely replace SWAP by a different payment mechanism: this option is the least flexible of all since it does not solve the problem at all, it only changes the coupling with a given technology (SWAP) for another. Additionally it could hurt the Swarm network, forcing the appearance of multiple subnetworks, each one handling its own payment mechanism, a situation that still could happen with the current Swarm design.

## Backwards Compatibility
<!--All SWIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The SWIP must explain how the author proposes to deal with these incompatibilities. SWIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
To preserve compatibility SWAP cheques and the SWAP chequebook smart contract will remain as the default payment mechanism that all nodes must at least support. This allows payments between nodes to always be made when no indication of the preferred payment method is sent during the handshake, or when there is no match between the payment methods supported by nodes exchanging data.

## Test Cases
<!--Test cases for an implementation are mandatory for SWIPs that are affecting changes to data and message formats. Other SWIPs can choose to include links to test cases if applicable.-->

No test cases for this SWIP are provided at this moment.

## Implementation
<!--The implementations must be completed before any SWIP is given status "Final", but it need not be completed before the SWIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

No implementation for this SWIP is provided at this moment.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
