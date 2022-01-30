Setup
=====

Setting up AWS EC2 Instances
----------------------------

To host the default image, a registry enclave provider and oracle enclave provider
image is required.  The cheapest supported instance type is c5a.2xlarge. 
See https://docs.aws.amazon.com/enclaves/latest/user/getting-started.html. We use the 
AWS Nitro Enclaves Developer AMI (https://aws.amazon.com/marketplace/pp/prodview-37z6ersmwouq2). 
Make sure you set a Security Group that allows for all incoming and outgoing connections. 
We require the following commands before we can run the build:

.. code-block:: console

    sudo yum update -y
    sudo yum install git gcc python3 python3-devel tmux go -y
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
    source $HOME/.cargo/env
    rustup target add x86_64-unknown-linux-musl

Then git clone this repo and proceed with:

.. code-block:: console

    cd default-image
    pip3 install -r ec2_instance/requirements.txt

After launching a fresh oracle instance, make sure you edit
.. code-block:: consol

    sudo nano /etc/nitro_enclaves/allocator.yaml

and set ``memory_mib: 6000``. Then ``sudo reboot``.


Build and run enclave
---------------------
Before running a registry enclave, you must run 

.. code-block:: console

    aws configure

to setup the AWS Access Key ID and Secret Access Key which you can generate [here](https://console.aws.amazon.com/iam/home#/security_credentials$access_key).
This is for allowing automated starting and stopping of oracle EC2 instances.

Then, for both oracle and registry enclaves, make sure you run

.. code-block:: console

    sh build.sh

in the `oracle_enclave` or `registry_enclave` folder respectively. 
Make sure you edit the environment variables inside `./serve_enclave.sh` appropriately:

.. code-block:: console

    PROVIDER=<---your Ethereum address-->
    REGISTRY_ENCLAVE_IP=<---the IP of an existing registry. Leave empty to start a new 'root' registry--->

Then run:

.. code-block:: console

    tmux new-session -d -s enclaveSession "sh serve_enclave.sh"

To start serving your enclave. You can attach to the console via

.. code-block:: console

    tmux attach-session -d -t enclaveSession

Exit the console via ``Ctrl+B``, then ``D``.
Important: make sure you install https://github.com/open-contracts/default-image/blob/main/oracle_enclave/backend/devRootCA.crt as a trusted certificate authority in your browser/system, otherwise your browser will reject the connection.

Running Oracle Enclaves
-----------------------

In order to save on AWS credits/spending, it is recommended to keep several stopped EC2 instances that can be started only
when a user wishes to interact with an oracle enclave. To enable this, from the EC2 dashboard, having selected (checked) the 
respective (stopped) oracle EC2 instances, go to ``Actions > Instance settings > Edit user data`` and paste the contents of
userdata.txt in the textbox.
 
