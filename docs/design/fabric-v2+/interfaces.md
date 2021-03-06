<!---
Licensed under Creative Commons Attribution 4.0 International License
https://creativecommons.org/licenses/by/4.0/
--->

# FPC Component Interfaces and States

This document defines the interfaces exposed by the FPC components and their internal state.


# ERCC

## Interface:
The ERCC interface is implemented through a "normal" chaincode interface.

```go
// chaincode interface (exposed to ecc and clients)
func Invoke(stub shim.ChaincodeStubInterface) pb.Response {}

// functions below are dispatched by Invoke implementation
// Note: Maybe use protobuf-to-json mapping
// (https://developers.google.com/protocol-buffers/docs/proto3#json)
// to map the methods and their arguments into to chaincode
// invocation form { "Function": "..", "Args": [....]}

// returns a set of credentials registered for a given chaincode id
// Note: to get the endpoints of FPC endorsing peers do the following:
// - discover all endorsing peers (and their endpoints) for the FPC chaincode using "normal" lifecycle
// - query `getEnclaveId` at all the peers discovered
// - query queryCCInfo with all received enclave_ids
// this gives you the endpoints and credentials including enclave_vk, and chaincode_ek
func queryListEnclaveCredentials(chaincode_id string) (allCredentials []Credentials) {}
func queryEnclaveCredentials(chaincode_id string, enclave_id string) (credentials Credentials) {}

// Optional Post-MVP;
// returns a list of all provisioned enclaves for a given chaincode id. A provisioned enclave is a registered enclave that has also the chaincode decryption key.
func queryListProvisionedEnclaves(chaincode_id string) (enclave_ids []string)

// returns the chaincode encryption key for a given chaincode id
func queryChaincodeEncryptionKey(chaincode_id string) (chaincode_ek []byte) {}

// register a new FPC chaincode enclave instance
func registerEnclave(credentials Credentials) error {}

// registers a CCKeyRegistration message that confirms that an enclave is provisioned with the chaincode encryption key. This method is used during the key generation and key distribution protocol. In particular, during key generation, this call sets the chaincode_ek for a chaincode if no chaincode_ek is set yet.
func registerCCKeys(msg CCKeyRegistrationMessage) error {}

// key distribution (Post-MVP features)
func putKeyExport(msg ExportMessage) error {}
func getKeyExport(chaincode_id string, enclave_id string) (ExportMessage, error) {}
```

## State:

The ERCC state is entirely stored on the ledger state using `putState` operations.

Ledger state is abstracted as a simple map:
```go
map[string][]byte
```

To store data, we use the following key scheme, which is inspired by [_lifecycle](https://github.com/hyperledger/fabric/blob/master/core/chaincode/lifecycle/lifecycle.go#L81). A variable is denoted using <variable_name> annotation. Note that this scheme is defined here as, both, ERCC and TLCC need to parse/retrieve this information.

The `enclave_id` a hex-encoded string of SHA256 hash over `enclave_vk`. 


```go
// stores the chaincode encryption key
namespaces/chaincode_ek/<chaincode_id> -> chaincode_ek

// stores the credentials(see definition below in ecc) for a given chaincode enclave
namespaces/credentials/<chaincode_id>/<enclave_id> -> Credentials

// stores key registration messages for registered enclaves which are provisioned with the chaincode encryption key
namespaces/provisioned/<chaincode_id>/<enclave_id> -> SignedCCKeyRegistrationMessage

// stores export messages. set with exportCCKeys and retrieved using importCCKeys
namespaces/exported/<chaincode_id>/<enclave_id> -> SignedExportMessage
```

This key scheme is design with the goal in mind to reduce the write conflicts for concurrent enclave registrations.
ERCC state can accessed and modified by the [lifecycle ledger shim](https://github.com/hyperledger/fabric/blob/master/core/chaincode/lifecycle/ledger_shim.go) or the normal [go-chaincode shim](https://github.com/hyperledger/fabric/blob/master/vendor/github.com/hyperledger/fabric-chaincode-go/shim/stub.go). Here an example:
```go
// returns the chaincode encryption key for AuctionChaincode1
k := fmt.Sprintf("namespaces/chaincode_ek/%s", "AuctionChaincode1")
chaincode_ek, err := getState(k)

prefix := fmt.Sprintf("namespaces/credentials/%s/", "AuctionChaincode1")
// allCredentials is a map[string][]byte containing the credentials for all enclaves associated with AuctionChaincode1
allCredentials, err := getStateRange(prefix)

// returns the credentials for a specific chaincode enclave
k := fmt.Sprintf("namespaces/credentials/%s/%s", "AuctionChaincode1", "SomeEnclaveId")
credentials, err := getState(k)
```

Data types are defined as follows:

```protobuf
syntax = "proto3"

// TODO update UML with this channel hash and tlcc_mrenclave and check at ercc that this hash corresponds to peers view of the channel id 

message CC_Parameters {
        string chaincode_id   = 1;      // chaincode name
        string version        = 2;      // chaincode version, this is the expected mrenclave
        string sequence       = 3;      // chaincode sequence number
        string channel_id     = 4;      // channel id        
}

message Host_Parameters {
        bytes peer_identity = 1;        // peer that hosts the enclave
}

message AttestedData {
        CC_Parameters cc_params    = 1;
        Host_Parameter host_params = 2;
        bytes enclave_vk           = 3; // chaincode enclave public key
        bytes channel_hash         = 4; // cryptographic hash of the channel genesis block;  
        string tlcc_mrenclave      = 5; // expected TLCC mrenclave  
}

message Credentials {
        AttestedData data    = 1;
        bytes evidence_bytes = 2;       // serialized attestation evidence
}

message CCKeyRegistrationMessage {
        bytes cc_params_hash = 1;       // defines the context as SHA2 of cc_params
        bytes chaincode_ek   = 2;       // public chaincode encryption key
        bytes enclave_id     = 3;       // creator of this message
}

message SignedCCKeyRegistrationMessage {
        bytes msg_bytes = 1;            // public chaincode encryption key
        bytes signature = 2;            // signature of the message creator (enclave)
}

message ExportMessage {
        bytes cc_params_hash = 1;       // defines the context as SHA2 of cc_params
        bytes chaincode_ek   = 2;       // public chaincode encryption key
        bytes cckeys_enc     = 3;       // chaincode keys encrypted for the receiver
        bytes enclave2_vk    = 4;       // receiver of this message
        bytes enclave_vk     = 5;       // creator of this message
}

message SignedExportMessage {
        bytes export_msg_bytes = 1;
        bytes signature        = 2;     // signature of the message creator
}

message AttestationEvidence {
        // this message is defined with more detail in #412
        // https://github.com/hyperledger-labs/fabric-private-chaincode/issues/412
        enum Type {
                SGX_EPID = 0;
                SGX_DCAP = 1;
                WHATEVER = 2;
        }
        Type type     = 1;
        bytes payload = 2;
}

```

ERCC keeps in instance of an attestation.Verifier to check an attestation evidence message. ERCC just passes the serialized attestation evidence message to the verifier. 
Depending on the attestation protocol (e.g., EPID- or DCAP-based attestation), the verifier implements the corresponding logic. Details of the evidence verification are defined in [#412](https://github.com/hyperledger-labs/fabric-private-chaincode/issues/412).

```go
type EnclaveRegistryCC struct {
	ra  attestation.Verifier
}
```


# TLCC

Some notes on TLCC. The TLCC instance is currently implemented as a system chaincode, that is, there exists only a single instance. As a peer can participate in many channels, there is a separate TLCC_enclave per each channel. The TLCC chaincode is responsible to multiplex requests for a given channel to the corresponding TLCC_enclave. Note for MVP, we only support a single channel but the interface specified here should already take multi-channel support into account.

## Interface:
The TLCC interface is implemented through a "normal" chaincode interface.

```go
// chaincode interface
func Invoke(stub shim.ChaincodeStubInterface) pb.Response {}

// functions below are dispatched by Invoke implementation

// used by the admin to join the channel
// Note: this methods creates a new tlcc_enclave and starts a block listener
// for the given channel_id that passes the genesis block and all
// subsequent blocks to the tlcc_enclave.
func joinChannel(channel_id string) error {}

// returns the SHA256 hash of the channel genesis block
// the tlcc enclave is initialized with
func getChannelHash(channel_id string) (string, error) {}
```

```c++
// Provided/implemented by a common logging module
// interface exposed to TLCC_enclave via cgo
void ocall_print_string([in, string] const char *str);
```

```go
// interface to establish/close a secure channel between an
// ECC_Enclave and a TLCC_enclave.
func session_setup(setup SessionSetup) (SessionSetupResponse, error) {}
func session_close(close SessionClose) (SessionCloseResponse, error) {}

// interface to invoke a TLCC request, such as, getMetadata and validate_identity
func session_request(request SessionRequest) (SessionRequestResponse, error) {}
```


```protobuf
syntax = "proto3"

// Setup and Close messages are used to establish a secure channel between
// ECC_Enclave and TLCC_Enclave. Once a secure channel is established,
// Session Requests to TLCC can be issued to invoke TLCC functions such
// as getMetadata or verifyMSPId.

// TODO How to integrate the the message authenticator?!
// should we abstract it as signature here even though we may
// use CMAC once a session is established?

// TODO: update UML and make sure that session
// establishment is done when ecc is created. In particular,
// this must happen before attestation. WHy is this? Because this is
// the way how ECC learns the channel hash and "binds" to it through attestation

message SessionSetup {
        string channel_id = 1;
        bytes enclave_vk  = 2;
}

message SessionSetupResponse {
        SessionSetup setup   = 1;
        bytes tlcc_mrenclave = 2;
        bytes channel_hash   = 3;
        bytes context        = 4;        // context that references the session
}

message SessionClose {
        bytes context = 1;
}

message SessionCloseResponse {
}

// TODO add signature or some other for of message
// authentication to session request/response.

message SessionRequest {
        bytes context    = 1;
        bytes tx_context = 2;
        bytes nonce      = 3;
        oneof request {
                GetMetadataRequest metadata               = 1; 
                ValidateIdentityRequest validate_identity = 2;
        }
}

message SessionRequestResponse {
        SessionRequest request = 1;
        oneof response {
                GetMetadataResponse metadata               = 1; 
                ValidateIdentityResponse validate_identity = 2; 
        }
}

```

## State:

TLCC keeps a local reference for each channel and its tlcc_enclave. This
reference is volatile. Also a reference to the peer is kept to access the
ledger and read blocks.

```go
type TrustedLedgerCC struct {
        channelMapping map[string]enclave.Stub
        peer           *peer.Peer              
}

type StubImpl struct {
        eid C.enclave_id_t
}
```

# TLCC_Enclave

## Interface:
Defined in the TLCC EDL.
```c++
// creates a new channel
public int ecall_join_channel(
        [in, size=gen_len] uint8_t *genesis, uint32_t gen_len);

// pushes next block to tlcc
public int ecall_next_block(
        [user_check] uint8_t *block_bytes, uint32_t block_size);
```

```c++
// this secure session functions are defined in a secure channel module
//included from secure_channel.edl

public int session_setup(
        [in] setup SessionSetup,
        [out] SessionSetupResponse);

public int session_close(
        [in] close SessionClose
        [out] SessionCloseResponse);

public int session_request(
        [in] request SessionRequest,
        [out] SessionRequestResponse;
```


Session requests are dispatched to the following interfaces:
```c++
// returns a cryptographic hash of the genesis block this tlcc enclave is initialized with.
public hash get_channel_hash();

// TODO update UMLs: these functions replace "check_ercc_commited_data"

// verify state data
public metadata get_state_metadata(
        const char *namespace,
        const char *key);

public metadata get_multi_state_metadata(
        const char *namespace,
        const char *comp_key);

// verify that a given identity is part of a msp
// the input is a serialized identity proto message as defined in
// https://github.com/hyperledger/fabric-protos/blob/master/msp/identities.proto#L15
public bool validate_identity(
        const uint8_t *serializedIdentity,
        const uint32_t len);

// checks if a given enclave identifier can endorse transactions
// as defined in the chaincode definition; this checks that the given enclave
// has correct the MRENCLAVE and enclave is part of an organization that can
// satisfy the endorsing policy of a given chaincode.
public bool can_endorse(
        const char *chaincode_id,       
        const char *enclave_id);
```

## State:
The internal TLCC state consists of the current ledger state (block number, integrity-metadata, and msp) and sessions information for ecc communication.

More details on the TLCC state will also be output of [#402](https://github.com/hyperledger-labs/fabric-private-chaincode/issues/402). Therefore, the state representation here is not strict about actually data types, but should illustrate what state is maintained.

```c++
// TODO: this needs some more brain cycles with respect of #402 and #410
typedef struct internal_tlcc_state_t {
        uint32_t sequence_number, // block number counter
        string channel_id,
        uint8_t genesis_hash, // cryptographic identifier of the channel

        // integrity metadata
        kvs_t metadata_state,
        
        // keep ercc and lifecycle separately as TLCC provides
        // additionally, such as `can_endorse`, requiring the information
        kvs_t ercc,
        kvs_t lifecycle,

        // msp 
        X509_STORE* root_certs_orderer, // root cert store for orderer org
        X509_STORE* root_certs_apps, // root cert store for application orgs

        // tlcc session
        session_keys_t session_keys
};
```

# ECC

## Interface:
The ECC interface is implemented through a "normal" chaincode interface. Methods with leading underscores
are treated as FPC commands. Normal `invoke` invocations are forwarded to a FPC chaincode enclave.

```go
// chaincode interface (exposed to admin/clients) implemented by invoke
func Invoke(stub shim.ChaincodeStubInterface) pb.Response {}

// functions below are dispatched by Invoke implementation
// some of these functions below may require access to the
// stub shim.ChaincodeStubInterface 

// triggered by an admin
func initEnclave() (Credentials, error) {}

// key generation
func generateCCKeys() (SignedCCKeyRegistrationMessage, error) {}

// key distribution (Post-MVP Feature)
func exportCCKeys(credentials Credentials) (SignedExportMessage, error) {}
func importCCKeys() (SignedCCKeyRegistrationMessage, error) {}

// returns the EnclaveId hosted by the peer
func getEnclaveId() (string, error) {}

// chaincode invoke
func chaincodeInvoke(transactionProposal []byte, ctx interface{}) (response, error) {}
```

This interface is implemented by ECC to let a chaincode enclave call into the peer

```c++
// Provided/implemented by a common logging module
// interface exposed to TLCC_enclave via cgo
void ocall_print_string([in, string] const char *str);
```

```c++
// state access
// these get_state calls are bound to the chaincode namespace
void ocall_get_state(
        [in, string] const char *key, 
        [out, size=max_val_len] uint8_t *val, uint32_t max_val_len, [out] uint32_t *val_len,
        [user_check] void *u_shim_ctx);

void ocall_put_state(
        [in, string] const char *key,
        [in, size=val_len] uint8_t *val, uint32_t val_len,
        [user_check] void *u_shim_ctx);

void ocall_get_state_by_partial_composite_key(
        [in, string] const char *comp_key,
        [out, size=max_len] uint8_t *values, uint32_t max_len, [out] uint32_t *values_len,
        [user_check] void *u_shim_ctx); 
```

```c++
// this secure session functions are defined in a secure channel module
//included from secure_channel.edl

public int session_setup(
        [in] setup SessionSetup,
        [out] SessionSetupResponse);

public int session_close(
        [in] close SessionClose
        [out] SessionCloseResponse);

// used, for instance, to retrieve integrity metadata for get_state operations
public int session_request(
        [in] request SessionRequest,
        [out] SessionRequestResponse;
```




## State:
An ECC instance keeps a local reference of the FPC chaincode enclave id.

ECC also keeps a reference to an EvidenceService is part of the attestation module as defined
[#412](https://github.com/hyperledger-labs/fabric-private-chaincode/issues/412).

ECC instance also keeps a reference to a sealed storage module as defined in [#421](https://github.com/hyperledger-labs/fabric-private-chaincode/issues/421).

```go
type EnclaveChaincode struct {
        erccStub ercc.EnclaveRegistryStub
        tlccStub tlcc.TLCCStub
        enclave  enclave.Stub
        verifier crypto.Verifier
        ev       attestation.EvidenceService // abstracts evidence creation for EPID/DCAP
        storage  sealedStorageModule         // more details in #421
}

type StubImpl struct {
	eid C.enclave_id_t
}
```


# ECC_Enclave

## Interface:
The ECC_Enclave interface specifies interface of an FPC chaincode enclave. 

Note: `sealed credentials` is renamed to `sealed state`
TODO: Update UML diagrams

```c++
// initializes an enclave with chaincode and host parameters
public int ecall_init(
        [in, size=cc_params_len] uint8_t *cc_params, uint32_t cc_params_len,
        [in, size=host_params_len] uint8_t *host_params, uint32_t host_params_len,
        [out] uint8_t *credentials);     // TBD is this a fixed length type?

// invoke a FPC chaincode
public int ecall_chaincode_invoke(
        [in, size=proposal_len]  uint8_t *proposal, uint32_t proposal_len,
        [out, size=response_len_in] uint8_t *response, uint32_t response_len_in,
        [out] uint32_t *response_len_out,
        [out] sgx_ec256_signature_t *signature,
        [user_check] void *u_shim_ctx);

// key generation and distribution
public int ecall_generate_cc_keys(
        [out] uint8_t *cckey_registration_msg);

public int ecall_export_cc_keys(
        [in] uint8_t *target_enclave_vk,
        [out] uint8_t *export_msg);

public int ecall_import_cc_keys(
        [in] uint8_t *export_msg,
        [out] uint8_t *cckey_registration_msg);
```

```c++
// Binding interface
// note this is Post-MVP feature, but exists here for completeness, TBD details
public int ecall_get_CSR(
        [in] args   
        [out] csr);
```

## State:
We use the `fsm` to control the state of ecc, for instance, to signal
when an ecc enclave is ready to process transaction invocations or additional
ecc state, such as key provision must be completed. More details on the provisioning using the sealed storage module in [#421](https://github.com/hyperledger-labs/fabric-private-chaincode/issues/421).

```c++
// this is create on enclave creation and kept inside the enclave
typedef struct internal_ecc_state_t {
        cc_params_t cc_params              // chaincode parameter
        host_params_t host_params          // host parameters

        ecc_fsm_t fsm,                     // ecc finite-state-machine state
                                           // TBD: define states such as CREATED, ..., READY)
                                           // Proposed by Michael:
                                           // no-keys, enclave-keys, cc-keys

        // enclave-specific keys
        sgx_ec256_public_t enclave_vk,     // signature verification key
        sgx_ec256_private_t enclave_sk,    // signature key
         
        // chaincode-specific keys
        sgx_aes_gcm_128bit_key_t sek,      // ledger state encryption key
        sgx_ec256_public_t chaincode_ek,   // argument encryption key
        sgx_ec256_private_t chaincode_dk,  // argument decryption key

        // tlcc session
        sgx_cmac_128bit_key_t session_key; // is set during tlcc-ecc binding
};
```

# FPC_SHIM

This interface is exposed to a FPC chaincode. The chaincode must implement `invoke` and can access the ledger state using the corresponding access methods.

```c++
// must be implemented by a FPC chaincode
int invoke(uint8_t* response,
    uint32_t max_response_len,
    uint32_t* actual_response_len,
    shim_ctx_ptr_t ctx);

// exposed to FPC chaincode
void put_state(const char* key, uint8_t* val, uint32_t val_len, shim_ctx_ptr_t ctx);

void get_state(
    const char* key, uint8_t* val, uint32_t max_val_len, uint32_t* val_len, shim_ctx_ptr_t ctx);

void get_state_by_partial_composite_key(
    const char* comp_key, std::map<std::string, std::string>& values, shim_ctx_ptr_t ctx);

void put_public_state(const char* key, uint8_t* val, uint32_t val_len, shim_ctx_ptr_t ctx);

void get_public_state(
    const char* key, uint8_t* val, uint32_t max_val_len, uint32_t* val_len, shim_ctx_ptr_t ctx);

void get_public_state_by_partial_composite_key(
    const char* comp_key, std::map<std::string, std::string>& values, shim_ctx_ptr_t ctx);

int get_string_args(std::vector<std::string>& argss, shim_ctx_ptr_t ctx);

int get_func_and_params(
    std::string& func_name, std::vector<std::string>& params, shim_ctx_ptr_t ctx);

void get_channel_id(char* channel_id,
    uint32_t max_channel_id_len,
    shim_ctx_ptr_t ctx);

void get_msp_id(char* msp_id,
    uint32_t max_msp_id_len,
    shim_ctx_ptr_t ctx);

void get_creator_name(char* msp_id,  // MSP id of organization to which transaction creator belongs
    uint32_t max_msp_id_len,         // size of allocated buffer for msp_id
    char* dn,                        // distinguished name of transaction creator
    uint32_t max_dn_len,             // size of allocated buffer for dn
    shim_ctx_ptr_t ctx);

extern int get_random_bytes(uint8_t* buffer, size_t length);

void log_critical(const char* format, ...);
void log_error(const char* format, ...);
void log_warning(const char* format, ...);
void log_notice(const char* format, ...);
void log_info(const char* format, ...);
void log_debug(const char* format, ...);
```
See https://github.com/hyperledger-labs/fabric-private-chaincode/blob/master/ecc_enclave/enclave/shim.h
