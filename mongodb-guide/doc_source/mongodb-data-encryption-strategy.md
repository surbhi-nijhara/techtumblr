# Data Encryption in Mongo DB

## Purpose
The purpose of this document is to understand how data can be encrypted in Mongo database. Encryption can be done both at rest and at field level.
In this part-1, we will see encryption at field level.


#### Client-Side Field Level Encryption: 
#### Methodology:

Mongodb supports **Client-Side Encryption** and it provides the encryption at a field level granularity. <br/>
MongoDb Community and Enterprise edition supports manual field encryption, while automatic field encryption is only supported by Enterprise editions.<br/>
Enterprise edition includes MongoAtlas (Cloud managed AmongoDB Enterprise editions).<br/>

Lets first see <br/>
**Automatic Field Encryption**:
![\[Diagram for Encryption:\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/mongodb-guide/diag_source/mongodb-encryption.jpg?raw=true)

MongoDB Enterprise or MongoAtlas supports Automatic Client-Side Field Level Encryption with help of a daemon [mongocryptd](https://docs.mongodb.com/manual/reference/security-client-side-encryption-appendix/#mongocryptd). <br/>
A Schema is required for [Automatic Field Level Encryption](https://docs.mongodb.com/manual/core/security-automatic-client-side-encryption/#field-level-encryption-automatic) which is represented in the extended version of JSON Schema. The schema contains the following information

* The algorithm

* The data encryption key

* BSON Type of the field ( required for Deterministic Encrypted Fields).

This schema can be configured at
 
a)**Client Application level**: This is enforced at the MongoDb driver level and automatically encrypts or decrypts the data. Example is [here](https://github.com/mongodb/mongo-java-driver/blob/master/driver-sync/src/examples/tour/ClientSideEncryptionAutoEncryptionSettingsTour.java) and can be seen line 80 onwards.<br/>

b)**Database level**: If the client fails to encrypt the data, the driver will retrieve the schema configured on the server side and use it to perform automatic encryption.<br/>

![\[Diagram for Mongo Client Side Encryption:\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/mongodb-guide/diag_source/mongodb-clientside-encrypt.png?raw=true)

Source: https://docs.mongodb.com/manual/core/security-client-side-encryption/

While automatic field level encryption seems the first and obvious choice given that it is less verbose, do check for following limitations early on to meet your business requirements. It restricts doing the following:<br/>
a) Encrypting individual elements of an array. The entire array field can be encrypted.<br/>
b) Even if you are good with encryption entire array field, it does not allow the 'Deterministic' encryption type. Only 'Randomized is supported.'<br/>
The reference is (here)[https://docs.mongodb.com/manual/reference/security-client-side-automatic-json-schema/].


**Manual Field Encryption**:
The sample code is (here)[the sample code is here. ]

#### Keys:

There are two types of keys that are used in encryption.

1) **Customer Master Key - CMK**

2) **Data Encryption Key - DEK**

Data Encryption key is also called as secret key. The client Application will deal with encryption and decryption of data using a DEK. DEK will be secured using Customer Master Key (CMK). Data Encryption Key is stored in MongoDB vault. It is essentially saved as a collection in Mongo DB. 

CMK in turn will be secured using [KMS](https://docs.mongodb.com/manual/core/security-client-side-encryption-key-management/). Currently MongoDB driver natively supports only AWS KMS(preferred) or local KMS. For any other key manager, the approach would be to make remote service calls and then be treated as local.<br/>
<br/>
For multi-tenant systems, it is recommended to use separate CMK for each tenant. 


#### Overall CS-FLE Challenges:
Overall limitations while using zure Key Vault(AKV) as the key store for client side field level encryption are as follows.<br/>
* Currently MongoDB Atlas natively supports only AWS KMS and local keystore for client-side field level encryption. Ref: here.<br/>
To use Azure Key Vault(AKV) for the stored master encryption key, the encryption key needs to be fetched and proxied through local key store configuration for mongo drivers to understand. Ref: here.<br/>
The local key store requires the key length as 96 bytes only. MongoDB documentation has demonstrated samples using Java SecureRandom() API to generate the keys. 
By default, the algorithm used is SHA1PRNG and the other algorithms supported can be found here. However, all of them are essentially hashing algorithms and not encryption algorithms as AES-128/AES-256 etc. Ref: here<br/>
Even other mentioned methods/tools like 'openssl' or 'dev urandom' used to generate the key for local key stores also use cryptographic randomness. <br/>
TEST_LOCAL_KEY=$(echo "$(head -c 96 /dev/urandom | base64 | tr -d '\n')")<br/>
mongo --nodb --shell --eval "var TEST_LOCAL_KEY='$TEST_LOCAL_KEY'" <br/>
OR <br/>
openssl rand -base64 32 > mongodb-keyfile <br/>

* If AES-128/192/256 is generated and stored as secret in AKV, after retrieving the key, padding is needed in the client application to make it 96 byte length before passing it to the local store.<br/> However this is not needed and a secure random key generation of 96 bytes is recommended.

* Additionally, this generated key needs to be stored as a ‘Secret’ in AKV because the ‘Key’ in AKV only supports asymmetric keys (in RSA or EC format).
Storing the master key as secret will not allow key rotation and other key management policies.
 
* While different fields can be configured using different keys, the same field for different values cannot be configured using different keys. i.e. Only one CMK and not multiple CMKs can be used to encrypt the **same** field values. The reason being the key needs to be associated with the schema itself.
A DEK is created using a CMK and a KeyID is generated.
The KeyId is configured with the fields in the schema. 

###Key Creation Steps
Though Mongodb mention complete steps as (here)[https://docs.mongodb.com/manual/reference/method/KeyVault.createKey/#example], stating here the same with a small twist where assuming a customer managed key exists in Azure key vault, and a data encryption key has to be created. Azure CLI and Mongo shell can be used for the same.

1.  az login --service-principal --username <username> --password <password> --tenant <tenant-id>
 
2. az keyvault secret show --name "<column-masterkey-name>" --vault-name "<key vault name>"
This command will output a value 
 
3. export TEST_LOCAL_KEY= <value from above>
 
4. mongo --nodb --shell --eval "var TEST_LOCAL_KEY='$TEST_LOCAL_KEY'
 
5. var ClientSideFieldLevelEncryptionOptions = {
  "keyVaultNamespace" : "encryption-test.__dataKeys",
  "kmsProviders" : {
    "local" : {
      "key" : BinData(0, TEST_LOCAL_KEY)
    }
  }
}
 
 
6. encryptedClient = Mongo(

  "mongodb+srv://<user>:<pw>@host",    -----> Change to the required connection uri
  ClientSideFieldLevelEncryptionOptions
 )
 
 
7. keyVault = encryptedClient.getKeyVault()
 
8. keyVault.createKey("local", ["data-encryption-key"])
 
 
With above, an encryption-test.__dataKeys", db.collection should get created.



#### Server-Side Encryption
This will be covered in part 2.





