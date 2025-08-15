# EIP-8001: Agent Coordination Framework

[![License: CC0-1.0](https://img.shields.io/badge/License-CC0%201.0-lightgrey.svg)](http://creativecommons.org/publicdomain/zero/1.0/)
[![Solidity](https://img.shields.io/badge/Solidity-^0.8.20-blue)](https://docs.soliditylang.org/)
[![Foundry](https://img.shields.io/badge/Built%20with-Foundry-000000.svg)](https://getfoundry.sh/)

> **Secure framework for autonomous agent coordination with modular security and cross-chain capabilities**

## Overview

The **Agent Coordination Framework** implements [EIP-8001](https://eips.ethereum.org/EIPS/eip-8001), providing a standardized, secure infrastructure for autonomous agents to coordinate complex operations in DeFi/GameFi, MEV extraction, cross-chain arbitrage, in-game NPCs and automated market making.

### Key Features

-  **Multi-Level Security**: Four-tier security system (BASIC → STANDARD → ENHANCED → MAXIMUM)
-  **Cryptographic Guarantees**: EIP-712 compliant signatures with replay protection
-  **Cross-Chain Ready**: Built for multi-chain coordination scenarios
-  **Gas Optimized**: Efficient batch operations and modular architecture
-  **Autonomous**: Direct agent-to-agent coordination without trusted intermediaries
-  **ERC Compatible**: Works alongside ERC-7683 and ERC-7521

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent Coordination Framework              │
├─────────────────────────────────────────────────────────────┤
│  Core Framework          │  Security Module                 │
│  ├─ Intent Management    │  ├─ Access Control               │
│  ├─ Multi-party Consensus│  ├─ Encryption Engine            │
│  ├─ EIP-712 Signatures   │  ├─ Timelock Protection          │
│  └─ Execution Engine     │  └─ Emergency Procedures         │
├─────────────────────────────────────────────────────────────┤
│  Optional Modules                                           │
│  ├─ Batch Coordination   ├─ Cross-Chain Bridge              │
│  ├─ Agent Discovery      └─ Integration Examples            │
└─────────────────────────────────────────────────────────────┘
```

## Quick Start

### Prerequisites

- [Foundry](https://getfoundry.sh/) (includes `forge`, `cast`, `anvil`)
- [Git](https://git-scm.com/)

### Installation

```bash
# Clone the repository
git clone https://github.com/kbryan/eip-8001
cd eip-8001

# Install dependencies
forge install

# Build the project
forge build

# Run tests
forge test
```

### Basic Usage

```solidity
import "./src/AgentCoordinationFramework.sol";
import "./src/AgentSecurityModule.sol";

// Deploy framework
AgentCoordinationFramework coordination = new AgentCoordinationFramework();
AgentSecurityModule security = new AgentSecurityModule(address(coordination));

// Create secure coordination intent
IAgentCoordinationCore.AgentIntent memory intent = IAgentCoordinationCore.AgentIntent({
    payloadHash: computedHash,
    expiry: uint64(block.timestamp + 3600),
    nonce: 1,
    chainId: uint32(block.chainid),
    agentId: msg.sender,
    coordinationType: keccak256("ARBITRAGE_V1"),
    maxGasCost: 500000,
    priority: 200,
    dependencyHash: bytes32(0),
    securityLevel: uint8(AgentSecurityModule.SecurityLevel.ENHANCED),
    participants: [agent1, agent2, agent3],
    coordinationValue: 10 ether
});

// Sign and propose coordination
bytes memory signature = signIntent(intent, privateKey);
bytes32 intentHash = coordination.proposeCoordination(intent, signature, payload);
```

## Security Levels

The AgentSecurityModule provides four progressive security levels:

| Level | Timelock | Encryption | Proof Required | Use Cases |
|-------|----------|------------|----------------|-----------|
| **BASIC** | None | Obfuscation | ❌ | Public coordination, testing |
| **STANDARD** | 5 min | XOR | ❌ | Regular DeFi operations |
| **ENHANCED** | 30 min | Multi-layer | ✅ | High-value arbitrage |
| **MAXIMUM** | 2 hours | Advanced + PKI | ✅ | Critical infrastructure |

### Security Example

```solidity
// Create enhanced security context
address[] memory participants = [alice, bob, charlie];
securityModule.createSecurityContext(
    intentHash,
    AgentSecurityModule.SecurityLevel.ENHANCED,
    participants,
    1800 // 30 minutes
);

// Encrypt sensitive coordination data
(bytes memory encryptedData, bytes memory keyData) = 
    securityModule.encryptCoordinationData(
        strategicData,
        participants,
        AgentSecurityModule.SecurityLevel.ENHANCED
    );
```

## Testing

### Run All Tests

```bash
# Basic test run
forge test

# Verbose output with gas reporting
forge test -vvv --gas-report

# Run specific test suite
forge test --match-contract AgentSecurityModuleTest -vvv
```

### Test Coverage

```bash
# Generate coverage report
forge coverage

# Generate detailed HTML coverage report
forge coverage --report lcov
genhtml lcov.info -o coverage-report
```

### Current Test Results

```
Ran 7 tests for test/FixedAddressTest.t.sol:FixedAddressTest
Suite result: ok. 7 passed; 0 failed

Ran 29 tests for test/AgentSecurityModule.t.sol:AgentSecurityModuleTest  
Suite result: ok. 29 passed; 0 failed

Test result: ok. 36 passed; 0 failed; finished in 8.34s
```

## Gas Benchmarks

| Operation | Security Level | Gas Cost | Description |
|-----------|---------------|----------|-------------|
| Create Context | BASIC | ~200k | Basic security setup |
| Create Context | ENHANCED | ~260k | Enhanced security with encryption |
| Propose Coordination | All | ~180k | Intent proposal with signatures |
| Accept Coordination | All | ~90k | Participant acceptance |
| Execute Coordination | All | ~120k | Final execution |
| Encrypt Data | STANDARD | ~45k | Standard encryption |
| Encrypt Data | ENHANCED | ~85k | Multi-layer encryption |

## Integration Examples

### DeFi Arbitrage Coordination

```solidity
// Create secure arbitrage coordination
bytes32 intentHash = integrationContract.createSecureArbitrageCoordination(
    tokenA,           // WETH
    tokenB,           // USDC  
    participants,     // [bot1, bot2, bot3]
    expectedProfit,   // 5 ETH
    AgentSecurityModule.SecurityLevel.ENHANCED
);

// Participants accept with security validation
integrationContract.acceptSecureCoordination(
    intentHash,
    signedAttestation,
    securityProof
);

// Execute with automatic decryption
integrationContract.executeSecureCoordination(
    intentHash,
    encryptedStrategy,
    executionData,
    finalSecurityProof
);
```

### Cross-Chain MEV Coordination

```solidity
// Setup cross-chain coordination
CrossChainConfig memory config = CrossChainConfig({
    targetChains: [1, 137, 42161],        // Ethereum, Polygon, Arbitrum
    targetContracts: [addr1, addr2, addr3],
    executionData: [data1, data2, data3],
    values: [0, 0, 0],
    timeoutBlocks: 100,
    dependencyHash: parentIntentHash,
    requireAtomicity: true
});

crossChainModule.initiateCrossChainCoordination(
    intentHash,
    config,
    proofData
);
```

##  Contract Architecture

### Core Contracts

- **`AgentCoordinationFramework.sol`**: Main coordination logic and EIP-712 implementation
- **`AgentSecurityModule.sol`**: Multi-level security, encryption, and access control
- **`SecurityIntegrationExample.sol`**: Complete integration examples

### Optional Modules

- **`AgentDiscoveryModule.sol`**: Agent registration and reputation system
- **`BatchCoordinationModule.sol`**: Efficient batch operations
- **`CrossChainCoordinationModule.sol`**: Cross-chain coordination primitives

### Interfaces

- **`IAgentCoordinationCore`**: Core coordination interface
- **`IAgentSecurityModule`**: Security module interface
- **`IAgentCrossChain`**: Cross-chain coordination interface

## Security Considerations

### Cryptographic Security

- **EIP-712 Compliance**: All signatures use structured data signing with domain separation
- **Replay Protection**: Monotonic nonces prevent replay attacks
- **Hash Verification**: All payloads verified during execution

### Economic Security

- **Value Limits**: `coordinationValue` enables risk assessment
- **Gas Protection**: `maxGasCost` prevents griefing attacks
- **Collateral Requirements**: Can be extended with economic guarantees

### Operational Security

- **Emergency Procedures**: Creator and owner can revoke access
- **Upgrade Paths**: Security levels can be upgraded but not downgraded
- **Audit Trail**: All events logged for transparency

### Known Limitations

- **On-chain Transparency**: All coordination is publicly visible (use encryption for sensitive data)
- **Gas Costs**: Complex coordination may be expensive
- **Finality Dependencies**: Cross-chain operations require careful finality handling

## Contributing

We welcome contributions! Please see our [Contributing Guidelines](CONTRIBUTING.md) for details.

### Development Setup

```bash
# Clone with submodules
git clone --recursive https://github.com/your-org/eip-8001-agent-coordination

# Install pre-commit hooks
pre-commit install

# Run full test suite
make test

# Format code
make format

# Generate documentation
make docs
```

### Code Standards

- **Solidity Style**: Follow [Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html)
- **Testing**: Maintain >95% test coverage
- **Documentation**: Document all public functions with NatSpec
- **Security**: All PRs require security review

## Documentation

- [EIP-8001 Specification](https://eips.ethereum.org/EIPS/eip-8001)
- [Security Module Guide](security-module.md)
- [Integration Examples](integration-examples.md)
- [API Reference](docs/api-reference.md)
- [Deployment Guide](docs/deployment.md)

## Ecosystem

### Compatible Standards

- [ERC-7683](https://eips.ethereum.org/EIPS/eip-7683): Cross-chain intent framework
- [ERC-7521](https://eips.ethereum.org/EIPS/eip-7521): General intent framework
- [EIP-712](https://eips.ethereum.org/EIPS/eip-712): Typed structured data hashing

### Use Cases

- **DeFi Arbitrage**: Multi-DEX arbitrage coordination
- **MEV Extraction**: Coordinated MEV strategies
- **Cross-Chain Operations**: Atomic cross-chain transactions
- **Market Making**: Collaborative liquidity provision
- **Yield Farming**: Coordinated farming strategies

## License

This project is licensed under [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/) - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Ethereum Foundation for EIP process and standards
- Foundry team for excellent development tools
- OpenZeppelin for security best practices
- Community reviewers and contributors

## Contact

- **Specification Discussion**: [Ethereum Magicians](https://ethereum-magicians.org/t/erc-8001-secure-intents-a-cryptographic-framework-for-autonomous-agent-coordination-draft-erc-8001/24989)
- **Issues**: [GitHub Issues](https://github.com/kbryan/eip-8001/issues)
- **Security**: kwame.bryan@gmail.com

---

**️ Disclaimer**: This software is experimental and under active development. Use at your own risk in production environments. Always conduct thorough testing and security audits before deploying to mainnet.
