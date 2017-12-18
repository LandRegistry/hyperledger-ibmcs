# How to deploy Hyperledger on IBM Cloud - Customised for HMLR Digital Street

The transaction_backend app built for the Digital Street hack day needs to communicate with Hyperledger Fabric via Hyperledger Composer. It was built to be compatible with Composer 0.13.1 with respect to the way it authenticates (connection profiles with certificates). However as of version 0.15 the authentication method has completely changed, to be via all-encompassing "cards" that need to be loaded into the environment using the Composer CLI. While we can export cards from Composer Playground and change the app to use them, when deployed to a cloud machine outside of our control there is currently no easy way to use the composer CLI to load cards and make them available to the app, and no way to use environment variables instead like we do for connection profiles (see [this issue](https://github.com/hyperledger/composer/issues/2328)). There were other breaking changes in 0.14 too with respect to access control in the business network itself that means the easiest option is to carry on using 0.13.1.

These instructions and scripts were designed to work with that version (the maintained ones that work with the latest versions are [here](https://github.com/IBM-Blockchain/ibm-container-service/)) and so we have modified them to simplify deployment (only one peer) and pin them to 0.13.1 images.

The following instructions have been copied and modified (to deal with the fact we are working with an older version) from [here](https://ibm-blockchain.github.io/setup/)

## Pre-requisites

As you'll be interacting with Kubernetes, IBM Bluemix and Hyperledger Composer, you need to install three command line tools:

- node (latest LTS)
- kubectl (latest)
- bx (latest)
- Hyperledger Composer CLI (0.13.1).

Install `node` from [here](https://nodejs.org/en/download/) - or using brew is easier on a Mac.

Install `kubectl` from [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/) choosing the option of your choice

Install `bx` from
[here](https://console.bluemix.net/docs/cli/reference/bluemix_cli/download_cli.html).

Validate the installations with `node --version`, `kubectl version` and `bx -v`.

Now add the container service plugin - this will let you interact with the IBM Container Service. Add the repo first (this tells the following command where to find the plugin to be installed) - you may get a message saying it already exists, if so that's fine.

```bash
bx plugin repo-add bluemix https://plugins.ng.bluemix.net
bx plugin install container-service -r bluemix
```

Install the Hyperledger Composer CLI tool with

```bash
npm install -g composer-cli@0.13.1
```

## Clone the repositories

Clone this git repository and transaction_backend to your local machine if you haven't already and cd into this one.

## Set up a container cluster

Point the Bluemix CLI at the (UK) API endpoint, then login

```bash
bx api api.eu-gb.bluemix.net
bx login
```

This will ask for your IBM ID and the account password.

You will be asked to select the number of an account (if using the our main one it's 1 - HM Land Registry). You also need to set an org and a space. Assuming these have already been set up in the web UI, you can just run the following (our main ones are HMLR Digital Street / dev)

```bash
bx target --cf
```

Now create the cluster on the IBM Container Service

```bash
bx cs cluster-create --name cluster_name_here
```

This could take up to 30 minutes. You can check progress with

```bash
bx cs clusters
```

Once the _State_ shows _normal_, it's done.  You should see something like this:

```bash
Listing clusters...
OK
Name                ID                                 State    Created                    Workers
cluster_name_here   0783c15e421749a59e2f5b7efdd351d1   normal   2017-05-09T16:13:11+0000   1
```

Once that's done, you can check the status of the worker node:

```bash
bx cs workers cluster_name_here
```

This will show the public and private IP addresses.  Note down the public IP address, as you will use this later to access the Blockchain network.

## Configure kubectl to use the cluster

Issue the following command

```bash
bx cs cluster-config cluster_name_here
```

The output will contain an `EXPORT` command which will point your local `kubectl` to the cluster.  Copy and paste that command into the command line and run it. It will be something like this:

```bash
export KUBECONFIG=/home/*****/.bluemix/plugins/container-service/clusters/cluster_name_here/kube-config-prod-dal10-blockchain.yml
```

 If you open any more command windows at a later date you may need to rerun that.

## Install the Blockchain network

The _kube-configs_ directory defines a Blockchain implementation which consists of:

- Hyperledger Fabric (a peer, a database, an orderer and a CA)
- Hyperledger Composer Playground
- Hyperledger Composer Rest Server
- utility containers for creating the keys and certificates
- utility containers for creating and joining a channel
- utility containers for installing and instantiating the test chaincode
- persistent storage to allow containers to share cryptographic material

To deploy the Blockchain:

```bash
cd scripts
./create_all.sh --with-couchdb
```

Once that's complete you can use the Kubernetes Dashboard to explore the services and pods which have been created.  Run

```bash
kubectl proxy
```

Now browse to [here](http://localhost:8001/ui) and you will see the dashboard.

You can get all of this information from the command line (try `kubectl get pods -a`), but it's convenient to have it all just a few clicks away.

## Create a local connection profile
We're going to deploy the business network, but to do that we need a connection profile to tell our local Hyperledger Composer CLI where to deploy it.  Local connection profiles are stored in _~/.composer-connection-profiles/_ by default.

Create a new connection profile directory for IBM Container Services and copy the example profile.

```bash
mkdir ~/.composer-connection-profiles/ibmcs
cp profile/connection.json ~/.composer-connection-profiles/ibmcs
```

Edit it to use the public IP address of your container cluster - the one from `bx cs workers cluster_name_here`.

## Copy the credentials from the running Hyperledger instance

When the Hyperledger instance was deployed to IBM Container Service, a set of cryptographic credentials was created.  We need to copy some of those off the peer so we can connect to it locally.

Start by getting the container name of the _Org1_ peer - it will be something like _blockchain-org1peer1-1820571918-bdqrv_.

```bash
kubectl get pods
```

Now extract two files from that peer - these are the certificate and key for the admin user.  _You need to use the container name you found in the previous step_.

```bash
kubectl cp blockchain-org1peer1-xxxxxxxxxx-xxxxx:/shared/crypto-config/peerOrganizations/org1.example.com/users/Admin\@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem cert.pem
kubectl cp blockchain-org1peer1-xxxxxxxxxx-xxxxx:/shared/crypto-config/peerOrganizations/org1.example.com/users/Admin\@org1.example.com/msp/keystore/key.pem key.pem
```

Import that identity into the local credential store, creating a new identity to use them with at the same time (called PeerAdmin).  Clear out the store first to remove any old keys.

```bash
rm -rf ~/.composer-credentials/*
composer identity import -p ibmcs -u PeerAdmin -c cert.pem -k key.pem
```

## Create and deploy the business network

Make sure that _composer-cli_ is version 0.13.1, or you will get compatibility errors. Run the following commands from the transaction_backend directory:

```bash
composer archive create -a hmlr-network.bna -t dir -n hmlr-network/
composer network deploy -a hmlr-network.bna -p ibmcs -i PeerAdmin -s anything
```

> **NB:** if you want to update an existing business network you need to use `composer network update`:

## Deploy the REST server

When the Composer REST server starts, it reads the model information from the business network which is deployed in the Blockchain.  Therefore you can't start it until _after_ you have deployed the business network.

That's the reason we didn't deploy the REST server along with all the other services; we're going to do that now.

Run the following command, adding in your business network name (that's the one defined in the _package.json_ file, not necessarily the file name).

```bash
./create/create_composer-rest-server.sh org-acme-biznet
```

Examine the running pods (either with the Kubernetes Dashboard or `kubectl get pods`) to see that the REST server has started.  View the logs from the Dashboard (or use `kubectl logs <container-name>`) to ensure that it is serving on port 3000 - this will be exposed externally as port 31090.

Now access the Composer REST server explorer; browse to http://your-ip-address:31090/explorer

**Congratulations!** You've successfully deployed a Hyperledger Fabric Blockchain and a business network to IBM Container Services, and exposed it as an API.

## Cleaning up

If you want to start from scratch you can remove the containers you deployed via Kubernetes with a single script.

```bash
cd scripts
./delete_all.sh
```

However there are some things left over so you should use your web browser to visit the kubernetes proxy described earlier and delete two Persistent Volume Claims (from the Overview page) and two Persistent Volumes (from the Persistent Volumes page).