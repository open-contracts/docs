How to write a simple Open Contract
====================================

Structuring your contract's git repo
------------------------------------

First, we need to clarify how to structure an Open Contract repo The ``contract.sol`` and ``README.md`` files aren't used by the protocol and just recommended to make it easier to interpret the oracle logic. The protocol ultimately only cares about the contract that is deployed on Ethereum, and users should verify its code on etherscan for example, which they can do by clicking on the contract address at the top of the automatically generated frontend at https://dapp.opencontracts.io/#/your-github-account/your-contract-repo . The remaining files *are* used by the protocol, so let's dive into them:

::

    contract-repo/
    ├── (README.md)
    ├── (contract.sol)
    ├── interface.json
    └── [oracle_folder]
        ├── oracle.py
        ├── pip_wheels.tar.gzaa
        └── requirements.txt

The first file that you will need to generate for your contract is the ``interface.json``.
It's job is to tell the frontend where to find the contract on chain and what functions it has. It also contains the names and descriptions it should display to the user. As an example, here is the ``interface.json`` from the example `proof-of-id
contract <https://github.com/open-contracts/proof-of-id>`_. 

.. code-block:: json

 {
  "name": "Proof of ID",
  "address": {
    "ropsten": "0x47d162636F3178e0279eBD7fb5e7803cd538C260",
    "optimism": "0xcB3420B31B75a938D937713C434d2379640E496F"
  },
  "descriptions": {
    "contract": "This is contract allows you to...",
    "createID": "This function starts an interactive oracle session in which...",
    "getAccount": "A simple function which returns the Ethereum account belonging to a given ID, if it exists.",
    "getID": "A simple function which returns the ID for a given account, if ..."
  },
  "abi": [{...}, {...}, ...]
 }

At the top level of the interface JSON object is the contract "name", and a mandatory dict containing the addresses of the contract on various chains. The supported strings are "mainnet", "optimism" and "ropsten", with more to come in the near future. Below, you can optionally specify "descriptions" dict containing a string that either explains the "contract" as a whole, or the respective contract function, e.g. "createID". These will be displayed to the user by the frontend. 

The final, mandatory field is the contract ABI (short for *Application Binary Interface*). The ABI is a list of functions that are exposed by the contract, including information about their respective inputs and outputs. When compiling a Solidity contract in the `Remix IDE <https://remix.ethereum.org/>`_, it automatically generates the ABI for you. You just need to copy it by clicking on the copy-icon below **Compilation Details** once you have compiled the contract, and paste it into the ``ìnterface.json``. Two additional ABI tipps:

* Tip 1: your Solidity code, and as a result the generated ABI, often doesn't contain a name for some of the input and output variables When you declare a mapping for example, its input usually isn't named, and the ABI will contain an empty string instead. In these cases, you can just edit the ABI names manually to make things easier for your users.
* Tip 2: because Solidity developers must represent floating point numbers as integers, another feature that we support is to add a "decimals" field to integer inputs or output, and our interface will know where to put the decimal point. For example, since 1 ETH is represented as a 1 followed by 18 zeros, we add ``"decimals": 18`` to the output element of the ``amountOffered`` `function of the FiatSwap contract <https://github.com/open-contracts/fiat-swap/blob/849e81eee05498536aeed8683d6ae977c82db1fd/interface.json#L160/>`_. 

After adding the ABI to the JSON, you will also need to deploy the contract to get its contract address.
To do this, open the **Deploy and Run Transactions** tab (bottom icon in Remix). For 'Environment', you want to select 'Injected Web3' which refers to your MetaMask wallet. When hitting "Deploy", your contract will be deployed to whatever network you set in your MetaMask wallet. For testing an Open Contract, we currently support the Ropsten testnet. Set your Metamask to Optimism if you're ready to go live with real ETH. Once the block containing your deployment is confirmed, you can copy the contract address from the corresponding element of the **Deployed Contracts** list and add it to the `interface.json`.

Now your ``interface.json`` is complete, and you can already interact with your contract using the `OpenContracts Dapp frontend <https://dapp.opencontracts.io>`_. The interface will treat every contract function as a regular Solidity function. 

However, what makes Open Contracts more interesting than regular smart contracts, are *oracle functions*, i.e. functions of your contract which can only be called with the results of a particular `oracle.py` python script, which is securely executed in an `AWS Nitro Enclave <https://aws.amazon.com/ec2/nitro/nitro-enclaves/>`_ with full internet access. For those functions, you need to create an additional directory in the root of your repo, with the folder name being the same as that of oracle function. This directory
should contain a file called `oracle.py`, which contains the logic for downloading some web data and extracting some information from it. It will also
contain a `requirements.txt` file, which will list all Python (pip) packages that your oracle script should use.

Once you created an oracle folder for each oracle function (in our sample contracts there's usually just one), you have to run the **oracle pack** script by running

.. code-block:: console

  $ curl -Ls pack.opencontracts.io | sh

in the command line at the root of your contract repo. This downloads and compresses the packages listed in the `requirements.txt` file and places them into the respective oracle folders. It also generates an `oracleHashes.json` file at the root of your repo, which contains the SHA-256 hash of every oracle folder, uniquely identifying its contents. As we will show you shortly, these hashes are hardcoded into your contract in a way that allows our protocol to ensure that the oracle function can only be called with the results of exactly this specific oracle folder, executed in one of our oracle enclaves.

.. _writing-deploying:

Writing and deploying smart contracts with oracle logic
-------------------------------------------------------
In order to create an Open Contract, you must first write a piece of solidity code that
defines the Ethereum smart contract logic. For a more comprehensive tutorial of
Ethereum smart contacts, we recommend starting `here <https://docs.soliditylang.org/en/v0.7.4/solidity-by-example.html>`_.

In this tutorial, we will go through writing the `Proof-of-id contract <https://github.com/open-contracts/proof-of-id/blob/main/contract.sol>`_ step-by-step.
Writing this contract can be broken into two main steps: writing the ``contract.sol`` and writing the oracle logic.

**Writing contract.sol**
First we'll cover the template contract that you will be using. Navigate to 
`Remix IDE <https://remix.ethereum.org/>`_ in your browser, and add the file
``contract.sol`` under the ``contracts/`` directory.

Like all other contracts, it will be importing the `OpenContractRopsten.sol <https://github.com/open-contracts/ethereum-protocol/blob/main/solidity_contracts/OpenContractRopsten.sol>`_ defined below:

.. code-block:: solidity

    contract OpenContract {
        OpenContractsHub private hub = OpenContractsHub(0x059dE2588d076B67901b07A81239286076eC7b89);

        // this call tells the Hub which oracleID is allowed for a given contract function
        function setOracleHash(bytes4 selector, bytes32 oracleHash) internal {
            hub.setOracleHash(selector, oracleHash);
        }

        modifier requiresOracle {
            // the Hub uses the Verifier to ensure that the calldata came from the right oracleID
            require(msg.sender == address(hub), "Can only be called via Open Contracts Hub.");
            _;
        }
    }


This contract defines a few simple properties: a hub (used to ensure that the
contract is being called from a trusted "hub"), and a map of allowed IDs, called an **oracleHash**. **oracleHashes**
are a unique hash of an oracle node that is allowed to execute a given contract, and is mapped to by a **selector**.
The **selector** is a function name that the oracle will call to resolve the contract.
When the constructor of a contract inheriting ``OpenContract`` is called, it will use the ``setOracle`` function to assign an oracleID to the contract. However, during development, the oracleID ``any`` is used to allow all oracle hashes. Next, the function modifier ``requiresOracle`` is used as a method to check that an oracleID is valid before proceeding to execute the contract's oracle function. You will see an example of this next when defining the proof-of-id contract's `createID <https://github.com/open-contracts/proof-of-id/blob/main/contract.sol#L23>`_ method, which uses the requiresOracle function.

The Proof-of-Id contract uses the secure enclaves to allow users to generate a unique encrypted ID that is verified using an external form of verification. A user proves their identity to the oracle by their SSN account. First, let us define the Proof-of-Id OpenContract in contract.sol in Remix under `contracts/contract.sol`: 

.. code-block:: solidity

    pragma solidity ^0.8.0;
    import "https://github.com/open-contracts/protocol/blob/main/solidity_contracts/OpenContractRopsten.sol";

    contract ProofOfID is OpenContract {

        mapping(bytes32 => address) private _account;
        mapping(address => bytes32) private _ID;

        constructor() {
            setOracle("any", this.createID.selector);  // developer mode, allows any oracle for 'createID'
        }
        ....
    }

In the first half of the contract, we define the solidity syntax version, followed by
importing the OpenContract.sol base contract implementation which we defined above.
Next, the contract ``ProofOfID`` is defined inheriting the OpenContract structure
(see `link <https://www.tutorialspoint.com/solidity/solidity_inheritance.htm>`_ for 
explanation of Solidity inheritance).
The two mappings _account and _ID form a bi-directional mapping between ETH
accounts addresses and the generated unique IDs for a user, which is acquired
once they have proven their identity to the oracle by securely verifying their last
4-digits of their SSN.
As mentioned above, in the constructor, the ``setOracle`` method currently uses
"any" for the oracleHash to allow the createID method to be called on any oracle 
node for development purposes.

Once the mappings and constructor are defined, functions used to get IDs and accounts
using these mappings are specified, followed by the createID method, which
is called by the oracle when the SSN proof has been verified.

.. code-block:: solidity

    contract ProofOfID is OpenContract{
    ....
        function getID(address account) public view returns(bytes32) {
            require(_ID[account] != bytes32(0), "Account doesn't have an ID.");
            return _ID[account];
        }

        function getAccount(bytes32 ID) public view returns(address) {
            require(_account[ID] != address(0), "ID was never created.");
            return _account[ID];
        }

        function createID(address user, bytes32 ID) public requiresOracle { 
            _ID[_account[ID]] = bytes32(0);
            _account[ID] = user;
            _ID[user] = ID;
        }
    }

Note that for any function to modify the _account and _ID mappings, they must
first call the OpenContract's checkOracle function modifier which confirms that the
request is being made by a valid oracleHash. It is important to note that
the first argument of any method using the checkOracle function must always
have oracleHash as it's first argument, so it can properly interact with the
function modifier. After this check is passed, the user is able to map their address
to their generated user ID, completing the Proof-of-ID contract. This will result
in their account paying out to the correct provider wallet.

Implementing the oracle logic and include it in your repo
---------------------------------------------------------

Last but not least, the `oracle.py <https://github.com/open-contracts/proof-of-id/blob/main/createID/oracle.py>`_ script is what enables the key contribution
of the OpenContracts platform: the ability to connect smart contract transactions
to events which are verified by the oracle. This script will parse an html
that gets generated by a user once they have opened an interactive session to
log into an account and access data that only they can provide.
To use this platform, the script imports the opencontracts module from the
`enclave-protocol <https://github.com/open-contracts/enclave-protocol/blob/main/oracle_enclave/user/opencontracts.py#L83>`_,
and some additional modules (bs4 and re) for parsing purposes.

Next, in every oracle script, an ``opencontracts.enclave_backend()`` context manager
is opened to give the script access to the enclave user API, which defines the following
functions:

* ``enclave.print()``: Prints something to the user console (which is displayed on the opencontracts.io contract frontend)
* ``enclave.interactive_session()``: Creates a secure browsing session inside the enclave for the user to navigate to a desired url containing the data to be parsed
* ``enclave.keccak()``: Wrapper to call the ``eth_utils.keccak`` function to generate a hash
* ``enclave.expect_delay()``: Function to create a loading bar in the front-end
* ``enclave.user()``: Returns the user ETH address after being verified by the enclave (by checking a signed random string)
* ``enclave.submit()``: Calls the oracle function in the smart contract once the enclave has verified the parsed data

.. code-block:: python

    import opencontracts
    from bs4 import BeautifulSoup
    import re

    with opencontracts.enclave_backend() as enclave:
      enclave.print(f'Proof of Identity started running in enclave!')

      def parser(url, html):
        target_url = "https://secure.ssa.gov/myssa/myprofile-ui/main"
        assert url == target_url, f"You clicked 'Submit' on '{url}', but should do so on '{target_url}'."
        strings = list(BeautifulSoup(html).strings)
        for key, value in zip(strings[:-1],strings[1:]):
          if key.startswith("Name:"): name = value.strip()
          if key.startswith("SSN:"): last4ssn = int(re.findall('[0-9]{4}', value.strip())[0])
          if key.startswith("Date of Birth:"): bday = value.strip()
        return name, bday, last4ssn


In this first section of the code, the parser method is defined to specify how it
will extract the SSN, birthday, and name of the users page once they have signed 
into the SSA government portal through the interactive session.

.. code-block:: python

   # ... imports
   with opencontracts.enclave_backend() as enclave:
      # ... parser
      name, bday, last4ssn = enclave.interactive_session(url='https://secure.ssa.gov/RIL/', parser=parser,
                                                         instructions="Login and visit your SSN account page.")

      # we divide all 10000 possible last4ssn into 32 random buckets, by using only the last 5=log2(32) bits
      # so last4ssn isn't revealed even if ssn_bucket can be reverse-engineered from ID
      ssn_bucket = int(enclave.keccak(last4ssn, types=('uint256',))[-1]) % 32
      ID = enclave.keccak(name, bday, ssn_bucket, types=('string', 'string', 'uint8'))

      # publishing your SSN reveals that last4ssn was one of the following possibilites:
      possibilities = list()
      enclave.expect_delay(8, "Computing ID...")
      for possibility in range(10000):
        bucket = int(enclave.keccak(possibility, types=("uint256",))[-1]) % 32
        if bucket == ssn_bucket: possibilities.append(str(possibility).zfill(4))
      n = len(possibilities)

      warning = f'Computed your ID: {"0x" + ID.hex()}, which may reveal your name ({name}), birthday ({bday})'
      enclave.print(warning + f' and that your last 4 SSN digits are one of the following {n} possibilites: {possibilities}')

      enclave.submit(enclave.user(), ID, types=('address', 'bytes32',), function_name='createID')

In the latter section, the enclave ``interactive_session`` stores the result from the parser 
as variables, and then performs a randomizing step which cryptographically obscures
the last 4 SSN from any non-trusted parties (using ``keccak``). Finally, it 
submits the result to the ``createID`` function, which stores the mapping from the
user's ETH account to their newly-generated unique SSN hash.

Congrats! You have completed the walkthrough of the first Open Contract!
Now you can try joining the `Discord <https://discord.gg/5X74aw2q>`_ and 
`Reddit <https://reddit.com/r/open_contracts>`_ communities to connect with
developers and learn more, buy our token on `Uniswap <https://app.uniswap.org/#/swap?outputCurrency=0xa2d9519A8692De6E47fb9aFCECd67737c288737F&chain=mainnet>`_.
