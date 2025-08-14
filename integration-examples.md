# Integration Examples

This document provides comprehensive examples of integrating the Agent Coordination Framework into real-world DeFi and MEV scenarios.

## Table of Contents

1. [DeFi Arbitrage Coordination](#defi-arbitrage-coordination)
2. [MEV Extraction Coordination](#mev-extraction-coordination)
3. [Cross-Chain Atomic Swaps](#cross-chain-atomic-swaps)
4. [Collaborative Market Making](#collaborative-market-making)
5. [Yield Farming Coordination](#yield-farming-coordination)
6. [Liquidation Coordination](#liquidation-coordination)
7. [Flash Loan Coordination](#flash-loan-coordination)

## DeFi Arbitrage Coordination

### Simple Two-DEX Arbitrage

```solidity
pragma solidity ^0.8.20;

import "../src/AgentCoordinationFramework.sol";
import "../src/AgentSecurityModule.sol";

contract ArbitrageCoordinator {
    AgentCoordinationFramework public immutable framework;
    AgentSecurityModule public immutable security;
    
    struct ArbitrageStrategy {
        address tokenA;
        address tokenB;
        address dexA;
        address dexB;
        uint256 amount;
        uint256 minProfit;
        uint256 deadline;
    }
    
    constructor(address _framework, address _security) {
        framework = AgentCoordinationFramework(_framework);
        security = AgentSecurityModule(_security);
    }
    
    function proposeArbitrage(
        ArbitrageStrategy calldata strategy,
        address[] calldata participants,
        AgentSecurityModule.SecurityLevel securityLevel
    ) external returns (bytes32 intentHash) {
        
        // Create coordination intent
        IAgentCoordinationCore.AgentIntent memory intent = IAgentCoordinationCore.AgentIntent({
            payloadHash: bytes32(0), // Will be computed
            expiry: uint64(strategy.deadline),
            nonce: framework.getAgentNonce(msg.sender) + 1,
            chainId: uint32(block.chainid),
            agentId: msg.sender,
            coordinationType: keccak256("ARBITRAGE_V1"),
            maxGasCost: 800000, // Higher for DEX interactions
            priority: 255, // Highest priority for arbitrage
            dependencyHash: bytes32(0),
            securityLevel: uint8(securityLevel),
            participants: participants,
            coordinationValue: strategy.minProfit
        });
        
        // Encode strategy data
        bytes memory strategyData = abi.encode(
            strategy.tokenA,
            strategy.tokenB,
            strategy.dexA,
            strategy.dexB,
            strategy.amount,
            strategy.minProfit,
            block.timestamp,
            msg.sender
        );
        
        // Create coordination payload
        IAgentCoordinationCore.CoordinationPayload memory payload = IAgentCoordinationCore.CoordinationPayload({
            version: keccak256("v1"),
            coordinationType: intent.coordinationType,
            coordinationData: strategyData,
            conditionsHash: keccak256(abi.encodePacked("min_profit:", strategy.minProfit)),
            timestamp: block.timestamp,
            metadata: ""
        });
        
        intent.payloadHash = framework.getPayloadHash(payload);
        
        // Create security context with appropriate level
        security.createSecurityContext(
            keccak256(abi.encode(intent)),
            securityLevel,
            participants,
            _getTimelockForValue(strategy.minProfit)
        );
        
        // If enhanced security, encrypt the strategy
        if (securityLevel >= AgentSecurityModule.SecurityLevel.ENHANCED) {
            (bytes memory encryptedData, bytes memory keyData) = security.encryptCoordinationData(
                strategyData,
                participants,
                securityLevel
            );
            payload.coordinationData = encryptedData;
            payload.metadata = keyData;
            intent.payloadHash = framework.getPayloadHash(payload);
        }
        
        // Sign intent (simplified - in practice use proper EIP-712)
        bytes memory signature = _signIntent(intent);
        
        // Propose coordination
        return framework.proposeCoordination(intent, signature, payload);
    }
    
    function executeArbitrage(
        bytes32 intentHash,
        ArbitrageStrategy calldata strategy
    ) external returns (uint256 profit) {
        
        // Get coordination status
        (uint8 status,,,address[] memory acceptedBy,) = framework.getCoordinationStatus(intentHash);
        require(status == 1, "Coordination not ready"); // READY status
        require(acceptedBy.length > 0, "No participants accepted");
        
        // Validate security requirements
        AgentSecurityModule.SecurityContext memory context = security.getSecurityContext(intentHash);
        (bool securityValid,) = security.validateSecurityLevel(
            intentHash,
            context.level,
            "" // Could include additional proofs
        );
        require(securityValid, "Security validation failed");
        
        // Execute arbitrage strategy
        profit = _executeArbitrageStrategy(strategy);
        
        // Distribute profits among participants
        _distributeProfits(profit, acceptedBy);
        
        return profit;
    }
    
    function _executeArbitrageStrategy(ArbitrageStrategy memory strategy) internal returns (uint256) {
        // 1. Buy on DEX A
        uint256 amountOut = _swapOnDex(strategy.dexA, strategy.tokenA, strategy.tokenB, strategy.amount);
        
        // 2. Sell on DEX B  
        uint256 finalAmount = _swapOnDex(strategy.dexB, strategy.tokenB, strategy.tokenA, amountOut);
        
        // Calculate profit
        require(finalAmount > strategy.amount, "Arbitrage not profitable");
        return finalAmount - strategy.amount;
    }
    
    function _swapOnDex(address dex, address tokenIn, address tokenOut, uint256 amountIn) internal returns (uint256) {
        // Simplified DEX interaction - would use actual DEX interfaces
        // UniswapV2, SushiSwap, etc.
        return amountIn * 101 / 100; // Simplified 1% price difference
    }
    
    function _distributeProfits(uint256 totalProfit, address[] memory participants) internal {
        uint256 profitPerParticipant = totalProfit / participants.length;
        for (uint256 i = 0; i < participants.length; i++) {
            // Transfer profit share (simplified)
            payable(participants[i]).transfer(profitPerParticipant);
        }
    }
    
    function _getTimelockForValue(uint256 value) internal pure returns (uint256) {
        if (value < 1 ether) return 0;
        if (value < 10 ether) return 300;
        if (value < 100 ether) return 1800;
        return 7200;
    }
    
    function _signIntent(IAgentCoordinationCore.AgentIntent memory intent) internal view returns (bytes memory) {
        // Simplified signing - use proper EIP-712 in production
        bytes32 hash = keccak256(abi.encode(intent));
        return abi.encodePacked(hash, uint8(27)); // Simplified signature
    }
}
```

### Multi-DEX Arbitrage with Complex Routing

```solidity
contract MultiDexArbitrage {
    struct ComplexArbitrageRoute {
        address[] tokens;           // Token path: [WETH, USDC, DAI, WETH]
        address[] dexes;           // DEX for each hop: [Uniswap, SushiSwap, Curve]
        uint256[] minAmountsOut;   // Minimum amounts for each hop
        uint256 initialAmount;
        uint256 deadline;
        bytes routingData;         // Complex routing parameters
    }
    
    function proposeComplexArbitrage(
        ComplexArbitrageRoute calldata route,
        address[] calldata participants
    ) external returns (bytes32 intentHash) {
        
        // Encode complex route data
        bytes memory routeData = abi.encode(
            route.tokens,
            route.dexes,
            route.minAmountsOut,
            route.initialAmount,
            route.deadline,
            route.routingData,
            block.timestamp,
            msg.sender
        );
        
        // Use ENHANCED security for complex routes
        AgentSecurityModule.SecurityLevel securityLevel = AgentSecurityModule.SecurityLevel.ENHANCED;
        
        // Calculate expected value and set coordination parameters
        uint256 expectedProfit = _calculateExpectedProfit(route);
        
        // Create intent with complex routing type
        IAgentCoordinationCore.AgentIntent memory intent = IAgentCoordinationCore.AgentIntent({
            payloadHash: bytes32(0),
            expiry: uint64(route.deadline),
            nonce: framework.getAgentNonce(msg.sender) + 1,
            chainId: uint32(block.chainid),
            agentId: msg.sender,
            coordinationType: keccak256("COMPLEX_ARBITRAGE_V1"),
            maxGasCost: 2000000, // Higher for complex routes
            priority: 200,
            dependencyHash: bytes32(0),
            securityLevel: uint8(securityLevel),
            participants: participants,
            coordinationValue: expectedProfit
        });
        
        // Create payload with encrypted route data
        (bytes memory encryptedRoute, bytes memory keyData) = security.encryptCoordinationData(
            routeData,
            participants,
            securityLevel
        );
        
        IAgentCoordinationCore.CoordinationPayload memory payload = IAgentCoordinationCore.CoordinationPayload({
            version: keccak256("COMPLEX_V1"),
            coordinationType: intent.coordinationType,
            coordinationData: encryptedRoute,
            conditionsHash: keccak256(abi.encodePacked("complex_route:", route.tokens.length)),
            timestamp: block.timestamp,
            metadata: keyData
        });
        
        intent.payloadHash = framework.getPayloadHash(payload);
        
        // Create security context with 30-minute timelock
        security.createSecurityContext(
            keccak256(abi.encode(intent)),
            securityLevel,
            participants,
            1800
        );
        
        bytes memory signature = _signIntent(intent);
        return framework.proposeCoordination(intent, signature, payload);
    }
    
    function _calculateExpectedProfit(ComplexArbitrageRoute memory route) internal view returns (uint256) {
        // Simulate the route to calculate expected profit
        uint256 currentAmount = route.initialAmount;
        
        for (uint256 i = 0; i < route.tokens.length - 1; i++) {
            currentAmount = _simulateSwap(
                route.dexes[i],
                route.tokens[i],
                route.tokens[i + 1],
                currentAmount
            );
        }
        
        return currentAmount > route.initialAmount ? currentAmount - route.initialAmount : 0;
    }
    
    function _simulateSwap(address dex, address tokenIn, address tokenOut, uint256 amountIn) internal view returns (uint256) {
        // Simulate swap to predict output - would use actual DEX quoters
        return amountIn * 99 / 100; // Simplified simulation
    }
}
```

## MEV Extraction Coordination

### Sandwich Attack Coordination

```solidity
contract SandwichCoordinator {
    struct SandwichOpportunity {
        address targetTx;           // Target transaction to sandwich
        address token0;
        address token1;
        address dex;
        uint256 frontrunAmount;     // Amount for frontrun transaction
        uint256 backrunAmount;      // Amount for backrun transaction
        uint256 expectedProfit;
        uint256 maxGasPrice;
        bytes32 targetTxHash;
    }
    
    function proposeSandwich(
        SandwichOpportunity calldata opportunity,
        address[] calldata participants
    ) external returns (bytes32 intentHash) {
        
        // Encode sandwich strategy
        bytes memory strategyData = abi.encode(
            opportunity.targetTx,
            opportunity.token0,
            opportunity.token1,
            opportunity.dex,
            opportunity.frontrunAmount,
            opportunity.backrunAmount,
            opportunity.expectedProfit,
            opportunity.maxGasPrice,
            opportunity.targetTxHash,
            block.timestamp
        );
        
        // Use MAXIMUM security for MEV operations
        AgentSecurityModule.SecurityLevel securityLevel = AgentSecurityModule.SecurityLevel.MAXIMUM;
        
        IAgentCoordinationCore.AgentIntent memory intent = IAgentCoordinationCore.AgentIntent({
            payloadHash: bytes32(0),
            expiry: uint64(block.timestamp + 300), // 5-minute window for MEV
            nonce: framework.getAgentNonce(msg.sender) + 1,
            chainId: uint32(block.chainid),
            agentId: msg.sender,
            coordinationType: keccak256("SANDWICH_MEV_V1"),
            maxGasCost: 1500000,
            priority: 255, // Maximum priority for MEV
            dependencyHash: opportunity.targetTxHash, // Dependent on target transaction
            securityLevel: uint8(securityLevel),
            participants: participants,
            coordinationValue: opportunity.expectedProfit
        });
        
        // Encrypt sensitive MEV strategy
        (bytes memory encryptedStrategy, bytes memory keyData) = security.encryptCoordinationData(
            strategyData,
            participants,
            securityLevel
        );
        
        IAgentCoordinationCore.CoordinationPayload memory payload = IAgentCoordinationCore.CoordinationPayload({
            version: keccak256("MEV_V1"),
            coordinationType: intent.coordinationType,
            coordinationData: encryptedStrategy,
            conditionsHash: keccak256(abi.encodePacked("target_tx:", opportunity.targetTxHash)),
            timestamp: block.timestamp,
            metadata: keyData
        });
        
        intent.payloadHash = framework.getPayloadHash(payload);
        
        // Create security context with 2-hour timelock (but can be executed early if all accept)
        security.createSecurityContext(
            keccak256(abi.encode(intent)),
            securityLevel,
            participants,
            7200
        );
        
        bytes memory signature = _signIntent(intent);
        return framework.proposeCoordination(intent, signature, payload);
    }
    
    function executeSandwich(
        bytes32 intentHash,
        SandwichOpportunity calldata opportunity
    ) external returns (uint256 profit) {
        
        // Verify all participants accepted and security is satisfied
        _validateExecution(intentHash);
        
        // Decrypt strategy data
        AgentSecurityModule.SecurityContext memory context = security.getSecurityContext(intentHash);
        // In practice, would decrypt the actual strategy data
        
        // Execute frontrun transaction
        _executeFrontrun(opportunity);
        
        // Wait for target transaction to be mined
        _waitForTargetTransaction(opportunity.targetTxHash);
        
        // Execute backrun transaction
        profit = _executeBackrun(opportunity);
        
        // Distribute MEV profits
        (,,,address[] memory participants,) = framework.getCoordinationStatus(intentHash);
        _distributeMEVProfits(profit, participants);
        
        return profit;
    }
    
    function _executeFrontrun(SandwichOpportunity memory opportunity) internal {
        // Execute frontrun swap to move price
        // This would interact with actual DEX contracts
    }
    
    function _executeBackrun(SandwichOpportunity memory opportunity) internal returns (uint256) {
        // Execute backrun swap to capture profit
        // This would interact with actual DEX contracts
        return opportunity.expectedProfit; // Simplified
    }
    
    function _waitForTargetTransaction(bytes32 targetTxHash) internal {
        // In practice, would monitor mempool and block inclusion
        // This is simplified for example purposes
    }
    
    function _distributeMEVProfits(uint256 totalProfit, address[] memory participants) internal {
        // MEV profits often distributed based on contribution/stake
        uint256 profitPerParticipant = totalProfit / participants.length;
        for (uint256 i = 0; i < participants.length; i++) {
            // Transfer profit share
            payable(participants[i]).transfer(profitPerParticipant);
        }
    }
    
    function _validateExecution(bytes32 intentHash) internal view {
        (uint8 status,,,address[] memory acceptedBy,) = framework.getCoordinationStatus(intentHash);
        require(status == 1, "Coordination not ready");
        require(acceptedBy.length > 0, "No participants");
        
        (bool securityValid,) = security.validateSecurityLevel(
            intentHash,
            AgentSecurityModule.SecurityLevel.MAXIMUM,
            "" // Would include complex MEV proofs
        );
        require(securityValid, "Security validation failed");
    }
}
```

## Cross-Chain Atomic Swaps

### Cross-Chain Arbitrage Coordination

```solidity
contract CrossChainArbitrageCoordinator {
    struct CrossChainRoute {
        uint32 sourceChain;
        uint32 targetChain;
        address sourceToken;
        address targetToken;
        address sourceDex;
        address targetDex;
        uint256 amount;
        uint256 minProfit;
        uint256 deadline;
        address bridgeContract;
        bytes bridgeData;
    }
    
    mapping(bytes32 => CrossChainRoute) public routes;
    mapping(bytes32 => mapping(uint32 => bool)) public chainExecutions;
    
    function proposeCrossChainArbitrage(
        CrossChainRoute calldata route,
        address[] calldata participants
    ) external returns (bytes32 intentHash) {
        
        // Encode cross-chain route data
        bytes memory routeData = abi.encode(
            route.sourceChain,
            route.targetChain,
            route.sourceToken,
            route.targetToken,
            route.sourceDex,
            route.targetDex,
            route.amount,
            route.minProfit,
            route.deadline,
            route.bridgeContract,
            route.bridgeData
        );
        
        // Use ENHANCED security for cross-chain operations
        IAgentCoordinationCore.AgentIntent memory intent = IAgentCoordinationCore.AgentIntent({
            payloadHash: bytes32(0),
            expiry: uint64(route.deadline),
            nonce: framework.getAgentNonce(msg.sender) + 1,
            chainId: route.sourceChain,
            agentId: msg.sender,
            coordinationType: keccak256("CROSS_CHAIN_ARBITRAGE_V1"),
            maxGasCost: 3000000, // Higher for cross-chain
            priority: 180,
            dependencyHash: bytes32(0),
            securityLevel: uint8(AgentSecurityModule.SecurityLevel.ENHANCED),
            participants: participants,
            coordinationValue: route.minProfit
        });
        
        // Encrypt route data for security
        (bytes memory encryptedRoute, bytes memory keyData) = security.encryptCoordinationData(
            routeData,
            participants,
            AgentSecurityModule.SecurityLevel.ENHANCED
        );
        
        IAgentCoordinationCore.CoordinationPayload memory payload = IAgentCoordinationCore.CoordinationPayload({
            version: keccak256("CROSS_CHAIN_V1"),
            coordinationType: intent.coordinationType,
            coordinationData: encryptedRoute,
            conditionsHash: keccak256(abi.encodePacked("chains:", route.sourceChain, route.targetChain)),
            timestamp: block.timestamp,
            metadata: keyData
        });
        
        intent.payloadHash = framework.getPayloadHash(payload);
        intentHash = keccak256(abi.encode(intent));
        
        // Store route for cross-chain execution
        routes[intentHash] = route;
        
        // Create security context with longer timelock for cross-chain
        security.createSecurityContext(
            intentHash,
            AgentSecurityModule.SecurityLevel.ENHANCED,
            participants,
            3600 // 1 hour for cross-chain coordination
        );
        
        bytes memory signature = _signIntent(intent);
        framework.proposeCoordination(intent, signature, payload);
        
        return intentHash;
    }
    
    function executeSourceChain(bytes32 intentHash) external returns (bytes32 bridgeTxHash) {
        CrossChainRoute memory route = routes[intentHash];
        require(route.sourceChain == block.chainid, "Wrong chain");
        
        _validateExecution(intentHash);
        
        // Execute source chain swap
        uint256 bridgeAmount = _swapOnSourceChain(route);
        
        // Initiate bridge transaction
        bridgeTxHash = _initiateBridge(route, bridgeAmount);
        
        // Mark source chain execution complete
        chainExecutions[intentHash][route.sourceChain] = true;
        
        return bridgeTxHash;
    }
    
    function executeTargetChain(bytes32 intentHash, bytes32 bridgeTxHash) external returns (uint256 profit) {
        CrossChainRoute memory route = routes[intentHash];
        require(route.targetChain == block.chainid, "Wrong chain");
        require(chainExecutions[intentHash][route.sourceChain], "Source not executed");
        
        // Verify bridge transaction completion
        require(_verifyBridgeCompletion(bridgeTxHash), "Bridge not complete");
        
        // Execute target chain swap
        profit = _swapOnTargetChain(route);
        
        // Mark target chain execution complete
        chainExecutions[intentHash][route.targetChain] = true;
        
        // Distribute cross-chain profits
        (,,,address[] memory participants,) = framework.getCoordinationStatus(intentHash);
        _distributeCrossChainProfits(profit, participants);
        
        return profit;
    }
    
    function _swapOnSourceChain(CrossChainRoute memory route) internal returns (uint256) {
        // Execute swap on source chain DEX
        return _swapOnDex(route.sourceDex, route.sourceToken, address(0), route.amount);
    }
    
    function _swapOnTargetChain(CrossChainRoute memory route) internal returns (uint256) {
        // Execute swap on target chain DEX
        return _swapOnDex(route.targetDex, address(0), route.targetToken, route.amount);
    }
    
    function _initiateBridge(CrossChainRoute memory route, uint256 amount) internal returns (bytes32) {
        // Initiate bridge transaction - would use actual bridge contracts
        // LayerZero, Wormhole, etc.
        return keccak256(abi.encodePacked(route.bridgeContract, amount, block.timestamp));
    }
    
    function _verifyBridgeCompletion(bytes32 bridgeTxHash) internal view returns (bool) {
        // Verify bridge transaction completion - would check actual bridge state
        return true; // Simplified
    }
    
    function _distributeCrossChainProfits(uint256 totalProfit, address[] memory participants) internal {
        uint256 profitPerParticipant = totalProfit / participants.length;
        for (uint256 i = 0; i < participants.length; i++) {
            payable(participants[i]).transfer(profitPerParticipant);
        }
    }
    
    function _swapOnDex(address dex, address tokenIn, address tokenOut, uint256 amountIn) internal returns (uint256) {
        // Simplified DEX interaction
        return amountIn * 98 / 100; // 2% slippage
    }
    
    function _validateExecution(bytes32 intentHash) internal view {
        (uint8 status,,,address[] memory acceptedBy,) = framework.getCoordinationStatus(intentHash);
        require(status == 1, "Coordination not ready");
        require(acceptedBy.length > 0, "No participants");
        
        (bool securityValid,) = security.validateSecurityLevel(
            intentHash,
            AgentSecurityModule.SecurityLevel.ENHANCED,
            ""
        );
        require(securityValid, "Security validation failed");
    }
}
```

## Collaborative Market Making

### Liquidity Pool Coordination

```solidity
contract LiquidityCoordinator {
    struct LiquidityStrategy {
        address pool;               // Uniswap V3 pool
        address token0;
        address token1;
        uint24 fee;
        int24 tickLower;           // Price range lower bound
        int24 tickUpper;           // Price range upper bound
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 duration;          // How long to provide liquidity
        uint256 minFeeReward;      // Minimum fee reward expected
    }
    
    function proposeLiquidityProvision(
        LiquidityStrategy calldata strategy,
        address[] calldata participants
    ) external returns (bytes32 intentHash) {
        
        bytes memory strategyData = abi.encode(
            strategy.pool,
            strategy.token0,
            strategy.token1,
            strategy.fee,
            strategy.tickLower,
            strategy.tickUpper,
            strategy.amount0Desired,
            strategy.amount1Desired,
            strategy.duration,
            strategy.minFeeReward
        );
        
        IAgentCoordinationCore.AgentIntent memory intent = IAgentCoordinationCore.AgentIntent({
            payloadHash: bytes32(0),
            expiry: uint64(block.timestamp + strategy.duration),
            nonce: framework.getAgentNonce(msg.sender) + 1,
            chainId: uint32(block.chainid),
            agentId: msg.sender,
            coordinationType: keccak256("LIQUIDITY_PROVISION_V1"),
            maxGasCost: 1000000,
            priority: 100, // Lower priority than arbitrage
            dependencyHash: bytes32(0),
            securityLevel: uint8(AgentSecurityModule.SecurityLevel.STANDARD),
            participants: participants,
            coordinationValue: strategy.minFeeReward
        });
        
        IAgentCoordinationCore.CoordinationPayload memory payload = IAgentCoordinationCore.CoordinationPayload({
            version: keccak256("LIQUIDITY_V1"),
            coordinationType: intent.coordinationType,
            coordinationData: strategyData,
            conditionsHash: keccak256(abi.encodePacked("pool:", strategy.pool)),
            timestamp: block.timestamp,
            metadata: ""
        });
        
        intent.payloadHash = framework.getPayloadHash(payload);
        
        security.createSecurityContext(
            keccak256(abi.encode(intent)),
            AgentSecurityModule.SecurityLevel.STANDARD,
            participants,
            600 // 10-minute timelock for liquidity operations
        );
        
        bytes memory signature = _signIntent(intent);
        return framework.proposeCoordination(intent, signature, payload);
    }
    
    function executeLiquidityProvision(
        bytes32 intentHash,
        LiquidityStrategy calldata strategy
    ) external returns (uint256 tokenId) {
        
        _validateExecution(intentHash);
        
        // Collect tokens from all participants
        (,,,address[] memory participants,) = framework.getCoordinationStatus(intentHash);
        _collectTokensFromParticipants(strategy, participants);
        
        // Provide liquidity to Uniswap V3 pool
        tokenId = _provideLiquidityToPool(strategy);
        
        // Set up fee collection mechanism
        _setupFeeCollection(intentHash, tokenId, participants);
        
        return tokenId;
    }
    
    function _collectTokensFromParticipants(
        LiquidityStrategy memory strategy,
        address[] memory participants
    ) internal {
        uint256 amount0PerParticipant = strategy.amount0Desired / participants.length;
        uint256 amount1PerParticipant = strategy.amount1Desired / participants.length;
        
        for (uint256 i = 0; i < participants.length; i++) {
            // Transfer tokens from participants (would need proper approvals)
            // IERC20(strategy.token0).transferFrom(participants[i], address(this), amount0PerParticipant);
            // IERC20(strategy.token1).transferFrom(participants[i], address(this), amount1PerParticipant);
        }
    }
    
    function _provideLiquidityToPool(LiquidityStrategy memory strategy) internal returns (uint256 tokenId) {
        // Interact with Uniswap V3 NonfungiblePositionManager
        // This would use the actual Uniswap contracts
        return 1; // Simplified return
    }
    
    function _setupFeeCollection(bytes32 intentHash, uint256 tokenId, address[] memory participants) internal {
        // Set up automatic fee collection and distribution
        // This would integrate with Uniswap's fee collection mechanism
    }
}
```

## Yield Farming Coordination

### Multi-Protocol Yield Strategy

```solidity
contract YieldFarmingCoordinator {
    struct YieldStrategy {
        address[] protocols;       // [Compound, Aave, Yearn]
        address[] tokens;         // Tokens to farm with
        uint256[] allocations;    // Percentage allocation to each protocol
        uint256 totalAmount;
        uint256 duration;         // Farming duration
        uint256 minYield;         // Minimum expected yield
        bool autoCompound;        // Whether to auto-compound rewards
    }
    
    function proposeYieldStrategy(
        YieldStrategy calldata strategy,
        address[] calldata participants
    ) external returns (bytes32 intentHash) {
        
        bytes memory strategyData = abi.encode(
            strategy.protocols,
            strategy.tokens,
            strategy.allocations,
            strategy.totalAmount,
            strategy.duration,
            strategy.minYield,
            strategy.autoCompound
        );
        
        IAgentCoordinationCore.AgentIntent memory intent = IAgentCoordinationCore.AgentIntent({
            payloadHash: bytes32(0),
            expiry: uint64(block.timestamp + strategy.duration),
            nonce: framework.getAgentNonce(msg.sender) + 1,
            chainId: uint32(block.chainid),
            agentId: msg.sender,
            coordinationType: keccak256("YIELD_FARMING_V1"),
            maxGasCost: 2000000, // Higher for multi-protocol interactions
            priority: 50, // Lower priority for longer-term strategies
            dependencyHash: bytes32(0),
            securityLevel: uint8(AgentSecurityModule.SecurityLevel.STANDARD),
            participants: participants,
            coordinationValue: strategy.minYield
        });
        
        IAgentCoordinationCore.CoordinationPayload memory payload = IAgentCoordinationCore.CoordinationPayload({
            version: keccak256("YIELD_V1"),
            coordinationType: intent.coordinationType,
            coordinationData: strategyData,
            conditionsHash: keccak256(abi.encodePacked("yield:", strategy.minYield)),
            timestamp: block.timestamp,
            metadata: ""
        });
        
        intent.payloadHash = framework.getPayloadHash(payload);
        
        security.createSecurityContext(
            keccak256(abi.encode(intent)),
            AgentSecurityModule.SecurityLevel.STANDARD,
            participants,
            1800 // 30-minute timelock for yield strategies
        );
        
        bytes memory signature = _signIntent(intent);
        return framework.proposeCoordination(intent, signature, payload);
    }
    
    function executeYieldStrategy(
        bytes32 intentHash,
        YieldStrategy calldata strategy
    ) external returns (uint256[] memory depositAmounts) {
        
        _validateExecution(intentHash);
        
        // Collect funds from participants
        (,,,address[] memory participants,) = framework.getCoordinationStatus(intentHash);
        _collectFundsFromParticipants(strategy, participants);
        
        // Deploy funds across protocols according to allocation
        depositAmounts = new uint256[](strategy.protocols.length);
        for (uint256 i = 0; i < strategy.protocols.length; i++) {
            uint256 allocation = strategy.totalAmount * strategy.allocations[i] / 100;
            depositAmounts[i] = _depositToProtocol(strategy.protocols[i], strategy.tokens[0], allocation);
        }
        
        // Set up auto-compound if enabled
        if (strategy.autoCompound) {
            _setupAutoCompound(intentHash, strategy);
        }
        
        return depositAmounts;
    }
    
    function _collectFundsFromParticipants(YieldStrategy memory strategy, address[] memory participants) internal {
        uint256 amountPerParticipant = strategy.totalAmount / participants.length;
        for (uint256 i = 0; i < participants.length; i++) {
            // Transfer funds from participants
            // IERC20(strategy.tokens[0]).transferFrom(participants[i], address(this), amountPerParticipant);
        }
    }
    
    function _depositToProtocol(address protocol, address token, uint256 amount) internal returns (uint256) {
        // Deposit to specific protocol (Compound, Aave, etc.)
        // This would use actual protocol interfaces
        return amount;
    }
    
    function _setupAutoCompound(bytes32 intentHash, YieldStrategy memory strategy) internal {
        // Set up automatic compounding mechanism
        // This would integrate with automation protocols like Gelato or Chainlink Keepers
    }
}
```

## Flash Loan Coordination

### Coordinated Flash Loan Arbitrage

```solidity
contract FlashLoanCoordinator {
    struct FlashLoanStrategy {
        address flashLoanProvider; // Aave, dYdX, etc.
        address asset;
        uint256 amount;
        address[] dexes;          // DEXes to arbitrage between
        address[] tokens;         // Token path for arbitrage
        uint256 minProfit;
        bytes strategyData;       // Complex strategy parameters
    }
    
    function proposeFlashLoanArbitrage(
        FlashLoanStrategy calldata strategy,
        address[] calldata participants
    ) external returns (bytes32 intentHash) {
        
        bytes memory encodedStrategy = abi.encode(
            strategy.flashLoanProvider,
            strategy.asset,
            strategy.amount,
            strategy.dexes,
            strategy.tokens,
            strategy.minProfit,
            strategy.strategyData
        );
        
        // Use ENHANCED security for flash loan operations
        IAgentCoordinationCore.AgentIntent memory intent = IAgentCoordinationCore.AgentIntent({
            payloadHash: bytes32(0),
            expiry: uint64(block.timestamp + 1800), // 30-minute window
            nonce: framework.getAgentNonce(msg.sender) + 1,
            chainId: uint32(block.chainid),
            agentId: msg.sender,
            coordinationType: keccak256("FLASH_LOAN_ARBITRAGE_V1"),
            maxGasCost: 5000000, // Very high for complex flash loan operations
            priority: 255, // Highest priority
            dependencyHash: bytes32(0),
            securityLevel: uint8(AgentSecurityModule.SecurityLevel.ENHANCED),
            participants: participants,
            coordinationValue: strategy.minProfit
        });
        
        // Encrypt flash loan strategy
        (bytes memory encryptedStrategy, bytes memory keyData) = security.encryptCoordinationData(
            encodedStrategy,
            participants,
            AgentSecurityModule.SecurityLevel.ENHANCED
        );
        
        IAgentCoordinationCore.CoordinationPayload memory payload = IAgentCoordinationCore.CoordinationPayload({
            version: keccak256("FLASH_LOAN_V1"),
            coordinationType: intent.coordinationType,
            coordinationData: encryptedStrategy,
            conditionsHash: keccak256(abi.encodePacked("flash_loan:", strategy.asset, strategy.amount)),
            timestamp: block.timestamp,
            metadata: keyData
        });
        
        intent.payloadHash = framework.getPayloadHash(payload);
        
        security.createSecurityContext(
            keccak256(abi.encode(intent)),
            AgentSecurityModule.SecurityLevel.ENHANCED,
            participants,
            1800 // 30-minute timelock
        );
        
        bytes memory signature = _signIntent(intent);
        return framework.proposeCoordination(intent, signature, payload);
    }
    
    function executeFlashLoanArbitrage(
        bytes32 intentHash,
        FlashLoanStrategy calldata strategy
    ) external returns (uint256 profit) {
        
        _validateExecution(intentHash);
        
        // Initiate flash loan
        _initiateFlashLoan(strategy);
        
        return profit;
    }
    
    function _initiateFlashLoan(FlashLoanStrategy memory strategy) internal {
        // Initiate flash loan from provider
        // This would call the actual flash loan provider (Aave, dYdX, etc.)
    }
    
    // Flash loan callback - would be called by flash loan provider
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external returns (bool) {
        
        // Decode strategy from params
        FlashLoanStrategy memory strategy = abi.decode(params, (FlashLoanStrategy));
        
        // Execute arbitrage strategy with flash loan funds
        uint256 profit = _executeArbitrageWithFlashLoan(strategy, amount);
        
        // Ensure we have enough to repay flash loan + premium
        require(profit > premium, "Insufficient profit to repay flash loan");
        
        // Repay flash loan
        // IERC20(asset).approve(msg.sender, amount + premium);
        
        return true;
    }
    
    function _executeArbitrageWithFlashLoan(
        FlashLoanStrategy memory strategy,
        uint256 amount
    ) internal returns (uint256 profit) {
        
        // Execute complex arbitrage strategy using flash loan funds
        uint256 currentAmount = amount;
        
        // Perform swaps across multiple DEXes
        for (uint256 i = 0; i < strategy.dexes.length; i++) {
            currentAmount = _performSwap(strategy.dexes[i], strategy.tokens[i], strategy.tokens[i + 1], currentAmount);
        }
        
        // Calculate profit after all swaps
        profit = currentAmount > amount ? currentAmount - amount : 0;
        
        return profit;
    }
    
    function _performSwap(address dex, address tokenIn, address tokenOut, uint256 amountIn) internal returns (uint256) {
        // Perform swap on specific DEX
        // This would use actual DEX interfaces
        return amountIn * 99 / 100; // Simplified 1% slippage
    }
}
```

## Integration Best Practices

### Security Considerations

1. **Always validate coordination status** before execution
2. **Use appropriate security levels** based on value and risk
3. **Implement proper access controls** for sensitive operations
4. **Include slippage protection** for all trading operations
5. **Set appropriate gas limits** for complex operations

### Gas Optimization

1. **Batch operations** where possible to reduce transaction costs
2. **Use efficient data structures** for storing coordination data
3. **Optimize loop iterations** in complex strategies
4. **Cache frequently accessed values** to reduce storage reads

### Error Handling

```solidity
function safeExecuteStrategy(bytes32 intentHash) external returns (bool success) {
    try this.executeStrategy(intentHash) returns (uint256 result) {
        emit StrategyExecuted(intentHash, result);
        return true;
    } catch Error(string memory reason) {
        emit StrategyFailed(intentHash, reason);
        _handleStrategyFailure(intentHash, reason);
        return false;
    } catch {
        emit StrategyFailed(intentHash, "Unknown error");
        _handleStrategyFailure(intentHash, "Unknown error");
        return false;
    }
}
```

### Monitoring and Analytics

```solidity
contract CoordinationAnalytics {
    struct CoordinationMetrics {
        uint256 totalValue;
        uint256 successRate;
        uint256 averageGasUsed;
        uint256 averageProfit;
        uint256 participantCount;
    }
    
    mapping(bytes32 => CoordinationMetrics) public metrics;
    
    function updateMetrics(bytes32 coordinationType, uint256 value, bool success, uint256 gasUsed) external {
        CoordinationMetrics storage metric = metrics[coordinationType];
        metric.totalValue += value;
        if (success) {
            metric.successRate = (metric.successRate + 100) / 2; // Simple moving average
        }
        metric.averageGasUsed = (metric.averageGasUsed + gasUsed) / 2;
    }
}
```

These integration examples demonstrate the versatility and power of the Agent Coordination Framework across various DeFi and MEV scenarios. Each example can be extended and customized based on specific requirements and market conditions.