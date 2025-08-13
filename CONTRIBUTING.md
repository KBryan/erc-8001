# Contributing to EIP-8001 Agent Coordination Framework

Thank you for your interest in contributing to the Agent Coordination Framework! This document provides guidelines and information for contributors.

## Table of Contents

1. [Code of Conduct](#code-of-conduct)
2. [Getting Started](#getting-started)
3. [Development Setup](#development-setup)
4. [Contribution Guidelines](#contribution-guidelines)
5. [Code Standards](#code-standards)
6. [Testing Requirements](#testing-requirements)
7. [Security Guidelines](#security-guidelines)
8. [Documentation Standards](#documentation-standards)
9. [Submission Process](#submission-process)
10. [Community](#community)

## Code of Conduct

### Our Pledge

We are committed to making participation in this project a harassment-free experience for everyone, regardless of age, body size, disability, ethnicity, gender identity and expression, level of experience, nationality, personal appearance, race, religion, or sexual identity and orientation.

### Our Standards

**Positive behavior includes:**
- Using welcoming and inclusive language
- Being respectful of differing viewpoints and experiences
- Gracefully accepting constructive criticism
- Focusing on what is best for the community
- Showing empathy towards other community members

**Unacceptable behavior includes:**
- Harassment, trolling, insulting/derogatory comments
- Public or private harassment
- Publishing others' private information without permission
- Other conduct which could reasonably be considered inappropriate

### Enforcement

Instances of abusive, harassing, or otherwise unacceptable behavior may be reported by contacting the project team. All complaints will be reviewed and investigated promptly and fairly.

## Getting Started

### Prerequisites

Before contributing, ensure you have:

- **Foundry**: Latest version installed ([installation guide](https://getfoundry.sh/))
- **Git**: For version control
- **Node.js** (optional): For additional tooling
- **VS Code** (recommended): With Solidity extensions

### First-Time Setup

1. **Fork the repository**
```bash
git clone https://github.com/kbryan/erc-8001.git
cd eip-8001
```

2. **Set up the development environment**
```bash
# Install dependencies
forge install

# Build the project
forge build

# Run tests to ensure everything works
forge test -vvvv
```

3. **Set up pre-commit hooks** (optional but recommended)
```bash
# Install pre-commit (if you have Python/pip)
pip install pre-commit
pre-commit install
```

## Development Setup

### Repository Structure

```
eip-8001/
├── src/                          # Smart contracts
│   ├── AgentCoordinationFramework.sol
│   ├── AgentSecurityModule.sol
│   └── modules/                  # Optional modules
├── test/                         # Test files
│   ├── AgentCoordinationFramework.t.sol
│   ├── AgentSecurityModule.t.sol
│   └── integration/              # Integration tests
├── docs/                         # Documentation
├── scripts/                      # Deployment and utility scripts
├── foundry.toml                 # Foundry configuration
└── README.md                    # Project overview
```

### Development Workflow

1. **Create a feature branch**
```bash
git checkout -b feature/your-feature-name
```

2. **Make your changes**
- Write code following our [style guidelines](#code-standards)
- Add comprehensive tests
- Update documentation as needed

3. **Test your changes**
```bash
# Run all tests
forge test

# Run with coverage
forge coverage

# Run gas analysis
forge test --gas-report
```

4. **Commit your changes**
```bash
git add .
git commit -m "feat: add new coordination type for flash loans"
```

5. **Push and create a pull request**
```bash
git push origin feature/your-feature-name
```

## Contribution Guidelines

### Types of Contributions

We welcome several types of contributions:

#### Bug Fixes
- Fix security vulnerabilities
- Resolve gas optimization issues
- Correct logic errors
- Improve error handling

#### New Features
- New coordination types
- Additional security modules
- Integration helpers
- Performance optimizations

#### Documentation
- API documentation improvements
- Tutorial and guide creation
- Code comment enhancements
- Example implementations

#### Testing
- Additional test cases
- Integration test scenarios
- Fuzzing and property-based tests
- Gas optimization tests

#### Infrastructure
- CI/CD improvements
- Development tooling
- Deployment scripts
- Monitoring and analytics

### Contribution Priorities

**High Priority:**
- Security improvements and audits
- Gas optimization
- Core feature enhancements
- Documentation improvements

**Medium Priority:**
- New coordination types
- Optional module development
- Integration examples
- Performance testing

**Low Priority:**
- Code style improvements
- Minor optimizations
- Additional tooling

## Code Standards

### Solidity Style Guide

We follow the [official Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html) with these additions:

#### Naming Conventions

```solidity
// Contracts: PascalCase
contract AgentCoordinationFramework { }

// Functions and variables: camelCase
function proposeCoordination() { }
uint256 coordinationValue;

// Constants: UPPER_SNAKE_CASE
uint256 constant MAX_PARTICIPANTS = 100;

// Internal/private functions: _camelCase with underscore prefix
function _validateIntent() internal { }

// Events: PascalCase
event CoordinationProposed();

// Modifiers: camelCase
modifier onlyFramework() { }
```

#### Code Organization

```solidity
contract ExampleContract {
    // 1. Type declarations
    using SafeMath for uint256;
    
    // 2. State variables
    mapping(bytes32 => CoordinationState) private _coordinations;
    
    // 3. Events
    event CoordinationProposed(bytes32 indexed intentHash);
    
    // 4. Modifiers
    modifier onlyAuthorized() { }
    
    // 5. Constructor
    constructor() { }
    
    // 6. External functions
    function proposeCoordination() external { }
    
    // 7. Public functions
    function getStatus() public view { }
    
    // 8. Internal functions
    function _validateIntent() internal { }
    
    // 9. Private functions
    function _computeHash() private pure { }
}
```

#### Documentation Standards

Use NatSpec for all public interfaces:

```solidity
/**
 * @title Agent Coordination Framework
 * @dev Implements secure multi-party coordination for autonomous agents
 * @notice This contract allows agents to propose, accept, and execute coordinated operations
 */
contract AgentCoordinationFramework {
    
    /**
     * @notice Proposes a new coordination intent
     * @dev Creates a coordination that participants can accept and execute
     * @param intent The coordination intent containing all coordination parameters
     * @param signature EIP-712 signature of the intent by the proposing agent
     * @param payload Coordination payload with execution data
     * @return intentHash Unique identifier for the proposed coordination
     * @custom:security Requires valid EIP-712 signature and intent validation
     * @custom:gas-cost Approximately 180k gas plus 20k per participant
     */
    function proposeCoordination(
        AgentIntent calldata intent,
        bytes calldata signature,
        CoordinationPayload calldata payload
    ) external returns (bytes32 intentHash) {
        // Implementation
    }
}
```

#### Gas Optimization Guidelines

1. **Use appropriate data types**
```solidity
// Good: Use smaller types when possible
uint64 expiry;
uint32 chainId;
uint8 securityLevel;

// Avoid: Unnecessary large types
uint256 expiry; // when uint64 is sufficient
```

2. **Pack structs efficiently**
```solidity
// Good: Packed struct
struct PackedStruct {
    uint128 value1;  // 16 bytes
    uint128 value2;  // 16 bytes
    bool flag;       // 1 byte
    uint8 level;     // 1 byte
    // Total: 34 bytes (2 storage slots)
}

// Bad: Unpacked struct
struct UnpackedStruct {
    uint256 value1;  // 32 bytes
    bool flag;       // 1 byte
    uint256 value2;  // 32 bytes
    uint8 level;     // 1 byte
    // Total: 66 bytes (3 storage slots)
}
```

3. **Use memory efficiently**
```solidity
// Good: Use calldata for read-only arrays
function processParticipants(address[] calldata participants) external {
    // participants is read-only, no copying needed
}

// Good: Use memory for arrays you modify
function processAndModify(address[] memory participants) internal {
    participants[0] = newAddress; // Can modify
}
```

4. **Optimize loops**
```solidity
// Good: Cache array length
uint256 length = participants.length;
for (uint256 i = 0; i < length; ++i) {
    // Process participants[i]
}

// Good: Use unchecked for safe arithmetic
for (uint256 i = 0; i < participants.length;) {
    // Process participants[i]
    unchecked { ++i; }
}
```

## Testing Requirements

### Test Coverage Requirements

- **Minimum coverage**: 95% for all new code
- **Critical functions**: 100% coverage required
- **Edge cases**: Must be thoroughly tested
- **Gas limits**: Must not exceed reasonable bounds

### Test Categories

#### Unit Tests
Test individual functions in isolation:

```solidity
contract AgentCoordinationFrameworkTest is Test {
    function testProposeCoordinationSuccess() public {
        // Arrange
        AgentIntent memory intent = createValidIntent();
        bytes memory signature = signIntent(intent, privateKey);
        CoordinationPayload memory payload = createValidPayload();
        
        // Act
        bytes32 intentHash = coordination.proposeCoordination(intent, signature, payload);
        
        // Assert
        assertEq(intentHash, expectedHash);
        (uint8 status,,,) = coordination.getCoordinationStatus(intentHash);
        assertEq(status, 0); // PROPOSED
    }
    
    function testProposeCoordinationFailsWithExpiredIntent() public {
        // Test specific failure case
        AgentIntent memory intent = createValidIntent();
        intent.expiry = uint64(block.timestamp - 1); // Expired
        
        vm.expectRevert("Intent expired");
        coordination.proposeCoordination(intent, signature, payload);
    }
}
```

#### Integration Tests
Test interactions between contracts:

```solidity
contract IntegrationTest is Test {
    function testSecureCoordinationEndToEnd() public {
        // Test complete flow from proposal to execution
        // with security module integration
    }
}
```

#### Property-Based Tests
Test invariants and properties:

```solidity
contract PropertyTest is Test {
    function testInvariantNonceAlwaysIncreases(uint64 startNonce) public {
        vm.assume(startNonce < type(uint64).max);
        
        // Setup
        uint64 initialNonce = coordination.getAgentNonce(agent);
        
        // Execute coordination
        _executeValidCoordination();
        
        // Verify nonce increased
        uint64 finalNonce = coordination.getAgentNonce(agent);
        assertGt(finalNonce, initialNonce);
    }
}
```

#### Gas Tests
Verify gas usage is reasonable:

```solidity
contract GasTest is Test {
    function testCoordinationGasCost() public {
        uint256 gasStart = gasleft();
        coordination.proposeCoordination(intent, signature, payload);
        uint256 gasUsed = gasStart - gasleft();
        
        // Should be under 200k gas for basic coordination
        assertLt(gasUsed, 200_000);
    }
}
```

### Testing Best Practices

1. **Test Setup**
```solidity
function setUp() public {
    // Deploy contracts
    coordination = new AgentCoordinationFramework();
    security = new AgentSecurityModule(address(coordination));
    
    // Setup test accounts with proper private key/address pairs
    aliceKey = 0x1;
    alice = vm.addr(aliceKey);
    
    // Fund accounts
    vm.deal(alice, 10 ether);
}
```

2. **Use Descriptive Test Names**
```solidity
// Good
function testProposeCoordinationFailsWhenIntentExpired() public { }
function testAcceptCoordinationSucceedsWithValidAttestation() public { }

// Bad
function testPropose() public { }
function testAccept() public { }
```

3. **Test Edge Cases**
```solidity
function testWithMaxParticipants() public {
    address[] memory participants = new address[](100); // Max allowed
    // Test with maximum participants
}

function testWithMinimumValues() public {
    // Test with minimum coordination value, minimum timelock, etc.
}
```

## Security Guidelines

### Security Review Process

All security-related changes must undergo thorough review:

1. **Self-review checklist**
2. **Peer review by maintainers**
3. **Security audit for major changes**
4. **Gas analysis for optimizations**

### Common Security Patterns

#### Access Control
```solidity
modifier onlyFramework() {
    require(msg.sender == COORDINATION_FRAMEWORK, "Unauthorized: framework only");
    _;
}

modifier onlyParticipant(bytes32 intentHash) {
    require(_participantAccess[intentHash][msg.sender], "Unauthorized: not participant");
    _;
}
```

#### Reentrancy Protection
```solidity
modifier nonReentrant() {
    require(!_locked, "Reentrant call");
    _locked = true;
    _;
    _locked = false;
}
```

#### Input Validation
```solidity
function createSecurityContext(
    bytes32 intentHash,
    SecurityLevel level,
    address[] calldata participants,
    uint256 customTimelock
) external validSecurityLevel(level) {
    require(intentHash != bytes32(0), "Invalid intent hash");
    require(participants.length > 0 && participants.length <= MAX_PARTICIPANTS, "Invalid participant count");
    require(customTimelock >= _minTimelocks[level], "Timelock too short");
    
    // Implementation
}
```

#### Safe External Calls
```solidity
// Good: Check return values
(bool success, bytes memory data) = target.call(payload);
require(success, "External call failed");

// Good: Use try/catch for external calls
try externalContract.someFunction() returns (bool result) {
    // Handle success
} catch Error(string memory reason) {
    // Handle revert with reason
} catch {
    // Handle other failures
}
```

### Security Checklist

Before submitting security-related code:

- [ ] All inputs validated
- [ ] Access controls implemented correctly
- [ ] Reentrancy protection where needed
- [ ] Integer overflow/underflow protection
- [ ] External calls handled safely
- [ ] Gas limits considered
- [ ] Storage layout doesn't create vulnerabilities
- [ ] EIP-712 signatures validated properly
- [ ] Replay protection implemented
- [ ] Emergency procedures available

## Documentation Standards

### Code Documentation

1. **Contract-level documentation**
```solidity
/**
 * @title Agent Security Module
 * @author Your Name
 * @notice Provides multi-level security for agent coordination
 * @dev Implements four security levels with encryption and access control
 * @custom:security-contact security@yourproject.org
 */
contract AgentSecurityModule {
```

2. **Function documentation**
```solidity
/**
 * @notice Creates a security context for coordination
 * @dev Sets up encryption, timelock, and access control
 * @param intentHash Hash of the coordination intent
 * @param level Security level (0-3)
 * @param participants Array of authorized participants
 * @param customTimelock Custom timelock period in seconds
 * @custom:requirements
 * - Only callable by coordination framework
 * - Intent hash must not already exist
 * - Custom timelock must meet minimum for level
 * @custom:effects
 * - Creates security context
 * - Generates encryption keys
 * - Sets up participant access
 */
function createSecurityContext(/*...*/) external { }
```

### README and Guide Updates

When adding new features, update:

- Main README.md with feature overview
- API documentation with new interfaces
- Integration examples with usage patterns
- Security documentation with new considerations

## Submission Process

### Pull Request Guidelines

1. **PR Title Format**
```
type(scope): description

Examples:
feat(security): add zero-knowledge proof validation
fix(core): resolve signature verification edge case
docs(api): add flash loan coordination examples
test(integration): add cross-chain coordination tests
```

2. **PR Description Template**
```markdown
## Description
Brief description of the changes

## Type of Change
- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## Testing
- [ ] I have added tests that prove my fix is effective or that my feature works
- [ ] New and existing unit tests pass locally with my changes
- [ ] I have run the gas analysis and costs are reasonable

## Security
- [ ] I have considered security implications of my changes
- [ ] I have followed secure coding practices
- [ ] I have updated security documentation if needed

## Documentation
- [ ] I have updated the documentation accordingly
- [ ] I have added inline code comments where necessary
- [ ] I have updated API documentation if interfaces changed

## Gas Analysis
- [ ] I have analyzed gas costs for my changes
- [ ] Gas costs are within acceptable ranges
- [ ] I have optimized for gas efficiency where possible
```

### Review Process

1. **Automated checks** (CI/CD):
    - Compilation success
    - Test execution
    - Gas analysis
    - Code style validation

2. **Manual review**:
    - Code quality assessment
    - Security review
    - Gas optimization review
    - Documentation completeness

3. **Final approval**:
    - Two maintainer approvals required
    - All checks must pass
    - Conflicts resolved

### Merge Requirements

- [ ] All CI checks pass
- [ ] At least 2 maintainer approvals
- [ ] No unresolved conversations
- [ ] Branch is up to date with main
- [ ] Gas costs within acceptable limits
- [ ] Security review completed (for security changes)

## Community

### Communication Channels

- **GitHub Issues**: Bug reports and feature requests
- **GitHub Discussions**: General questions and ideas
- **Ethereum Magicians**: EIP-8001 specification discussion
- **Discord**: Real-time community chat (link in README)

### Getting Help

1. **Check existing documentation** first
2. **Search closed issues** for similar problems
3. **Ask in GitHub Discussions** for general questions
4. **Create an issue** for bugs or feature requests

### Recognition

Contributors will be recognized:

- In the project README contributors section
- In release notes for significant contributions
- Through GitHub's contribution tracking
- With potential co-authorship on academic papers

## License

By contributing, you agree that your contributions will be licensed under the same CC0-1.0 license that covers the project.

---

Thank you for contributing to the Agent Coordination Framework! Your efforts help advance the state of autonomous agent coordination in the Ethereum ecosystem.