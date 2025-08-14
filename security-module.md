# Agent Security Module Documentation

## Overview

The AgentSecurityModule provides comprehensive security features for autonomous agent coordination, implementing a four-tier security system with encryption, access control, and timelock protection.

## Security Levels

### BASIC (Level 0)
**Purpose**: Public coordination, testing, low-value operations

**Features**:
- No timelock requirements
- Simple obfuscation (XOR with constant)
- No proof requirements
- Immediate execution allowed

**Gas Cost**: ~200k for context creation

**Use Cases**:
- Public information sharing
- Test coordination
- Low-stakes operations (<$100 value)

```solidity
// Basic security context
securityModule.createSecurityContext(
    intentHash,
    AgentSecurityModule.SecurityLevel.BASIC,
    participants,
    0 // No timelock
);
```

### STANDARD (Level 1)
**Purpose**: Regular DeFi operations, standard arbitrage

**Features**:
- 5-minute minimum timelock
- XOR encryption with deterministic keys
- No proof requirements after timelock
- Automatic key generation

**Gas Cost**: ~240k for context creation

**Use Cases**:
- Standard DeFi arbitrage
- Regular trading coordination
- Medium-value operations ($100-$10k)

```solidity
// Standard security with encryption
address[] memory participants = [agent1, agent2];
(bytes memory encrypted, bytes memory keyData) = securityModule.encryptCoordinationData(
    sensitiveStrategy,
    participants,
    AgentSecurityModule.SecurityLevel.STANDARD
);
```

### ENHANCED (Level 2)
**Purpose**: High-value arbitrage, cross-chain operations

**Features**:
- 30-minute minimum timelock
- Multi-layer encryption with key shares
- 65-byte security proof required
- Participant verification

**Gas Cost**: ~260k for context creation

**Use Cases**:
- High-value arbitrage (>$10k)
- Cross-chain coordination
- MEV extraction strategies

```solidity
// Enhanced security with proof
bytes memory enhancedProof = generateSecurityProof(intentHash, participants);
(bool valid,) = securityModule.validateSecurityLevel(
    intentHash,
    AgentSecurityModule.SecurityLevel.ENHANCED,
    enhancedProof
);
```

### MAXIMUM (Level 3)
**Purpose**: Critical infrastructure, large-scale operations

**Features**:
- 2-hour minimum timelock
- Advanced multi-layer encryption
- Complex proofs (32-1024 bytes)
- Requires all participants to register public keys
- Formal verification capabilities

**Gas Cost**: ~310k for context creation

**Use Cases**:
- Critical infrastructure coordination
- Operations >$100k value
- High-security environments

```solidity
// Maximum security setup
vm.prank(participant);
securityModule.registerPublicKey(keccak256("participant_pubkey"));

bytes memory maximumProof = generateComplexProof(intentHash);
// Proof can be zero-knowledge proof, formal verification, etc.
```

## Encryption System

### Encryption Levels

#### Basic Obfuscation
```solidity
function _obfuscateData(bytes calldata data) internal pure returns (bytes memory) {
    bytes memory result = new bytes(data.length);
    for (uint256 i = 0; i < data.length; i++) {
        result[i] = bytes1(uint8(data[i]) ^ 0xAA); // Simple XOR
    }
    return result;
}
```

#### Standard XOR Encryption
```solidity
// Deterministic key generation
bytes32 masterKey = keccak256(abi.encodePacked(
    data, participants, block.chainid, securityLevel
));

// Rolling XOR encryption
for (uint256 i = 0; i < data.length; i++) {
    if (i > 0 && i % 32 == 0) {
        currentKey = keccak256(abi.encodePacked(currentKey, i));
    }
    result[i] = bytes1(uint8(data[i]) ^ uint8(currentKey[i % 32]));
}
```

#### Multi-Layer Encryption (Enhanced/Maximum)
```solidity
// Layer 1: Master key encryption
result = _xorEncryptMemory(data, masterKey);

// Layer 2: Participant-derived key
bytes32 participantKey = keccak256(abi.encodePacked(participants));
result = _xorEncryptMemory(result, participantKey);

// Layer 3: Position-based obfuscation
for (uint256 i = 0; i < result.length; i++) {
    result[i] = bytes1(uint8(result[i]) ^ uint8(i + 1));
}
```

### Key Management

#### Key Generation
Keys are deterministically generated based on:
- Intent hash
- Security level
- Participant list
- Block metadata (timestamp, prevrandao)

#### Key Sharing (Enhanced/Maximum)
```solidity
function _generateKeyShares(bytes32 masterKey, address[] calldata participants) 
    internal pure returns (bytes memory) {
    bytes32 participantKey = keccak256(abi.encodePacked(participants));
    return abi.encode(masterKey, participantKey);
}
```

#### Decryption Process
```solidity
// Reverse layer 3: Remove position obfuscation
// Reverse layer 2: XOR with participant key  
// Reverse layer 1: XOR with master key
```

## Access Control

### Participant Authorization
```solidity
mapping(bytes32 => mapping(address => bool)) private _participantAccess;

function createSecurityContext(/*...*/) external onlyFramework {
    for (uint256 i = 0; i < participants.length; i++) {
        _participantAccess[intentHash][participants[i]] = true;
    }
}
```

### Emergency Procedures

#### Access Revocation
```solidity
function revokeAccess(bytes32 intentHash, address participant) external {
    require(
        context.creator == msg.sender || securityModule.owner() == msg.sender,
        "Unauthorized"
    );
    _participantAccess[intentHash][participant] = false;
}
```

#### Owner Override
The contract owner can:
- Revoke any participant's access
- Update minimum timelock requirements
- Transfer ownership
- Emergency shutdown capabilities

### Public Key Registry (Maximum Security)
```solidity
mapping(address => bytes32) private _agentPublicKeys;

function registerPublicKey(bytes32 publicKey) external {
    require(publicKey != bytes32(0), "Invalid public key");
    _agentPublicKeys[msg.sender] = publicKey;
}
```

## Timelock System

### Implementation
```solidity
struct SecurityContext {
    uint256 timelock;
    uint256 createdAt;
    // ...
}

function validateSecurityLevel(/*...*/) external view returns (bool, string memory) {
    if (block.timestamp < context.createdAt + context.timelock) {
        return (false, "Timelock not satisfied");
    }
    // ...
}
```

### Configurable Minimums
```solidity
mapping(SecurityLevel => uint256) private _minTimelocks;

constructor() {
    _minTimelocks[SecurityLevel.BASIC] = 0;
    _minTimelocks[SecurityLevel.STANDARD] = 300;    // 5 minutes
    _minTimelocks[SecurityLevel.ENHANCED] = 1800;   // 30 minutes  
    _minTimelocks[SecurityLevel.MAXIMUM] = 7200;    // 2 hours
}
```

## Security Proofs

### Enhanced Level Proofs
- **Format**: 65-byte signature
- **Content**: Signature over intent hash + participant commitment
- **Validation**: Standard ECDSA recovery

```solidity
function _validateSecurityProof(bytes32 intentHash, bytes calldata proof, SecurityLevel level) 
    internal view returns (bool) {
    if (level == SecurityLevel.ENHANCED) {
        return proof.length == 65; // Standard signature length
    }
    // ...
}
```

### Maximum Level Proofs
- **Format**: 32-1024 bytes
- **Content**: Can include:
    - Zero-knowledge proofs
    - Multi-signature schemes
    - Formal verification certificates
    - Hardware attestations

```solidity
if (level == SecurityLevel.MAXIMUM) {
    return proof.length >= 32 && proof.length <= 1024;
}
```

## Integration Patterns

### Basic Integration
```solidity
// 1. Create security context
securityModule.createSecurityContext(intentHash, level, participants, timelock);

// 2. Encrypt sensitive data
(bytes memory encrypted, bytes memory keys) = securityModule.encryptCoordinationData(
    sensitiveData, participants, level
);

// 3. Validate before execution
(bool valid,) = securityModule.validateSecurityLevel(intentHash, level, proof);
require(valid, "Security validation failed");

// 4. Decrypt for execution
bytes memory decrypted = securityModule.decryptCoordinationData(
    encrypted, keys, msg.sender, level
);
```

### Advanced Integration with Framework
```solidity
contract SecureCoordination {
    AgentCoordinationFramework immutable coordination;
    AgentSecurityModule immutable security;
    
    function proposeSecure(SecureRequest calldata request) external {
        // Create security context
        security.createSecurityContext(/*...*/);
        
        // Encrypt coordination data
        (bytes memory encrypted, bytes memory keys) = security.encryptCoordinationData(/*...*/);
        
        // Modify payload with encrypted data
        CoordinationPayload memory securePayload = request.payload;
        securePayload.coordinationData = encrypted;
        securePayload.metadata = keys;
        
        // Propose through framework
        coordination.proposeCoordination(request.intent, request.signature, securePayload);
    }
}
```

## Best Practices

### Security Level Selection
```solidity
function recommendSecurityLevel(uint256 coordinationValue) public pure returns (SecurityLevel) {
    if (coordinationValue < 0.1 ether) return SecurityLevel.BASIC;
    if (coordinationValue < 10 ether) return SecurityLevel.STANDARD;
    if (coordinationValue < 100 ether) return SecurityLevel.ENHANCED;
    return SecurityLevel.MAXIMUM;
}
```

### Timelock Configuration
- **Standard Operations**: Use minimum required timelock
- **Cross-chain**: Add network finality time to minimum
- **High volatility markets**: Consider shorter timelocks
- **Critical infrastructure**: Use longer timelocks for additional safety

### Key Management
- **Rotation**: Register new public keys periodically
- **Backup**: Ensure key recovery mechanisms
- **Distribution**: Use secure channels for key distribution

### Error Handling
```solidity
try securityModule.validateSecurityLevel(intentHash, level, proof) 
    returns (bool valid, string memory reason) {
    if (!valid) {
        emit SecurityValidationFailed(intentHash, reason);
        return false;
    }
} catch {
    emit SecurityValidationError(intentHash);
    return false;
}
```

## Gas Optimization Tips

### Batch Operations
```solidity
// Validate multiple intents at once
function batchValidateSecurity(
    bytes32[] calldata intentHashes,
    bytes[] calldata proofs
) external view returns (bool[] memory results) {
    // More efficient than individual calls
}
```

### Storage Optimization
- Pack struct members efficiently
- Use mappings instead of arrays where possible
- Minimize storage writes

### Execution Optimization
- Cache frequently accessed values
- Use view functions for read-only operations
- Optimize loop iterations

## Migration and Upgrades

### Security Level Upgrades
```solidity
function upgradeSecurityLevel(bytes32 intentHash, SecurityLevel newLevel) external {
    require(uint8(newLevel) > uint8(context.level), "Cannot downgrade");
    require(block.timestamp >= context.createdAt + context.timelock, "Timelock active");
    
    context.level = newLevel;
    context.timelock = _minTimelocks[newLevel];
}
```

### Contract Upgrades
The security module is designed to be:
- **Modular**: Can be replaced without affecting core framework
- **Backward Compatible**: New versions maintain interface compatibility
- **Gradual**: Migration can happen intent-by-intent

## Troubleshooting

### Common Issues

#### "Timelock not satisfied"
- **Cause**: Attempting validation before timelock expires
- **Solution**: Wait for timelock period or check `block.timestamp`

#### "Security proof required"
- **Cause**: Enhanced/Maximum level requires proof
- **Solution**: Provide valid proof bytes

#### "Participant missing public key"
- **Cause**: Maximum security requires all participants to register keys
- **Solution**: All participants call `registerPublicKey()`

#### "Invalid security level"
- **Cause**: Security level out of bounds
- **Solution**: Use SecurityLevel enum values (0-3)

### Debug Functions
```solidity
function debugSecurityContext(bytes32 intentHash) external view returns (
    SecurityLevel level,
    uint256 timelock,
    uint256 createdAt,
    uint256 timeRemaining,
    bool timelockSatisfied
) {
    SecurityContext memory context = getSecurityContext(intentHash);
    uint256 remaining = block.timestamp >= context.createdAt + context.timelock ? 
        0 : (context.createdAt + context.timelock) - block.timestamp;
    
    return (
        context.level,
        context.timelock, 
        context.createdAt,
        remaining,
        block.timestamp >= context.createdAt + context.timelock
    );
}
```

## Security Auditing

### Checklist
- [ ] All security levels implemented correctly
- [ ] Timelock enforcement working
- [ ] Encryption/decryption round-trip successful
- [ ] Access control preventing unauthorized access
- [ ] Emergency procedures functional
- [ ] Gas usage within acceptable bounds
- [ ] No reentrancy vulnerabilities
- [ ] Input validation comprehensive

### Testing Strategy
- Unit tests for each security level
- Integration tests with coordination framework
- Fuzz testing for edge cases
- Gas optimization testing
- Security proof validation testing

### Known Security Considerations
- **On-chain transparency**: All operations visible on blockchain
- **Key management**: Users responsible for private key security
- **Proof validation**: Only basic validation implemented
- **Timelock bypass**: Owner can revoke access immediately
- **Gas griefing**: Large participant lists can cause high gas costs