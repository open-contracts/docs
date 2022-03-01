How to setup an Oracle Provider node
=====

Setting up AWS EC2 Instances
----------------------------
This doc will outline how to set up and provide and our oralce enclave to the protocol, earning OPN in exchange.
We will update the docs once the provision of registry enclaves is supported as well.

You need an AWS account, see https://docs.aws.amazon.com/enclaves/latest/user/getting-started.html. 
The cheapest supported instance type is c5a.4xlarge, at the moment, although we might support cheaper instances
in the future if we slim down the image a bit. You need to use the AWS Nitro Enclaves Developer AMI (https://aws.amazon.com/marketplace/pp/prodview-37z6ersmwouq2). 
Make sure you set a Security Group that allows for all incoming and outgoing connections, at least on ports 8080, 8081 and 14500. 
After loggin into your node, you need to run the following commands before you can run the build:

.. code-block:: console

    sudo yum update -y
    sudo yum install git gcc python3 python3-devel tmux go -y
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
    source $HOME/.cargo/env
    rustup target add x86_64-unknown-linux-musl
    cat /etc/nitro_enclaves/allocator.yaml | sed -e "s/512/11000/g" | sudo tee /etc/nitro_enclaves/allocator.yaml
    wget http://www.dest-unreach.org/socat/download/socat-1.7.4.2.tar.gz
    tar xzf socat-1.7.4.2.tar.gz
    cd socat-1.7.4.2 && ./configure && make && sudo make install
    cd .. && rm socat-1.7.4.2/ socat-1.7.4.2.tar.gz -rf
    git clone https://github.com/open-contracts/enclave-protocol
    cd enclave-protocol
    pip3 install -r ec2_instance/requirements.txt
    sudo reboot

Which installs dependencies and downloads our repo from https://github.com/open-contracts/enclave-protocol.
Login again afterwards, and proceed to build the enclave images.


Build and run enclave
---------------------
To build the enclave images from source, you simply run:

.. code-block:: console

    sh build.sh

in the `oracle_enclave` or `registry_enclave` folder respectively. 
However, the creation of the docker image is not fully deterministic yet, such that you would obtain different enclave PCR0 hashes than those 
permitted by the protocol:

* Oracle Enclave: ``ba19b0568898f7eed74c911b29cccd6d026f918fa77953a705f142d06d33f51dfb00bda8f4e90e13755374548b0430e4``
* Registry Enclave: ``87eb8417e993acfa43fc8e46a1ef0fe50cce86e9b8dcaf22b1e623f8f11a272a0352b851a0503327fb55fd6ecf0259d3``

To get those, you need to run ``sh tars2image.sh`` in the respective enclave folder instead, which will decompress our docker image and deterministally create the enclave image from there. It will also start an interactive terminal in the docker container so you can convice yourself of the source code executed by the enclave.

Before serving your enclave, make sure you edit the environment variables inside `serve_oracle.sh` appropriately:

.. code-block:: console

  PROVIDER=<---the Ethereum address that should get your rewards-->
  REGISTRY=<---the IP of an existing registry--->
  PRICE=<---the amount of OPN * 10^18 that you demand users pay when submitting your results. --->
  NO_DEMAND_TIMEOUT=<---how long the oracle enclave should run before shutting down. leave empty to run forever-->

Note that the registry will always forward the user to the cheapest available oracle. Finally, start your enclave via:

.. code-block:: console

    sh serve_enclave.sh

You can now log out. To see the output of the EC2 side, you can attach to the console via

.. code-block:: console

    tmux attach

Exit the console via ``Ctrl+B``, then ``D``.

Running a Registry Enclave
-----------------------

Right now, the verifier contract does not permit new registries yet, to give the main developers tighter controls of the system initially.
However, you are free to run the code - it's analogous to the oracle enclave code. We'll provide more detailed instructions once the registries are permissionless.
