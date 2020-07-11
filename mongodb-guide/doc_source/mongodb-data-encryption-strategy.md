# Data Encryption in Mongo DB

## Purpose
The purpose of this document is to understand how data can be encrypted in Mongo database. 
We will see encryption at client side encryption and server side encryption. 


![\[Diagram for Encryption:\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/mongodb-guide/diag_source/mongodb-encryption.jpg?raw=true)

#### Client-Side Field Level Encryption: 
#### Methodology:


Mongodb supports **Client-Side Encryption** and it provides the encryption at a field level granularity. 
MongoAtlas supports Automatic Client-Side Field Level Encryption with help of [mongocryptd](https://docs.mongodb.com/manual/reference/security-client-side-encryption-appendix/#mongocryptd). 
A Schema is required for [Automatic Field Level Encryption](https://docs.mongodb.com/manual/core/security-automatic-client-side-encryption/#field-level-encryption-automatic) which is represented in the extended version of JSON Schema. The schema contains the following information

* The algorithm

* The data encryption key

* BSON Type of the field ( required for Deterministic Encrypted Fields).

This schema can be configured at
 
a) **Client Application level**: This is enforced at the MongoDb driver level and automatically encrypts or decrypts the data. Example is [here](https://github.com/mongodb/mongo-java-driver/blob/master/driver-sync/src/examples/tour/ClientSideEncryptionAutoEncryptionSettingsTour.java) and can be seen line 80 onwards.

b) **Database level**. If the client fails to encrypt the data, the driver will retrieve the schema configured on the server side and use it to perform automatic encryption.

![\[Diagram for Mongo CLient SIde Encryption:\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/mongodb-guide/diag_source/mongodb-clientside-encrypt.png?raw=true)

 Source: https://docs.mongodb.com/manual/core/security-client-side-encryption/


#### Keys:

There are two types of keys that are used in encryption.

1) **Customer Master Key - CMK**

2) **Data Encryption Key - DEK **

Data Encryption key is also called as secret key. 
   
The client Application will deal with encryption and decryption of data using a DEK.

DEK will be secured using Customer Master Key (CMK). Data Encryption Key is stored in MongoDB vault.  Data Encryption Key is stored in MongoDB vault. It is essentially saved as a collection in Mongo DB. 


CMK in turn will be secured using [KMS](https://docs.mongodb.com/manual/core/security-client-side-encryption-key-management/). Currently MongoDB driver natively supports only AWS KMS(preferred) or local KMS. For any other key manager, the approach would be to make remote service calls and then be treated as local.

For multi-tenant systems, it is recommended separate CMK for each tenant. 


CMK: Connection Configuration - https://docs.mongodb.com/manual/reference/method/KeyVault.createKey/

Secret Key is the key(and not CMK) used to encrypt data using AES. Though Customer provided keys (CMK) can also be used to encrypt the data, it is not practiced.
CMK encrypted secret key using RSA - stronger but slower.



#### Server-Side Encryption
Encryption at Rest: 
MongoDB Enterprise version or cloud variant MongoAtlas supports disk Encryption at Rest by default.
Encryption key configure can be done.
KeyVault : MongoDB Key vault or Azure Key vault.




