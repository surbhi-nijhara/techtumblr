# Data Encryption in Mongo DB

## Purpose
The purpose of this document is to understand how data can be encrypted in Mongo database. 
We will see encryption at client layer and the database layers. 


![\[Diagram for Encryption:\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/momgodb-encryption/diag_source/ms-sql-cdc-app.jpg?raw=true)

#### Client-Side Field Level Encryption: 
##### Methodology:
Mongodb supports Client-Side Encryption and it provides th encryption at a field level granularity. 
MongoAtlas supports Automatic Client-Side Field Level Encryption with help of mongocryptd. A Schema is required for Automatic Field Level Encryption which is represented in the extended version of JSON Schema. The schema contains the following information
The algorithm
The data encryption key
BSON Type of the field ( required for Deterministic Encrypted Fields).
This schema can be configured at 
a) Client Application level: This is enforced at the MongoDb driver level and automatically encrypts or decrypts the data. Example is here and can be seen line 80 onwards.
b) Database level. If the client fails to encrypt the data, the driver will retrieve the schema configured on the server side and use it to perform automatic encryption.


##### Keys:

There are two types of keys that are used in encryption.
1) Customer Master Key - CMK
2) Data Encryption Key - DEK 
   Data Encryption key is also called as secret key. 
   
The client Application will deal with encryption and decryption of data using a DEK.
DEK will be secured using Customer Master Key (CMK). Data Encryption Key is stored in MongoDB vault.  It is essentially saved as a collection in Mongo DB. 

CMK in turn will be secured using KMS. Currently MongoDB driver natively supports only AWS KMS(preferred) or local KMS. For any other key manager, the approach would be to male remote service calls and then be treated as local.

For multi-tenant systems, it is recommended separate CMK for each tenant. 


#### Encryption at Rest: 
MongoDB already supports disk Encryption at Rest by default.