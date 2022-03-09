---
title: Muridata v1 协议设计 (已经废弃)
date: 2022-03-08 10:54:26
---

这篇文章只是作为一个备份

<!--more-->

# MuriData Protocol

MuriData is a ==decentralized vector indexing and searching service==. With blockchain providing transaction and verification ability, It allows anyone to create vector databases and send searching tasks to it.

## Technology Stack
- BlockChain
- BFT Consensus
- Consistent hashing
- Distributed Hash Table
- Distributed Computing
- Distributed Storage
- Vector Indexing (Annoy Faiss NMSLib)
- And Some Kawaii Characters (Very important!!!)

## Glossary
- **MuriData** : The general term of this system.
- **Magu** : One vector database with index.
- **MurIndexer** : Single node in a **Magu** who perform searching tasks.
- **MuriChain** : The blockchain where muridata runs on.
- **MURI** : The token of this whole ecosystem.

## Introduction
MuriData is the name of the decentralized vector searching service. It includes a blockchain to organize nodes, give incentives and allocate tasks.

With MuriData, everyone could either publish a vector database or search on existing databases. Some specialized Indexing nodes will deal with queries and reach a consensus towards the best result.


## Design

### Layer 1 : Blockchain
The MuriData will run on the base of MuriChain.
MuriChain is designed to handle the transactions of the token **MURI**, which could be used as both a payment method of searching for data and a governance tool for the whole community.
There is no special requirement for the implementation of MuriChain. It could be a fork of either Ethereum or Polkadot etc. However, to allocate tasks, it must be able to track a list of **Magu** and perform incentives and punishments, which also requires a higher scalability and speed.

### Layer 2 : Magus
**Magu** refers to the Japanese version of "mug". In MuriData this means a cluster of nodes who maintain a corresponding database. Because in order to search for a vector, nodes must store the vector data and related index data, which is impossible when the vector database is extremely large. To increase the scalability, support big data, and also prevent the single point of failure (SPOF), the searching task must be run on a number of nodes. With **Magus**, searching tasks could be finished efficiently and effectively.

### Operations

#### Publishing a Magu (Vector Database)
The database is created when an account sends a transaction indicating the creation of one database, with the staking of some **MURI** Token and related data published at the network. The whole network would orgnize a **Magu** for this database and randomly choose some nodes to become its members, **MurIndexer**s. These nodes would form a small consensus group dealing with any tasks related to this database.

#### Becoming a MurIndexer
A node who wants to become a **MurIndexer** and provide indexing ability for the whole network will first stake some **MURI** with the promise of being online and store some data during a time interval. After staked some token, it would be on the list of candidates and allocated to a random **Magu**. It will start to store some vectors and perform tasks.

#### Searching for vectors
When a searching task is commited on the blockchain, the **Magu** would receive the task and start the searching process. Every **MurIndexer** would search for the nearst k vectors within its local data and send it to its **Magu**. Then all **MurIndexer**s will reach a consensus on the top k best result among those commitments and finally commit it on the blockchain. The task would finish and receive a result.

#### Fishing at Muri Network

> Fishers are the one who eventually being fished

Now Shrimp could eat the one who fish them. 

### Detalis

#### Vector Searching
A vector is [**a quantity or phenomenon that has two independent properties: magnitude and direction**](https://whatis.techtarget.com/definition/vector). In computer, it is stored as a series of numbers. 
To compare the similarity of two vectors, we need a method to calculate the distance between two vectors. There are several ways:
- Euclidean Distance
- Normalized Distance
- Hamming Distance

And also to speed up the process of searching a vector, there are many indexing methods:

- Annoy Index
- Hnsw Index

#### Distributed Computing
We could easily design a market of vector searching by simply just connect a vector searching server with its clients. However, there are several problems.

1. The vector searching server is a centralized form without any tolerance to SPOF, so the request may not be responded.
2. As a centralized server, it could not provide a searching service fast enough for use, due to the limited processing ability.
3. The result is totally controlled by just one server. There is nothing to prove the correctness of its result.

Unlike any other form of computation whose answer is definite, tasks like vector searching could only produce the nearest vector as result. Sometimes the nearest may not be find out because the index method is inaccurate. Therefore it is hard to prove that the result is the nearest. However, we could still get the answer as accurately as possible, by putting many nodes into competing:

1. Select many nodes in the network and form a smaller consensus group.
2. When task is received, every node start to work and find out the result.
3. All nodes in this group reach a consensus towards the answer by communication.

#### Distributed Storage
Just like the problems faced by distributed computing, the storage also has many difficulties to be deal with. It is inefficient to let every node store the same copy, while storing pieces of data in many node pulls the network into the risk of losing data. The way is to maintain several replications of one single piece of a data in a number of nodes, above a certain threshold. And when a node shutdown with its data, other node should rapidly get the copy of these data and store them. In MuriData system, we use **Consistent hashing** to achieve this.
The vector database will be broken into many pieces called 'segment', each segment contains a merkel tree to prove the existence of a vector, and has a unique hash to identify. Also, segments would be allocated to all nodes through consistent hashing.
All nodes joined to the small consensus group will be placed on a hash ring. Through calculation, the segment of vector database would get its location on the hash ring and then the host it will be stored on. One segment would have many replications on the hash ring. When one node quits, the data is also avaliable in other nodes. And to ensure the right number of copy exists, all node must frequently send proofs to the blockchain.


#### Distributed Hash Table
With the need of forming subgroups, nodes should be able to find and connect with each other. A DHT could be helpful.

#### Progress
```mermaid
sequenceDiagram

participant Baka
participant MuriChain
participant Magu
participant Node

Node ->> MuriChain : Stake to prove online.
Node ->> MuriChain : Node Info.
Baka ->> MuriChain : Register for a Magu.
MuriChain ->> MuriChain : Orgnize new Magu...
Baka ->> MuriChain : Add batch vectors..

MuriChain ->> Node : Allocate to a Magu..
activate Node
Node ->> Magu : Request for vector datas.
deactivate Node
Magu ->> Node : Vector datas.
Node ->> Node : Build Indexes.

loop Proof of Replicate
MuriChain ->> Node : Replicationg of vectors Challange.
activate Node
Node ->> MuriChain : Proof.
deactivate Node
MuriChain ->> Node: Reward for storing data.
end


Baka ->> MuriChain : Search for vector {...}
MuriChain ->> Magu : Task starts, input {...}

Magu ->> Node: Start searching task.
Node ->> Node: Find Nearst Vectors.
Node ->> Magu: Commit for best results.
loop Waiting for all nodes..
Magu ->> Magu : Ranking the results.
end

Magu ->> MuriChain: Commit the best K results.
Magu ->> MuriChain: Ranking of Nodes.
MuriChain ->> Node: Reward.

MuriChain ->> Baka: Final Result.
```

# MuriData Protocol Detailed Design

This document includes a possible implementation of the MuriData Protocol.

## BlockChain Details

All the consensus about **Magu**s and token incentives will be handled by **MuriChain**. It should be able to track a list of **Magu**s and their corresponding Vector Segments.

### Magus List

One Magu represents One database. A new Magu with special id and name would be created in the blockchain.
Its current state could be represented as:

| Id | Access Controller | Nodes | Vector Segments | Vector Dimensions | Node Quantity |
|----|-------------------|-------|-----------------|--------------------|------------|

***Name*** is specified by the creater.
***Access Controller*** refers to the account that can manage and add vectors to it.
***Nodes*** is the list of node in it.
***Vector Segments*** can be the list of all **vector segments** in it.

### Nodes List

Every Magu must have a list of nodes for allocating tokens and tasks.

| NodeId | NodeStatus | AccountPublicKey |
|--------|------------|------------------|

***NodeId*** is generated from the public key of a node, which is also used in DHT for peer discovery.
***NodeStatus*** indicates the status of this node, including its allowed disk spaces, computing ability and so on.
***AccountPublicKey*** is the account that controls the node. An account could be added to a node and is responsible for the behavior of this node.

### Vector Segments

| Merkel Root | Vector Data Source |
|-------------|--------------------|

A vector segment is identified by its ***Merkel Root***. It is also used to proof the existence of one vector. 
***Vector Data Source*** is the source of vector data, such as ipfs or other blockchain, usually being a hash pointer.

[Code Example](vector_segment.py)

## Searching for vector

When MurIndexers in a Magu receive a query from MuriChain, they will start searching for the nearst results for this vector. They search them in the index, adding to a list of vectors, then give each of them a merkle proof and signature. After finishing this work, they will send the hash of this list to their Magu. When all node send their result hash, they will send their result. Any node who refuse to send the hash in a specified time interval or send the vector list will be punished, and removed from this Magu.
Their vector list looks like this:

| Segment Merkle Root | Merkle Proof | Vector | Signature |
|---------------------|--------------|--------|-----------|

The node id will be also sent with this list.

When MurIndexer receive all the vector lists of their partners, they will start ranking the vectors in all of the lists. All node will get a ordered list of vectors. Then the first-finished node will propose its result in the Magu, starting a BFT consensus. If this proposal gets most of the signature in this group, it will be sent to MuriChain.

## Node Details

MuriData Node is sperated into two: the blockchain node and index node.
Blockchain node deals with blockchain and index node is responsible for performing tasks. 
Index node is connected with blockchain node. They communicate by rpc calls.

