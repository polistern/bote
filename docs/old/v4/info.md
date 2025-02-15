# I2P-Bote V4

## 1. Introduction

I2P-Bote is a serverless pseudonymous email exchange service for the I2P network. Emails are
stored encrypted in a Kademlia DHT formed by all I2P-bote nodes.
There is a web interface that lets the user send/read email and change settings.
An SMTP/POP interface to use with an email client is planned for the future.

Email can be sent through a number of other nodes (relays) for increased anonymity, or directly to
a set of storage nodes for faster delivery. Receiving email through relays is not yet implemented.
All nodes are created equal. There are no "supernodes" or designated relay/storage nodes.
Everybody acts as a potential relay and storage node. At some point in the future, the maximum
amount of disk space used for relayed/stored email packets will be user-configurable.
Before an email is sent, it is broken up into packets 30 kb or smaller and encrypted with the
recipient's public key. The packets are then stored in the DHT.

Email packets are stored redundantly in a Kademlia DHT (distributed hash table).
Stored email packets and relay packets are kept for at least 100 days, during which the recipient
can download them. If a node runs out of email storage space, and there are no old packets that
can be deleted, the node refuses storage requests.

Below is a diagram of how an email Packet is routed from a sender to a recipient:

```
                    .-------.        .-------.        .--------.
                    | Relay |  --->  | Relay |  --->  | Storer |  --------------------.
                _   `-------'        `-------'        `--------'                       `\
                /`                                                                       |
               /                                                                         |
 .--------.   /     .-------.        .-------.         .--------.                        |
 | Sender |  ---->  | Relay |  --->  | Relay |  --->  | Storer |  -------------.         |
 `--------'   \     `-------'        `-------'         `--------'               `\       |
               \                                                                  |      |
               _\/                                                                |      |
                    .-------.        .-------.        .--------.                  |      |
                    | Relay |  --->  | Relay |  --->  | Storer |  ------.         |      |
                    `-------'        `-------'        `--------'         `\       |      |
                                                                           |      |      |
                                                                           V      V      V

                                        .--------------- Kademlia DHT -------------------------.
                                        |                                .---------.           |
                                        |   .---------.                  | Storage |           |
                                        |   | Storage |     .---------.  |  Node   |           |
                                        |   |  Node   |     | Storage |  `---------'           |
                                        |   `---------'     |  Node   |                        |
                                        |                   `---------'            .---------. |
                                        |  .---------.   .---------.               | Storage | |
                                        |  | Storage |   | Storage |  .---------.  |  Node   | |
                                        |  |  Node   |   |  Node   |  | Storage |  `---------' |
                                        |  `---------'   `---------'  |  Node   |              |
                                        |                             `---------'              |
                                        `------------------------------------------------------'

                                                                          |      |      |
                     .-------.        .-------.        .---------.       /       |      |
                     | Relay |  <---  | Relay |  <---  | Fetcher |  <---'        |      |
                  /  `-------'        `-------'        `---------'               |      |
                 /                                                               |      |
               \/_                                                               |      |
 .-----------.       .-------.        .-------.        .---------.              /       |
 | Recipient |  <--  | Relay |  <---  | Relay |  <---  | Fetcher |  <----------'        |
 `-----------'  _    `-------'        `-------'        `---------'                      |
               |\                                                                       |
                 \                                                                      |
                  \  .-------.        .-------.        .---------.                     /
                     | Relay |  <---  | Relay |  <---  | Fetcher |  <-----------------'
                     `-------'        `-------'        `---------'
```

The number of relays for sending or retrieving an email is user-configurable.

For higher performance (but reduced anonymity), it is possible to use no relays and
no storers/fetchers, i.e. sender and recipient talk to the storage nodes directly. If both sender
and recipient chose not to use relays, the diagram looks like this:

```
 .--------.
 | Sender |  ----------------------------.
 `--------'                               `\
                                            |
                                            V

               .--------------- Kademlia DHT -------------------------.
               |                                .---------.           |
               |   .---------.                  | Storage |           |
               |   | Storage |     .---------.  |  Node   |           |
               |   |  Node   |     | Storage |  `---------'           |
               |   `---------'     |  Node   |                        |
               |                   `---------'            .---------. |
               |  .---------.   .---------.               | Storage | |
               |  | Storage |   | Storage |  .---------.  |  Node   | |
               |  |  Node   |   |  Node   |  | Storage |  `---------' |
               |  `---------'   `---------'  |  Node   |              |
               |                             `---------'              |
               `------------------------------------------------------'

                                            |
                                            |
 .-----------.                             /
 | Recipient |  <-------------------------'
 `-----------'
```

I2P-Bote uses base64 strings for addresses. They are called email destinations and can be
between 86 and 512 characters long, depending on the type_ of encryption the user chooses (see 7.1)

## 2. Kademlia

The Kademlia implementation used by I2P-Bote differs from standard Kademlia in several ways:

 * Items can be deleted from the DHT
 * No caching of DHT items because they are only retrieved once and then deleted
 * I2P-Bote uses sibling lists (s-buckets) as suggested in the S/Kademlia paper

There are three types of data that is stored in the DHT: Email Packets, Index Packets, and Contacts.

Index packets contain the DHT keys of one or more email packets. The DHT key of an index Packet is the SHA-256 hash of the Email Destination the email packets are destined for.   
To check for new email for a given Email Destination, I2P-Bote first queries the DHT for index packets for that Email Destination, then queries the DHT for all email Packet keys in the index packets.  
When a complete set of email packets has been received, the email is reconstructed and placed in the inbox.

TODO Contacts

To delete an email Packet, or an entry in an index Packet, the delete request must contain the correct verification code (the delete verification hash).   
The verification code is the SHA-256 hash of another block of data called the delete authorization which is encrypted in the email Packet.   
This means no third party can delete the Packet until the recipient has decrypted the Packet and sent out a delete request containing the delete authorization.   
Note that once a delete request for a given email Packet has been received by a node that is storing the email Packet, the node knows the delete authorization code and can propagate the delete request to other nodes that don't know yet that the Packet has been deleted.

To delete an index Packet entry, a delete verification hash is required as well. It is the same hash as the one for the email Packet which the index Packet entry points to.

## 3. Packet Types

As of release 0.2.5, the protocol version is 4. It is incompatible to earlier versions.   
Version 4 clients will talk to higher versions. There will be at least one more protocol version at some point in the future; this version will be backwards compatible to version 4.   
All packets start with one byte for the Packet type_, followed by one byte for the Packet version.   
Strings in packets are UTF8 encoded.   

### 3.1. Data Packets

These are always sent wrapped in a Communication Packet (see 3.2).   
E denotes encrypted data.   
For supported encryption and signature algorithms, see 7.1   

```
   Packet Type        | Field | Data Type  | Description
  --------------------+-------+------------+--------------------------------------------------------
   Email Packet,      |       |            | An email or email fragment, one recipient. Part of
   encrypted          |       |            | the Packet is encrypted with the recipient's key.
                      | TYPE  | 1 byte     | Value = 'E'
                      | VER   | 1 byte     | Protocol version
                      | KEY   | 32 bytes   | DHT key of the Packet, SHA-256 hash of LEN+DATA
                      | TIM   | 4 bytes    | The time the Packet was stored on a storage node
                      | DV    | 32 bytes   | Delete verification hash, SHA-256 hash of DA
                      | ALG   | 1 byte     | ID number of the encryption algorithm used
                      | LEN   | 2 bytes    | Length of DATA
                      | DATA  | byte[]   E | Data for decryption and Unencrypted Email Packet
                      |       |            | ToDo: add info about alg-specific prefixes
  --------------------+-------+------------+--------------------------------------------------------
   Email Packet,      |       |            | Storage format for the Incomplete Email Folder
   unencrypted        |       |            | 
                      | TYPE  | 1 byte     | Value = 'U'
                      | VER   | 1 byte     | Protocol version
                      | MSID  | 32 bytes   | Message ID in binary format
                      | DA    | 32 bytes   | Delete authorization key, randomly generated
                      | FRID  | 2 bytes    | Fragment Index of this Packet (0..NFR-1)
                      | NFR   | 2 bytes    | Number of fragments in the email
                      | MLEN  | 2 bytes    | Length of the MSG field
                      | MSG   | byte[]     | email content (MLEN bytes)
  --------------------+-------+------------+--------------------------------------------------------
   Index Packet       |       |            | Contains the DHT keys of one or more Email Packets,
                      |       |            | a Delete Verification Hash, and time stamp for each
                      |       |            | DHT key.
                      | TYPE  | 1 byte     | Value = 'I'
                      | VER   | 1 byte     | Protocol version
                      | DH    | 32 bytes   | SHA-256 hash of the recipient's email destination
                      | NP    | 4 bytes    | Number of entries in the Packet
                      | KEY1  | 32 bytes   | DHT key of the first Email Packet
                      | DV1   | 32 bytes   | Delete verification hash for KEY1
                      |       |            | (SHA-256 hash of DA in the email Packet)
                      | TIM1  | 4 bytes    | The time the key KEY1 was added
                      | KEY2  | 32 bytes   | DHT key of the second Email Packet
                      | DV2   | 32 bytes   | Delete verification hash for KEY2
                      |       |            | (SHA-256 hash of DA in the email Packet)
                      | TIM2  | 4 bytes    | The time the key KEY2 was added
                      | ...   | ...        | ...
                      | KEYn  | 32 bytes   | DHT key of the n-th Email Packet
                      | DVn   | 32 bytes   | Delete verification hash for KEYn
                      |       |            | (SHA-256 hash of DA in the email Packet)
                      | TIMn  | 4 bytes    | The time the key KEYn was added
  --------------------+-------+------------+--------------------------------------------------------
   Deletion Info      |       |            | Contains information about deleted  DHT items, which
   Packet             |       |            | can be Email Packets or Index Packet entries.
                      | TYPE  | 1 byte     | Value = 'T'
                      | VER   | 1 byte     | Protocol version
                      | NP    | 4 bytes    | Number of entries in the Packet
                      | KEY1  | 32 bytes   | First DHT key
                      | DA1   | 32 bytes   | Delete Authorization for KEY1
                      | TIM1  | 4 bytes    | The time the key KEY1 was added
                      | KEY2  | 32 bytes   | Second DHT key
                      | DA2   | 32 bytes   | Delete Authorization for KEY2
                      | TIM2  | 4 bytes    | The time the key KEY2 was added
                      | ...   | ...        | ...
                      | KEYn  | 32 bytes   | The n-th DHT key
                      | DAn   | 32 bytes   | Delete Authorization for KEYn
                      | TIMn  | 4 bytes    | The time the key KEYn was added
  --------------------+-------+------------+--------------------------------------------------------
   Peer List          |       |            | Response to a Find Close Peers Request or a
                      |       |            | Peer List Request
                      | TYPE  | 1 byte     | Value = 'L'
                      | VER   | 1 byte     | Protocol version
                      | NUMP  | 2 bytes    | Number of peers in the list
                      | P1    | 384 bytes  | Destination key
                      | P2    | 384 bytes  | Destination key
                      | ...   | ...        | ...
                      | Pn    | 384 bytes  | Destination key
  --------------------+-------+------------+--------------------------------------------------------
   Directory Entry    |       |            | A name/Email Destination mapping
   (Contact.java)     | TYPE  | 1 byte     | Value = 'C'
                      | VER   | 1 byte     | Protocol version
                      | KEY   | 32 bytes   | DHT key (SHA-256 hash of the UTF8-encoded name in
                      |       |            | lower case)
                      | DLEN  | 2 bytes    | Length of the DEST field
                      | DEST  | byte[]     | Email destination, 8 bit encoded (not base64)
                      | SALT  | 4 bytes    | A value such that scrypt(KEY|DEST|SALT) ends with
                      |       |            | a zero byte
                      | PLEN  | 2 bytes    | Length of PIC, max 7500
                      | PIC   | byte[]     | User picture in a browser-compatible format
                      |       |            | (JPG, PNG, GIF)
                      | COMP  | 1 byte     | Compression type_ for TEXT
                      | TLEN  | 2 bytes    | Length of the TEXT field, max 2000
                      | TEXT  | byte[]     | User defined text in UTF8
  --------------------+-------+------------+--------------------------------------------------------
```

### 3.2. Communication Packets

Communication packets are used for sending data between two I2P-Bote nodes.   
They contain a data Packet, see 3.1.   
All Communication Packets start with a four-byte prefix, followed by one byte for the Packet type, one byte for the Packet version, and a 32-byte Correlation ID.   

```  
   Packet Type        | Field | Data Type  | Description
  --------------------+-------+------------+--------------------------------------------------------
   Relay Request      |       |            | A Packet that tells the receiver to communicate with
                      |       |            | a peer, or peers, on behalf of the sender. It contains
                      |       |            | an I2P destination, a delay time, and a Communication
                      |       |            | Packet.
                      | PFX   | 4 bytes    | Packet prefix, must be 0x6D 0x30 0x52 0xE9
                      | TYPE  | 1 byte     | Value = 'R'
                      | VER   | 1 byte     | Protocol version
                      | CID   | 32 bytes   | Correlation ID, used for responses
                      | HLEN  | 2 bytes    | HashCash length
                      | HK    | byte[]     | HashCash token (HLEN bytes)
                      | DLY   | 4 bytes    | Delay in seconds before sending
                      | NEXT  | 384 bytes  | Destination to forward the Packet to
                      | RET   | Ret. Chain | An empty or non-empty Return Chain
                      | DLEN  | 2 bytes    | Length of the DATA field in bytes
                      | DATA  | byte[]   E | Encrypted with the recipient's public key. Can contain
                      |       |            | another Relay Request, a Store Request, or a Deletion
                      |       |            | Query.
                      | PAD   | byte[]     | Optional padding
  --------------------+-------+------------+--------------------------------------------------------
   Relay Return       |       |            | Contains a Return Chain and a payload_. The final
   Request            |       |            | destination is unknown to the sender or any of the
   (not implemented)  |       |            | hops. Used for responding to a relayed data Packet.
                      | PFX   | 4 bytes    | Packet prefix, must be 0x6D 0x30 0x52 0xE9
                      | TYPE  | 1 byte     | Value = 'K'
                      | VER   | 1 byte     | Protocol version
                      | CID   | 32 bytes   | Correlation ID, used for confirmation between two
                      |       |            | relay peers. A new correlation ID is set at every hop.
                      | RET   | Ret. Chain | A non-empty Return Chain
                      | DLEN  | 2 bytes    | Length of the DATA field in bytes
                      | DATA  | byte[]   E | The payload. AES256-encrypted at every hop in the
                      |       |            | return chain.
  --------------------+-------+------------+--------------------------------------------------------
   Return Chain       |       |            | Not a Packet itself, but part of a Relay (Return)
   (not implemented)  |       |            | Request
                      | RL1   | 2 bytes    | Length of RET1. Zero means an empty return chain,
                      |       |            | nothing follows.
                      | RET1  | RL1 bytes  | Hop 1 in the return chain. Contains a zero byte,
                      |       |            | followed by the hop's I2P destination and an AES key
                      |       |            | to encrypt the payload_ with.
                      | RL2   | 2 bytes   E| Length of RET2.
                      | RET2  | RL2 bytes E| Hop 2 in the return chain. Contains a zero byte,
                      |       |            | followed by the hop's I2P destination and an AES key
                      |       |            | to encrypt the payload_ with. This field is encrypted
                      |       |            | with the PK of hop 2.
                      | RL3   | 2 bytes   E| Length of RET3.
                      | RET3  | RL3 bytes E| Hop 3 in the return chain. Contains a zero byte,
                      |       |            | followed by the hop's I2P destination and an AES key
                      |       |            | to encrypt the payload_ with. This field is encrypted
                      |       |            | with the PKs of hops 3 and 2.
                      | ...   | ...        | ...
                      | RLn   | 2 bytes   E| Length of RETn.
                      | RETn  | RLn bytes E| Last hop in the return chain. Contains a zero byte,
                      |       |            | followed by the hop's I2P destination and an AES key
                      |       |            | to encrypt the payload_ with. This field is encrypted
                      |       |            | with the PKs of hops n...2.
                      | RLn+1 | 2 bytes   E| Length of RETn+1.
                      | RETn+1| RLn+1 byt E| Number of hops (>0) followed by all AES256 keys
                      |       |            | (one per hop)
                      | RLn+2 | 2 bytes    | Value=0 to indicate this is the end of the return chain
  --------------------+-------+------------+--------------------------------------------------------
   Fetch Request      |       |            | Request to a chain endpoint to retrieve an Index 
   (not implemented)  |       |            | Packet or Email Packet.
                      | PFX   | 4 bytes    | Packet prefix, must be 0x6D 0x30 0x52 0xE9
                      | TYPE  | 1 byte     | Value = ''
                      | VER   | 1 byte     | Protocol version
                      | CID   | 32 bytes   | Correlation ID, used for responses
                      | DTYP  | 1 byte     | Type of data to retrieve
                      | KEY   | 32 bytes   | DHT key to look up
                      | KPR   | 384 bytes  | Email keypair of recipient
                      | RLEN  | 2 bytes    | Length of the RET field
                      | RET   | byte[]     | Relay Packet, contains the return chain (RLEN bytes)
  --------------------+-------+------------+--------------------------------------------------------
   Response Packet    |       |            | Response to a Retrieve Request, Fetch Request,
                      |       |            | Find Close Peers Request, or a Peer List Request
                      | PFX   | 4 bytes    | Packet prefix, must be 0x6D 0x30 0x52 0xE9
                      | TYPE  | 1 byte     | Value = 'N'
                      | VER   | 1 byte     | Protocol version
                      | CID   | 32 bytes   | Correlation ID of the request Packet
                      | STA   | 1 byte     | Status code:
                      |       |            |   0 = OK
                      |       |            |   1 = General error
                      |       |            |   2 = No data found
                      |       |            |   3 = Invalid Packet
                      |       |            |   4 = Invalid HashCash
                      |       |            |   5 = Not enough HashCash provided
                      |       |            |   6 = No disk space left
                      | DLEN  | 2 bytes    | Length of the DATA field; can be 0 if no payload
                      | DATA  | byte[]     | A Data Packet
  --------------------+-------+------------+--------------------------------------------------------
   Peer List Request  |       |            | A request for a list of
                      |       |            | high-reachability relay peers
                      | PFX   | 4 bytes    | Packet prefix, must be 0x6D 0x30 0x52 0xE9
                      | TYPE  | 1 byte     | Value = 'A'
                      | VER   | 1 byte     | Protocol version
                      | CID   | 32 bytes   | Correlation ID, used for responses
  --------------------+-------+------------+--------------------------------------------------------
```

####  DHT Communication Packets
  
```
   Packet Type        | Field | Data Type  | Description
  --------------------+-------+------------+--------------------------------------------------------
   Retrieve Request   |       |            | A request to a peer to return a data item for a given
                      |       |            | DHT key and data type
                      | PFX   | 4 bytes    | Packet prefix, must be 0x6D 0x30 0x52 0xE9
                      | TYPE  | 1 byte     | Value = 'Q'
                      | VER   | 1 byte     | Protocol version
                      | CID   | 32 bytes   | Correlation ID, used for responses
                      | DTYP  | 1 byte     | Type of data to retrieve:
                      |       |            |   'I' = Index Packet
                      |       |            |   'E' = Email Packet
                      |       |            |   'C' = Directory Entry
                      | KEY   | 32 bytes   | DHT key to look up
  --------------------+-------+------------+--------------------------------------------------------
   Deletion Query     |       |            | A request to a peer to return a Deletion Info Packet 
                      |       |            | for a given Email Packet DHT key. If the Email Packet 
                      |       |            | is not known to be deleted, no response is expected
                      |       |            | back.
                      | PFX   | 4 bytes    | Packet prefix, must be 0x6D 0x30 0x52 0xE9
                      | TYPE  | 1 byte     | Value = 'Y'
                      | VER   | 1 byte     | Protocol version
                      | CID   | 32 bytes   | Correlation ID, used for responses
                      | KEY   | 32 bytes   | DHT key to look up
  --------------------+-------+------------+--------------------------------------------------------
   Store Request      |       |            | DHT Store Request
                      | PFX   | 4 bytes    | Packet prefix, must be 0x6D 0x30 0x52 0xE9
                      | TYPE  | 1 byte     | Value = 'S'
                      | VER   | 1 byte     | Protocol version
                      | CID   | 32 bytes   | Correlation ID, used for responses
                      | HLEN  | 2 bytes    | HashCash length
                      | HK    | byte[]     | HashCash token (HLEN bytes)
                      | DLEN  | 2 bytes    | Length of the DATA field
                      | DATA  | byte[]     | Data Packet to store (DLEN bytes).
                      |       |            | Can be an Index Packet, Email Packet, or Email
                      |       |            | Destination.
  --------------------+-------+------------+--------------------------------------------------------
   Email Packet       |       |            | Request to delete an Email Packet by DHT key
   Delete Request     |       |            |
                      | PFX   | 4 bytes    | Packet prefix, must be 0x6D 0x30 0x52 0xE9
                      | TYPE  | 1 byte     | Value = 'D'
                      | VER   | 1 byte     | Protocol version
                      | CID   | 32 bytes   | Correlation ID, used for responses
                      | KEY   | 32 bytes   | DHT key of the Email Packet to delete
                      | DA    | 32 bytes   | Delete Authorization (SHA-256 must equal DV in the
                      |       |            | email pkt)
  --------------------+-------+------------+--------------------------------------------------------
   Index Packet       |       |            | Request to remove one or more entries (Email Packet 
   Delete Request     |       |            | keys) from an Index Packet
                      | PFX   | 4 bytes    | Packet prefix, must be 0x6D 0x30 0x52 0xE9
                      | TYPE  | 1 byte     | Value = 'X'
                      | VER   | 1 byte     | Protocol version
                      | CID   | 32 bytes   | Correlation ID, used for responses
                      | DH    | 32 bytes   | The Email Destination hash of the Index Packet
                      | N     | 1 byte     | Number of entries in the Packet
                      | DHT1  | 32 bytes   | First DHT key to remove
                      | DA1   | 32 bytes   | Delete Authorization (SHA-256 must equal DV in the 
                      |       |            | email Packet referenced by DHT1)
                      | DHT2  | 32 bytes   | Second DHT key to remove
                      | DA2   | 32 bytes   | Delete Authorization (SHA-256 must equal DV in the 
                      |       |            | email Packet referenced by DHT2)
                      | ...   | ...        | ...
                      | DHTn  | 32 bytes   | n-th DHT key to remove
                      | DAn   | 32 bytes   | Delete Authorization (SHA-256 must equal DV in the 
                      |       |            | email Packet referenced by DHTn)
  --------------------+-------+------------+--------------------------------------------------------
   Find Close Peers   |       |            | Request for k peers close to a key
                      | PFX   | 4 bytes    | Packet prefix, must be 0x6D 0x30 0x52 0xE9
                      | TYPE  | 1 byte     | Value = 'F'
                      | VER   | 1 byte     | Protocol version
                      | CID   | 32 bytes   | Correlation ID, used for responses
                      | KEY   | 32 bytes   | DHT key
  --------------------+-------+------------+--------------------------------------------------------
```

## 4. Protocols

### 4.1. Storing a DHT item via relays

I2P nodes involved: A=Sender, R1...Rn=Relays, S1...Sm=Storage Nodes

1. A onion-encrypts the Store Request with the public keys of all hops, resulting in a Relay Packet.
2. A sends the Relay Packet to R1.
3. R1 decrypts the Packet, waits a random amount of time, and sends it to R2.
4. R2 confirms delivery with R1.
5. Repeat until Packet arrives at Rn.
6. Rn decrypts the Relay Packet into an Email Packet.
7. Rn sends the Packet to S1,...,Sm through a Kademlia STORE

### 4.2. Return Chains

In order to make it impossible for two non-adjacent nodes on the return chain to identify Relay Return Request packets, the payload_ and the entire Return Chain are re-encrypted at every hop.   
A new Correlation ID is generated at every hop.   
Here is a simplified diagram of how a Relay Return Packet is sent from one hop to the next:   

(A) The Packet is received. HOP1 through HOP3 contain encrypted data (encrypted with the public key for the receiving I2P node):

```
      
      +------------------- Relay Return Request --------------------------+
      |                                                                   |
      |                                .-----KEY1-----.                   |
      |                .---KEY1---.   |  .---KEY2---.  |                  |
      |     .-KEY1-.  |  .-KEY2-.  |  | |  .-KEY3-.  | |  .---------.     |
      |     | HOP1 |  |  | HOP2 |  |  | |  | HOP3 |  | |  | PAYLOAD |     |
      |     `------’  |  `------’  |  | |  `------’  | |  `---------’     |
      |                `----------’   |  `----------’  |                  |
      |                                `--------------’                   |
      |                                                                   |
      +-------------------------------------------------------------------+
      
      KEYx denotes a layer of encryption.
```

(B) The receiving node decrypts HOP1, HOP2, and HOP3. HOP1 is now in plain text. It contains thedestination of the next hop and an AES key.

```  
      +------------------- Relay Return Request --------------------------+
      |                                                                   |
      |                                .-----KEY2-----.                   |
      |                .---KEY2---.   |  .---KEY3---.  |                  |
      |     .------.  |  .------.  |  | |  .------.  | |  .---------.     |
      |     | HOP1 |  |  | HOP2 |  |  | |  | HOP3 |  | |  | PAYLOAD |     |
      |     `------’  |  `------’  |  | |  `------’  | |  `---------’     |
      |                `----------’   |  `----------’  |                  |
      |                                `--------------’                   |
      |                                                                   |
      +-------------------------------------------------------------------+
```
(C) HOP1 is replaced with random data and moved to the end of the chain. The payload is encrypted with the AES key from HOP1:

```
      +------------------- Relay Return Request --------------------------+
      |                                                                   |
      |                      .-----KEY2-----.                             |
      |      .---KEY2---.   |  .---KEY3---.  |                            |
      |     |  .------.  |  | |  .------.  | |  .------.  .---AES---.     |
      |     |  | HOP3 |  |  | |  | HOP2 |  | |  | HOP1 |  | PAYLOAD |     |
      |     |  `------’  |  | |  `------’  | |  `------’  `---------’     |
      |      `----------’   |  `----------’  |                            |
      |                      `--------------’                             |
      |                                                                   |
      +-------------------------------------------------------------------+
```

(D) The Packet is sent to the next node in the chain.

### 4.3. Retrieving DHT items via relays (not implemented yet)

I2P nodes involved:

- A=Address Owner
- F=Fetcher
- O1...On=Outbound Relays
- I1...In=Inbound Relays,
- S1...Sm=Storage Nodes

1.  A builds a Relay Request containing a Retrieve Request and sends it to F via O1, O2, ..., On as described in 4.1
2.  F queries the DHT for the DHT item
3.  F encrypts the DHT data with the public key for hop 1 of the return chain
4.  F moves the (RL1,RET1) block of the return chain to the end (behind RETn+1)
5.  F sends the results of steps 3) and 4) to I1
6.  I1 decrypts all (RLx,RETx) blocks of the return chain and finds I2 and the AES key in the first block
7.  I1 replaces (RL1,RET1) with random data that is encrypted with the PK for I2
8.  I1 moves the (RL1,RET1) block of the return chain to the end (behind RETn+1)
9.  I1 encrypts the payload_ with the AES key
10. I1 sends the results of steps 7) and 8) to I2
11. Repeat from 6) for I2, I3, ..., until the data reaches A (when the first byte of RET1 is non-zero)
12. A decrypts the (RL1,RET1) block which contains all AES keys
13. A decrypts the payload_ with the AES keys, starting from the last hop's key
14. The decrypted data contains the DHT data Packet

### 4.4. Deleting an Email Packet

Every time a recipient receives an email Packet directly from one or more storage nodes, it asks the storage node(s) to delete the Packet.   
If the email Packet is received through one or more relays, the recipient issues a delete request to a (possibly different) relay endpoint, which entails an additional findClosestNodes lookup because the endpoint has to find out which nodes are storing the Packet.   
  
The recipient also sends a delete request for index Packet, asking the storage nodes to delete the DHT key of the email Packet.   
Each index Packet node then removes the DHT key from the stored index Packet, and adds the DHT key to a list of deleted DHT keys it maintains for each Index Packet (plus the delete verification hash).   
The purpose of this is so a node that is about to replicate an email Packet can find out if it missed an earlier delete request for that Packet, in which case the node "replicates" the delete request rather than the Packet itself.   
This helps reduce storage space by removing old Email Packets from nodes that weren't online at the time the delete request was sent initially.
  
### 4.5 Deleting an Index Packet Entry

Similar to deleting an email Packet.

### 4.6. Replication

See comments at the beginning of src/i2p/bote/network/kademlia/ReplicateThread.java

## 5. Algorithms Used By Nodes Locally

### 5.1. Retrieving Email via relays

See also http://forum.i2p/viewtopic.php?p=19927#19927 ff.
   
1. Randomly choose a set of n relay nodes,  R1...Rn (outgoing chain for request)
2. Randomly choose a set of m relay nodes, S1...Sm. (return chain / inbound mail chain)

  - with Sm being the node closest to the receiver and S1 the chain node closest to fetcher

3. Generate m+1 random 32-byte arrays (for XORing the return packets), X1...Xm+1
    (This is done so the outgoing chain's endpoint or inbound first hop need
    not know all inbound hops in order to perform hop-to-hop encryption. Each
    hop will decrypt one 32-byte random array and encrypt the Packet with it,
    so it looks different at each hop without one node at the beginning
    knowing all hops)
4. With Sm's public key, encrypt the local destination key and Xm+1
5. With Sm-1's public key, encrypt Sm's destination key, Xm, and the data from step 4)
6. With Sm-2's public key, encrypt Sm-1's destination key, Xm-1, and the data from step 5)
7. Repeat until the entire return chain, S1...Sm, has been onion-encrypted, and also include a delay for each hop.
8. Add S1's destination key and X1 to the data from step 7)
9. Add the data (which is a Relay Packet) to a Retrieve Request Packet.
10. Encrypt the resulting Packet, using the public keys of relays Rn (Fetcher), Rn-1, ..., R1, and add the destination key of the next relay each time. Also included is a delay for each relay.
11. Set a random Correlation ID on the Packet at each layer
12. Add the Packet to the relay Packet folder.
13. When a reply Packet is received, decrypt it first with the local node's privatedecryption key, then with all the random keys X1...Xm+1.

## 6. Files

### 6.1. Identities file

Stores all email identities the local user has created. The file uses the Java Properties format, and can be read into / written out from a Java Properties object.

The following property keys are currently stored and recognized:

* identity#.publicName  - The public name of the identity, included in emails.
* identity#.key         - Base64 of the identity keys.
* identity#.salt        - Salt used to generate a fingerprint.
* identity#.description - Description of the identity, only displayed locally.
* identity#.picture     - Base64 of byte[] containing picture data
* identity#.text        - Text associated with the identity.
* identity#.published   - Has the identity been published to the public addressbook?
  (# is an integer index)

The base64 key contains two public keys (encryption+signing) and two private keys.

The identities file can optionally contain the property key "default", with its value set to an Email Destination (i.e. two public keys).   
If the Email Destination matches one of the identities, that identity is used as the default.

### 6.2. Included .jar files and third-party source code

```
bcprov-jdk15on-152.jar     The BouncyCastle library and SecurityProvider.
mailapi-1.5.4.jar          Part of JavaMail 1.5.4
lzma-9.20.jar              An LZMA implementation from http://www.7-zip.org/sdk.html
ntruenc-1.2.jar            An NTRU implementation from http://sf.net/projects/ntru/
                           (only the NTRUEncrypt part, not NTRUSign)
flexi-gmss-1.7p1.jar       A stripped down and patched version of the FlexiProvider
                           sources (http://www.flexiprovider.de/), containing only the
                           classes needed for GMSS.
                           The patch consists of a modified de.flexiprovider.core.CoreRegistry
                           and de.flexiprovider.api.Registry and its purpose is to
                           reduce dependencies on classes not related to GMSS.
scrypt-1.4.0.jar           An scrypt implementation from https://github.com/wg/scrypt/
```

## 7. Cryptography

### 7.1. Asymmetric encryption and signing

Supported encryption and signature algorithms:

* ALG=1: ElGamal-2048 / DSA-1024 / AES-256 / SHA-256 (same as the I2P router uses)
* ALG=2: ECDH-256 / ECDSA-256 / AES-256 / SHA-256
* ALG=3: ECDH-521 / ECDSA-521 / AES-256 / SHA-512
* ALG=4: NTRUEncrypt-1087 / GMSS-512 / AES-256 / SHA-512

### 7.2. Password Encryption

The identities file, the address book, and all email files are encrypted with AES-256.   
To generate the AES key from the password, scrypt (http://www.tarsnap.com/scrypt.html) is used.   
The file format is:

```
     Field | #bytes | Description
    -------+--------+------------------------------------------------------------------
     SOF   | 4      | Start of file, contains the characters "IBef"
     VER   | 1      | Format version, must be 1
     N     | 4      | scrypt CPU cost parameter
     r     | 4      | scrypt memory cost parameter
     p     | 4      | scrypt parallelization parameter
     SALT  | 32     | Salt for scrypt
     IV    | 32     | IV for AES
     DATA  |        | The encrypted data
```

Parameters N through SALT are cached in a file named derivparams so all encrypted files can use the same key derivation parameters.   
This makes decryption much faster because the key only needs to be derived once per session.

### 7.3. Fingerprints For Directory Entries

TODO
H = scrypt(name, dest, zuf.wert); die letzten 8 Binärstellen von H müssen 0 sein
`13*7+22+18 = 131`

## 9. Glossary of Terms

### Email Destination

As the name implies, an Email Destination is an identifier by which somebody can be reached via I2P-Bote.   
An Email Destination is a Base64 string containing a public encryption key and a signature verification key.   
Example of a 512-character Email Destination (ElGamal-2048/DSA-1024):

```
  uQtdwFHqbWHGyxZN8wChjWbCcgWrKuoBRNoziEpE8XDt8koHdJiskYXeUyq7JmpG
  In8WKXY5LNue~62IXeZ-ppUYDdqi5V~9BZrcbpvgb5tjuu3ZRtHq9Vn6T9hOO1fa
  FYZbK-FqHRiKm~lewFjSmfbBf1e6Fb~FLwQqUBTMtKYrRdO1d3xVIm2XXK83k1Da
  -nufGASLaHJfsEkwMMDngg8uqRQmoj0THJb6vRfXzRw4qR5a0nj6dodeBfl2NgL9
  HfOLInwrD67haJqjFJ8r~vVyOxRDJYFE8~f9b7k3N0YeyUK4RJSoiPXtTBLQ2RFQ
  gOaKg4CuKHE0KCigBRU-Fhhc4weUzyU-g~rbTc2SWPlfvZ6n0voSvhvkZI9V52X3
  SptDXk3fAEcwnC7lZzza6RNHurSMDMyOTmppAVz6BD8PB4o4RuWq7MQcnF9znElp
  HX3Q10QdV3omVZJDNPxo-Wf~CpEd88C9ga4pS~QGIHSWtMPLFazeGeSHCnPzIRYD
```

Example of a 86-character Email Destination (ECC-256):

```
  1Lcvly8no5of6juJKxqy-xA-MStM2c2XKorepH1oqs5yKBkg9-ZcG4G4kZY1E~2672cMA806l9EicQLmlehB1m
```

### Email Address

Email Addresses in I2P-Bote are shortcuts for Email Destinations.   
Email Address <--> Email Destination mappings are stored in two places: the local address book and the distributed address directory.

### Email Identity

An Email Identity is an Email Destination plus a name you want to be known as when you use that identity.   
Technically speaking, an Email Identity consists of four things:

* An Email Destination (i.e. two public keys)
* The two private keys for the Email Destination
* A public name which is shown to other people in emails
* A description which is not shown to anybody but you.  
  It helps you remember which Email Identity you use for which purpose.

An email identity is not required for sending emails (in that case only "Anonymous" can be selected in the "sender" field).

### I2P destination

The address of a I2P-Bote node on the I2P network. There is normally no need to know it.   
I2P destinations and Email Destinations look similar, but are completely independent of each other.
