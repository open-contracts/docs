Writing a simple Open Contract
====================================

Structuring your contract's git repo
------------------------------------

First, we need to clarify how to structure an Open Contract repo. The ``contract.sol`` and ``README.md`` files aren't used by the protocol and just recommended to make it easier to interpret the oracle logic. The protocol ultimately only cares about the contract that is deployed on Ethereum, and users should verify its code on etherscan for example, which they can do by clicking on the contract address at the top of the automatically generated frontend at https://dapp.opencontracts.io/#/your-github-account/your-contract-repo . The remaining files *are* used by the protocol, so let's dive into them:

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
To do this, open the **Deploy and Run Transactions** tab (bottom icon in Remix). For "Environment", you want to select "Injected Web3" which refers to your MetaMask wallet. When hitting "Deploy", your contract will be deployed to whatever network you set in your MetaMask wallet. For testing an Open Contract, we currently support the Ropsten testnet. Set your Metamask to Optimism if you're ready to go live with real ETH. Once the block containing your deployment is confirmed, you can copy the contract address from the corresponding element of the **Deployed Contracts** list and add it to the `interface.json`.

Now your ``interface.json`` is complete, and you can already interact with your contract using the `OpenContracts Dapp frontend <https://dapp.opencontracts.io>`_. The interface will treat every contract function as a regular Solidity function. 

However, what makes Open Contracts more interesting than regular smart contracts, are *oracle functions*, i.e. functions of your contract which can only be called with the results of a particular `oracle.py` python script, which is securely executed in an `AWS Nitro Enclave <https://aws.amazon.com/ec2/nitro/nitro-enclaves/>`_ with full internet access. For those functions, you need to create an additional directory in the root of your repo, with the folder name being the same as that of oracle function. This directory
should contain a file called `oracle.py`, which contains the logic for downloading some web data and extracting some information from it. It will also
contain a `requirements.txt` file, which will list all Python (pip) packages that your oracle script should use.

Once you created an oracle folder for each oracle function (in our sample contracts there's usually just one), you have to run the **oracle pack** script by running

.. code-block:: console

  $ curl -Ls pack.opencontracts.io | sh

in the command line at the root of your contract repo. This downloads and compresses the packages listed in the `requirements.txt` file and places them into the respective oracle folders. It also generates an `oracleHashes.json` file at the root of your repo, which contains the SHA-256 hash of every oracle folder, uniquely identifying its contents. For the proof-of-id contract, it looks as follows:

.. code-block:: json

  {
      "createID": "0x28316674db6d4af06cdeb422d0fe308a4704b01b3e3487813a0d9dab458be665"
  }

because `createID` is the only folder in the repo containing an `oracle.py`, as `createID` is going to be the only oracle function of the contract. As we will show you next, these hashes are hardcoded into your contract in a way that allows our protocol to ensure that the oracle function can only be called with the results of exactly this specific oracle folder, executed in one of our oracle enclaves.

NOTE (!): unfortunately, the download is currently not deterministic. So running the same command twice will result in a different oracle hash. To verify that a given folder hashes to a certain value, you should therefore run the "pack oracles" script without the download, via:

.. code-block:: console

  $ curl -Ls pack.opencontracts.io | DL=NO sh

.. _writing-deploying:

Writing and deploying smart contracts with oracle logic
-------------------------------------------------------
In order to create an Open Contract, you must first write a piece of solidity code that
defines the Ethereum smart contract logic. For a more comprehensive tutorial of
Ethereum smart contacts, we recommend starting `here <https://docs.soliditylang.org/en/v0.7.4/solidity-by-example.html>`_.

In this tutorial, we will go through writing the `Proof-of-ID contract <https://github.com/open-contracts/proof-of-id/blob/main/contract.sol>`_ step-by-step.
Writing this contract can be broken into two main steps: writing the ``contract.sol`` and writing the oracle logic.

**Writing contract.sol**
First, navigate to `Remix IDE <https://remix.ethereum.org/>`_ in your browser, and create an empty file
``contract.sol`` under the ``contracts/`` directory.

Like all other contracts (on ropsten), we will import the `OpenContractRopsten.sol <https://github.com/open-contracts/ethereum-protocol/blob/main/solidity_contracts/OpenContractRopsten.sol>`_ which looks as follows:

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
    
    interface OpenContractsHub {
        function setOracleHash(bytes4, bytes32) external;
    }


This defines the parent class for all Open Contracts, consisting three two simple parts: a pointer (called *interface* in solidity) to the Open Contracts Hub. Then it defines a `setOracleHash` function, which calls the Hub's `function with the same name <https://github.com/open-contracts/ethereum-protocol/blob/99e3d47be68f253dd78a60c0f05e6a3279bf8a47/solidity_contracts/Hub.sol#L19/>`_. This tells our protocol which ``oracleHash`` you want to allow for a given function.
The second is the `requiresOracle` function modifier, which you can place at the top of a function to declare it as an oracle function, as we will see shortly. This will ensure that the function can only be called through our protocol.

Let's see how the Proof-of-ID contract inherits from the ``OpenContract`` class. Place the following code into your ``contract.sol`` file in Remix:

.. code-block:: solidity

    pragma solidity ^0.8.0;
    import "https://github.com/open-contracts/protocol/blob/main/solidity_contracts/OpenContractRopsten.sol";

    contract ProofOfID is OpenContract {

        mapping(bytes32 => address) private _account;
        mapping(address => bytes32) private _ID;

        constructor() {
            setOracleHash(this.createID.selector, 0x28316674db6d4af06cdeb422d0fe308a4704b01b3e3487813a0d9dab458be665);
        }
        ....
    }


In the first half of the contract, we define the solidity syntax version and import the OpenContractRopsten.sol we examined above.
Next, the contract ``ProofOfID`` inherits the OpenContract structure
(see `link <https://www.tutorialspoint.com/solidity/solidity_inheritance.htm>`_ for 
explanation of Solidity inheritance), which just means it can now use the ``setOracleHash`` function and `requiresOracle`` modifier from its parent. The two mappings _account and _ID will form a bi-directional mapping between ETH accounts addresses and the generated unique IDs for a user, which we will later compute from the unique personal information displayed on the user's social security account website. The oracle folder containing the corresponding logic has the hash ``0x283...```. Using the ``setOracleHash`` expression in the constructor of the contract, we declare that the ``createID`` function (identified by the four bytes returned from ``this.createID.selector``) can only be called with the results from this oracle folder.

Once the mappings and constructor are defined, we can write our functions.
The first two are simple solidity functions which tell us the account for a given ID and vice versa. 
The interesting one is the ``createID`` function, which contains ``requiresOracle`` at the top:

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

This function allows accounts to register a new ID, updating the ``_ID`` and ``_account`` mappings accordingly. The ``requiresOracle`` modifier makes 
sure that this function can only be called by the Hub, which in turn makes sure that the results it forwards were computed by an oracle with the hash ``0x283..``.


Implementing the oracle logic and include it in your repo
---------------------------------------------------------

Last but not least, the `oracle.py <https://github.com/open-contracts/proof-of-id/blob/main/createID/oracle.py>`_ script is the key feature
of the OpenContracts protocol: the ability to condition smart contracts on real-world information which is by the oracle enclave. 

For the Proof-of-ID contract, this script will instruct the user to log into and save their social security account website, then parse its html to extract their personal information, and compute their personal ID from it.

To use this platform, the script imports the ``opencontracts`` module which (currently) only exists
`inside the enclave <https://github.com/open-contracts/enclave-protocol/blob/main/oracle_enclave/user/opencontracts.py#L83>`_, and exposes the a few special Open Contracts features. Otherwise, the script can of course import all other python modules that are contained in standard python (such as ``re``) or in `requirements.txt` (such as ``bs4``).

Next, in every oracle script, an ``with opencontracts.session() as session:`` context manager is opened to give the script access to the enclave user API, and forward errors to the user wheverer possible. The ``session`` object contains the following special functions:

* ``session.print(message)``: Displays a message to the user
* ``session.interactive_browser(url, parser, instructions)``: Creates an interactive browsing session in which the user controls a chrome instance inside the enclave, is instructed to navigate somewhere and save the result.
* ``session.keccak(*args, types)``: Computes Solidity's version of SHA-256 called "keccack", wrapping ethereum's ``eth_utils.keccak`` package
* ``session.expect_delay(seconds, message)``: Displays a loading bar to the user, with a message to the user why they have to wait
* ``session.user()``: Returns the user's ETH address after being verified by the enclave (by letting them sign a random string)
* ``session.submit(*args, types, function_name)``: Calls the oracle function in the smart contract with the final results

The ``oracle.py`` file of the Proof-of-ID contract begins as follows:

.. code-block:: python

    import opencontracts
    from bs4 import BeautifulSoup
    import re
    
    with opencontracts.session() as session:
      session.print(f'Proof of ID started running in enclave!')
      
      instructions = """
      1) Log into your account
      2) Navigate to 'My Profile'
      3) Click 'Submit'
      """
      
      def parser(url, html):
        target_url = "https://secure.ssa.gov/myssa/myprofile-ui/main"
        assert url == target_url, f"You clicked 'Submit' on '{url}', but should do so on '{target_url}'."
        strings = list(BeautifulSoup(html).strings)
        for key, value in zip(strings[:-1],strings[1:]):
          if key.startswith("Name:"): name = value.strip()
          if key.startswith("SSN:"): last4ssn = int(re.findall('[0-9]{4}', value.strip())[0])
          if key.startswith("Date of Birth:"): bday = value.strip()
        return name, bday, last4ssn
      
      name, bday, last4ssn = session.interactive_browser('https://secure.ssa.gov/RIL/', parser, instructions)

After importing everything and opening the context manager, we define the ``instructions`` for the user and a ``parser`` function, which takes as input a ``url`` string a ``html`` string, performs the necessary checks and parses the info we need. The external ``BeautifulSoup`` package makes this very easy. Both the ``parser`` and the ``instructions`` are passed to the ``session.interactive_browser`` function. The instructions are displayed to the user. Everytime they hit save, the parser is executed over their current html. If it throws an error, the error message is displayed to the user. If it doesn't, the browser closes and the results are returned.

Next, we compute the user's ID from their private details:

.. code-block:: python

   ...
   with opencontracts.enclave_backend() as enclave:
      ...
      
      # we divide all 10000 possible last4ssn into 32 random buckets, by using only the last 5=log2(32) bits
      # so last4ssn isn't revealed even if ssn_bucket can be reverse-engineered from ID
      ssn_bucket = int(session.keccak(last4ssn, types=('uint256',))[-1]) % 32
      ID = session.keccak(name, bday, ssn_bucket, types=('string', 'string', 'uint8'))  
      
      # publishing your SSN reveals that last4ssn was one of the following possibilites:
      possibilities = list()
      session.expect_delay(8, "Computing ID...")
      for possibility in range(10000):
        bucket = int(session.keccak(possibility, types=("uint256",))[-1]) % 32
        if bucket == ssn_bucket: possibilities.append(str(possibility).zfill(4))
      n = len(possibilities)
    
      warning = f'Computed your ID: {"0x" + ID.hex()}, which may reveal your name ({name}), birthday ({bday})'
      session.print(warning + f' and that your last 4 SSN digits are one of the following {n} possibilites: {possibilities}')
      
      session.submit(session.user(), ID, types=('address', 'bytes32',), function_name='createID')

Since this contract is a bit privacy sensitive, we also display a warning to the user telling them exactly what sensitive information might be revealed by submitting their ID to the public blockchain. It may reveal their name and birthday, but doesn't reveal their last 4 ssn digits, only reduces the possibilities for those 4 digits from 10000 to around 300.

Finally, it submits the result to the ``createID`` function, which stores the mapping from the user's ETH account to their newly-generated unique ID.

Congrats! You have completed the walkthrough of the first Open Contract!
Please join our `Discord <https://discord.gg/5X74aw2q>`_ community to get developer support and build some contracts together! 
