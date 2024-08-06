
# [L-0] wrong calculation of nuke factor and forge potential

# Impact
the `EntropyGenerator::deriveTokenParameters` function calculates the nuke factor and forge potential wrong
```js
    function deriveTokenParameters(uint256 slotIndex, uint256 numberIndex)
        public
        view
        returns (uint256 nukeFactor, uint256 forgePotential, uint256 performanceFactor, bool isForger)
    {
        uint256 entropy = getEntropy(slotIndex, numberIndex);

>>>        nukeFactor = entropy / 4000000;
>>>        forgePotential = getFirstDigit(entropy);
        performanceFactor = entropy % 10;

        // exmaple logic to determine a boolean property based on entropy
        uint256 role = entropy % 3;
        isForger = role == 0;

        return (nukeFactor, forgePotential, performanceFactor, isForger); // return derived parammeters
    }
```
forge potential is the second digit from right and the nuke factor always returns zero and should be 40k instead of `4000000`


# [L-1] useless variable in the `mintWithBudget` function

# Impact
in the `TraitForgeNft::mintWithBudget` function, the `amountMinted` is just incrementing for each mint process but didn't use anywhere. remove it
```js
  function mintWithBudget(
    bytes32[] calldata proof
  )
    public
    payable
    whenNotPaused
    nonReentrant
    onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
  {
    uint256 mintPrice = calculateMintPrice();
>>>    uint256 amountMinted = 0; // @audit-low useless variable
    uint256 budgetLeft = msg.value;
    // ...
  }
```