Tutorial: Writing an OpenContract
=====

.. Define oracle node in "glossary," as well as OpenContract, oracle.py, and contract.sol

In this tutorial, you will be creating your own `contract.sol` and `oracle.py` to handle the format and logic of your OpenContract that will execute on an oracle node. 

For a fully implemented example OpenContract, see the `fiat-swap repo <https://github.com/open-contracts/fiat-swap>`_.

Introduction
--------------------

In order to make an OpenContract, you must first write a piece of solidity code that
defines the Ethereum smart contract logic. For a more comprehensive tutorial of
Ethereum smart contacts, we recommend starting `here <https://docs.soliditylang.org/en/v0.7.4/solidity-by-example.html>`_.

In this tutorial, we will be recreating the fiat-swap OpenContract to enable a user to
exchange be using the `Remix IDE <https://remix.ethereum.org/>`_, which
will allow you to easily develop and modify your contact.
At the top of `contract.sol` are the pragma and import statements, which define the
version solidity and import the OpenContact object type.

First we'll cover the template contract that you will be using. Navigate to 
`Remix IDE <https://remix.ethereum.org/>`_ in your browser, and add the file
`contract.sol` under the `contracts/artifacts/` directory.

This contract will be importing the `OpenContractRopsten.sol` contract defined below: 

.. code-block:: solidity

    contract OpenContract {
        address private hub = 0xACf12733cBa963201Fdd1757b4D7A062AD096dB1;
        mapping(bytes8 => bytes32) private allowedID;

        function setOracle(bytes32 oracleID, bytes8 selector) internal {
            allowedID[selector] = oracleID;
        }

        modifier checkOracle(bytes32 oracleID, bytes4 selector) {
            require(msg.sender == hub, "Can only be called via Open Contracts Hub.");
            if (allowedID[selector] != "any") {
                require(oracleID == allowedID[selector], "Incorrect OracleID.");
            }
            _;
        }
    }

This contract defines a few simple properties: a hub (used to ensure that the
contract is being called from a trusted "hub"), and a map of allowed IDs. IDs
are a unique hash for the code defining a specific contract, mapped to by
a selector, which is a function name that the oracle will call to resolve the contract.
When the constructor of a contract inheriting `OpenContract` is called, it will
use these functions to specify the contract.

Next, we will use the example of the `fiat-swap contract <https://github.com/open-contracts/fiat-swap/>`_ as an illustration of a
contract inheriting from the `OpenContract` class, and creating additional functionality
to design a trustless fiat to crypto on-ramp.

Step-by-step: `fiat-swap` contract
--------------------------------
.. TODO: include motivation for why OpenContracts is a suitable protocol for fiat-swap

To enable fiat-swap, we need to build a contract that can handle the logic
of a transaction between two parties: a (eth) buyer and seller. We will first
build that logic into an OpenContract that can guarantee the correct parties
are able to transact, using a payment service of their choosing, and no one else.

First, let us define the FiatSwap OpenContract in contract.sol using solidity: 

.. code-block:: solidity

    pragma solidity ^0.8.0;
    import "https://github.com/open-contracts/protocol/blob/main/solidity_contracts/OpenContractRopsten.sol";

    contract FiatSwap is OpenContract {
        mapping(bytes32 => address) seller;
        mapping(bytes32 => address) buyer;
        mapping(bytes32 => uint256) amount;
        mapping(bytes32 => uint256) lockedUntil;

        constructor() {
            setOracle("any", this.buyTokens.selector); // developer mode, allows any oracleID for 'buyTokens'
        }
        ....
    }

The first lines specify the version of Solidity to be used, and ensure that the OpenContract definition is visible to be imported 
by this contract. Inside the contract, we see a number of addresses, which map a unique transaction ID to the correct buyer, 
seller, amount, and timeout for the transaction. Since this contract inherits from the OpenContract defined earlier, it must define a constructor
which sets the Oracle providers that provide access to this contract. In this case, we
use "any" to allow any oracle node to execute this contract. Additionally, the `this.buyTokens.selector` argument
specifies which of the methods implemented in this contract can be called by the oracle.py code (which we 
cover later in this tutorial).

Next, we will define the logic to create an offerID, which is a unique transaction ID
generated once the seller posts an offer specifying the amount and price, as well as a
few auxilliary arguments:

.. code-block:: solidity

    function offerID(string memory sellerHandle, uint256 priceInCent, string memory transactionMessage,
                     string memory paymentService, string memory buyerSellerSecret) public pure returns(bytes32) {
        return keccak256(abi.encode(sellerHandle, priceInCent, transactionMessage, paymentService, buyerSellerSecret));
    }

`keccak256(abi.encode(...))` is the standard method to compute a hash or structured data in solidity, and is described
further in the
`ABI spec <https://docs.soliditylang.org/en/latest/units-and-global-variables.html?highlight=abi.encode#abi-encoding-and-decoding-functions>`_.
Similar to the oracleID created earlier, this function generates a hash from the input arguments, and is used to uniquely identify
this transaction. The offerIDs are used as keys by the mappings created with the OpenContract class to map to the 
buyer, seller, amount, and time the contract is valid for. The additional arguments here are to set the transactionMessage the buyer must use when sending money via
the paymentService (PayPal/Venmo) to trigger the contract, and the buyerSellerSecret, which is a password to
generate the correct offerID, so that only the right buyer can trigger the OpenContract.

Next, we provide some helper functions which can be used to check the status and details of the contract.

.. code-block:: solidity

    // sellers should lock their offers, to give the buyer time to make and verify their online payment.
    function secondsLocked(bytes32 offerID) public view returns(int256) {
        return int256(lockedUntil[offerID]) - int256(block.timestamp);
    }

    // every offer has a unique offerID which can be computed with this function.
    function weiOffered(bytes32 offerID) public view returns(uint256) {
        require(msg.sender == buyer[offerID], "No ether offered for you at this offerID.");
        require(secondsLocked(offerID) > 1200, "Offer isn't locked for at least 20min. Ask the seller for more time.");
        return amount[offerID];
    }

The `secondsLocked()` mathod determines whether the contract has expired or not to ensure that the
seller is still willing to sell the ETH at that price. It uses the earlier mappings
lockedUntil to lookup the timeout limit specified by the seller, as well as the
global `block and msg.sender variables <https://docs.soliditylang.org/en/latest/units-and-global-variables.html#special-variables-and-functions>`_ which gives the current timestamp
and address of the message sender.
`weiOffered()` confirms the amount of ETH offered by the seller by looking up `offerID` in the `amount` mapping, 
and asserts (written as `require` in solidity) the buyer and the `secondsLocked` values are correct.

Finally, we implement the main functionality of the contract, which handles making and retracting a contarct offer, and sending the tokens
once the oracle has verified that the buyer has sent fiat currency via the chosen paymentService.

First, the `offerTokens` method is called by the seller from the OpenContracts website
to set a price, amount, buyer address and expiriation time for the transaction after
an offerID has been generated (specifying the message and secret). The msg.sender
variable records the ETH address of the seller from which the ETH is sent after
the contract is upheld.

.. code-block:: solidity

    // to make an offer, the seller specifies the offerID, the buyer, and the time they give the buyer
    function offerTokens(bytes32 offerID, address buyerAddress, uint256 lockForSeconds) public payable {
        amount[offerID] = msg.value;
        buyer[offerID] = buyerAddress;
        lockedUntil[offerID] = block.timestamp + lockForSeconds;
        seller[offerID] = msg.sender;
    }

If, after seeing a fluctuation in the price or changing their mind the seller
wishes to retract their offer, they can call the `retractOffer` function
from the OpenContracts website to cancel their contract before it is fulfilled.
This is only after the locking period for their contract has expired (which requires
that `secondsLocked(offerID) <= 0`, after which the amount for the contract
is set to 0, closing the contract by sending the locked ETH back to the buyer.
Note that to ensure that only the seller can retract the offer, the function
also requires that the address of the seller matches the address of the caller
of the function (`msg.sender`).

.. code-block:: solidity

    // sellers can retract their offers once the time lock lapsed.
    function retractOffer(bytes32 offerID) public returns(bool) {
        require(seller[offerID] == msg.sender, "Only seller can retract offer.");
        require(secondsLocked(offerID) <= 0, "Can't retract offer during the locking period.");
        uint256 payment = amount[offerID];
        amount[offerID] = 0;
        return payable(msg.sender).send(payment);
    }

Finally, the `buyTokens` function method is defined to allow a buyer to receive
the ETH after verifying their external transaction to the seller with the oracle.
As we will show later with our `oracle.py` code, this function is triggered
by the oracle's Python API after the transaction from the buyer to the seller
on the external payment service is verified in the Oracle Enclave.

.. code-block:: solidity

    // to accept a given offerID and buy tokens, buyers have to verify their payment 
    // using the oracle whose oracleID was specified in the constructor at the top
    function buyTokens(bytes32 oracleID, address payable msgSender, bytes32 offerID) 
    public checkOracle(oracleID, this.buyTokens.selector) returns(bool) {
        require(buyer[offerID] == msgSender);
        uint256 payment = amount[offerID];
        amount[offerID] = 0;
        return msgSender.send(payment);
    }

Note that the inherited `checkOracle` method called here to ensure that the 
Oracle sending the buyTokens method is verified before the payment is allowed to
go through.

