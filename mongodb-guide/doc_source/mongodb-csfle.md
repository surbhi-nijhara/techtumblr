# CSFLE in Mongo DB

### Purpose
The purpose of this document is to understand how data can be encrypted in Mongo database. Encryption can be done both at rest and at field level.
In this blog, we will see what is encryption at field level, ways to encrypt, the steps to create keys and the feature challenges that we encountered.

Let us first brush up the basic pre-requisites to understand the intended feature of encryption.

### Keys:

There are two types of keys used in encryption of data.

1) **Customer Master Key - CMK**

2) **Data Encryption Key - DEK**

Data Encryption key or simply Encryption keys is symmetric key. The client application deals with encryption and decryption of data using a DEK. 
In the MongoDb context, Data Encryption Key is stored in MongoDB vault which is essentially a collection in Mongo DB. 

DEK is secured using Customer Master Key (CMK). 
CMK, in turn, is secured using [KMS](https://docs.mongodb.com/manual/core/security-client-side-encryption-key-management/). 

Currently MongoDB driver natively supports only AWS KMS(preferred) or local KMS. For any other key manager, the approach is be to make KMS provider remote service calls and then be treated as local.<br/>
<br/>

Let us next see what is client side field encryption all about. 

### **Client Side Field Encryption** :

Mongodb supports **Client Side Field Encryption** and it provides the encryption at a field level granularity. <br/>

MongoDb Community and Enterprise edition supports manual field encryption, while automatic field encryption is only supported by Enterprise editions.<br/>
Enterprise edition includes MongoAtlas (Cloud managed AmongoDB Enterprise editions).<br/>

Lets first see <br/>
**Automatic Field Encryption**:
![\[Diagram for Encryption:\]](https://github.com/surbhi-nijhara/techtumblr/blob/field-encryption/mongodb-guide/diag_source/file-auto-encrypt-arch.png?raw=true)

MongoDB Enterprise or MongoAtlas supports Automatic Client-Side Field Level Encryption with help of a daemon [mongocryptd](https://docs.mongodb.com/manual/reference/security-client-side-encryption-appendix/#mongocryptd). <br/>
The daemon should be installed on the host depending on its type viz VM or dockerized enviornment and should become part of IaC configuration.

A Schema is required for [Automatic Field Level Encryption](https://docs.mongodb.com/manual/core/security-automatic-client-side-encryption/#field-level-encryption-automatic) which is represented in the extended version of JSON Schema. The schema contains the following information

* The algorithm
* The data encryption key
* BSON Type of the field ( required for Deterministic Encrypted Fields).

This schema can be configured at
 
a)**Client Application level**: This is enforced at the MongoDb driver level and automatically encrypts or decrypts the data. Example is [here](https://github.com/mongodb/mongo-java-driver/blob/master/driver-sync/src/examples/tour/ClientSideEncryptionAutoEncryptionSettingsTour.java) and can be seen line 80 onwards.<br/>

b)**Database level**: If the client fails to encrypt the data, the driver will retrieve the schema configured on the server side and use it to perform automatic encryption.<br/>

![\[Diagram for Mongo Client Side Encryption:\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/mongodb-guide/diag_source/mongodb-clientside-encrypt.png?raw=true)

[Source](https://docs.mongodb.com/manual/core/security-client-side-encryption/)

While automatic field level encryption seems the first and obvious choice given that it is less verbose, do check for following limitations early on to meet your business requirements. It restricts doing the following:<br/>
a) Encrypting individual elements of an array is not possible at this time. The entire array field needs to be encrypted.<br/>
b) Even if you are good with encryption entire array field, it does not allow the 'Deterministic' encryption type. Only 'Randomized is supported.'<br/>
The reference is [here](https://docs.mongodb.com/manual/reference/security-client-side-automatic-json-schema/)


**Manual Field Encryption**:
Manual Feild Encryption does not include the mongocryptd daemon and hence the explicit encryption, decryption methids need to be called.

![\[Diagram for Encryption:\]](https://github.com/surbhi-nijhara/techtumblr/blob/field-encryption/mongodb-guide/diag_source/field-encrypt-arch.png?raw=true)

The sample code is [here](https://github.com/paychex/mongo-csfl-encryption-java-demo)


### Overall CS-FLE Challenges:
Overall limitations while using zure Key Vault(AKV) as the key store for client side field level encryption are as follows.<br/>
* Currently MongoDB Atlas natively supports only AWS KMS and local keystore for client-side field level encryption. Ref: [here](https://docs.mongodb.com/manual/core/security-client-side-encryption-key-management/).<br/>
To use Azure Key Vault(AKV) for the stored master encryption key, the encryption key needs to be fetched and proxied through local key store configuration for mongo drivers to understand. Ref:[here](https://www.mongodb.com/blog/post/clientside-field-level-encryption-faq--webinar).<br/>

* The local key store requires the key length as 96 bytes only. MongoDB documentation has demonstrated samples using Java SecureRandom() API to generate the keys. 
By default, the algorithm used is SHA1PRNG and the other algorithms supported can be found [here](https://docs.oracle.com/javase/8/docs/technotes/guides/security/StandardNames.html#SecureRandom). However, all of them are essentially hashing algorithms and not encryption algorithms as AES-128/AES-256 etc. Ref: [here]()<br/>

There are other methods/tools like 'openssl' or 'dev urandom'  to generate the key for local key stores using cryptographic randomness. <br/>
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

### Key Creation Steps
Though Mongodb mentioned steps are [here](https://docs.mongodb.com/manual/reference/method/KeyVault.createKey/#example), stating the end-end steps where assuming a customer managed key exists in Azure key vault, and a data encryption key has to be created. Azure CLI and Mongo shell can be used for the same.

1.  az login --service-principal --username <username> --password <password> --tenant <tenant-id><br/>
2. az keyvault secret show --name "<column-masterkey-name>" --vault-name "<key vault name>"<br/>
This command will output a value.<br/>
3. export TEST_LOCAL_KEY= <value from above><br/>
4. mongo --nodb --shell --eval "var TEST_LOCAL_KEY='$TEST_LOCAL_KEY'<br/>
5. var ClientSideFieldLevelEncryptionOptions = {
  "keyVaultNamespace" : "encryption-test.__dataKeys",
  "kmsProviders" : {
    "local" : {
      "key" : BinData(0, TEST_LOCAL_KEY)
    }
  }
}<br/>
6. encryptedClient = Mongo(
  "mongodb+srv://<user>:<pw>@host",    -----> Change to the required connection uri
  ClientSideFieldLevelEncryptionOptions
 )<br/>
7. keyVault = encryptedClient.getKeyVault()<br/>
8. keyVault.createKey("local", ["data-encryption-key"])<br/>
With above, an encryption-test.__dataKeys", db.collection should get created.

**Note:**<br/>
If MongoAtlas is used, You won't be able to run --nodb --eval option in Mongo shell connection for Atlas since there is no access to local terminal or bash shell. Execute eval command in Atlas is unsupported in Atlas. Refer to d[ocumentation](https://docs.atlas.mongodb.com/reference/unsupported-commands-paid-tier-clusters/index.html).
We need to use the compatible MongoDB driver to enable the client side application.Rrefer to the [documentation](https://docs.mongodb.com/manual/core/security-explicit-client-side-encryption/#explicit-manual-client-side-field-level-encryption). Also, kindly be aware of the known [limitations](https://docs.mongodb.com/manual/reference/security-client-side-encryption-limitations/).





