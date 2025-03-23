# Sablier

## [M-01] Tokens symbol containing a double quote breaks the NFT JSON payload

<https://github.com/Cyfrin/2024-05-Sablier/blob/main/v2-core/src/SablierV2NFTDescriptor.sol>

### Impact

The JSON payload of the NFT representing the stream is partially broken

### Proof of concept

The NFT tokenURI is a JSON payload that includes the underlying ERC20 token's symbol in multiple places.

The format of a JSON payload is composed of double quotes (`"`) to represent the keys (`{"key" : ...}`) of the data but also the data itself when it is a string (`{"key" : "I am a string"}`).

Injecting a `"` in the data string can mess up the JSON payload due to an un-equivalent number of double quotes, potentially making it impossible to parse or even worse, changing the payload itself which could lead to honey-pot attacks, phishing attacks...

Since the `SablierV2NFTDescriptor::safeAssetSymbol()` function replaces the ERC20 token symbol in case it is too big, the surface attack is restricted but the fact that the tokenURI can be broken **MUST** be considered.

The below is a proof of concept that demonstrates what an attacker could do by creating an ERC20 token with a malicious symbol.

Modify the symbol of the Mock ERC20 token in `v2-core/test/Base.t.sol` on line 58 like such (pick one or the other) :

```solidity
58: dai = new ERC20Mock("Dai Stablecoin", "DAI\"BROKEN"); // Simply breaks the JSON
58: dai = new ERC20Mock("Dai Stablecoin", "\",\"value\":\"Injected"); // Partially changes the JSON
```

Add the following test to `v2-core/test/integration/concrete/lockup-dynamic/token-uri/tokenURI.t.sol`

```solidity
function test_Broken_TokenURI() external skipOnMismatch givenNFTExists {
    string memory tokenURI = lockupDynamic.tokenURI(defaultStreamId);
    tokenURI = vm.replace({ input: tokenURI, from: "data:application/json;base64,", to: "" });
    string memory actualDecodedTokenURI = string(Base64.decode(tokenURI));
    string memory filename = "broken.json";
    vm.writeFile(filename, actualDecodedTokenURI);
}
```

The test will write the `JSON payload` of the NFT in `v2-core/broken.json` which we can open in our browser (Firefox) and notice it is malformed

```sh
SyntaxError: JSON.parse: expected ',' or '}' after property value in object at line 1 column 48 of the JSON data
```

## Recommended mitigation steps

Modify the `SablierV2NFTDescriptor::safeAssetSymbol()` function to make sure it escapes every special characters in the `token symbol` and `token name`, especially the `\` and `"` characters

## [L-01] Malicious user can honeypot other users to buy their stream on an NFT marketplace and cancel it right before the purchase happens

<https://github.com/Cyfrin/2024-05-Sablier/blob/main/v2-core/src/abstracts/SablierV2Lockup.sol>

### Impact

*note the issue can also occur on NFT lending marketplaces with a few variations but the attack path is similar*

Victims think they are buying a worth stream but they end up with an empty stream.

This results in a loss of funds for the victims, profit for the malicious user and loss of trust in the Sablier protocol.

### Proof of concept

In the majority of NFT marketplaces such as Opensea and Blur, users can list their NFT by approving it to the marketplace contract.

Once the NFT finds a buyer, the marketplace contract transfers the NFT from the owner to the buyer who pays to receive it.

Regarding Sablier, a stream is represented as an NFT and can have the ability to be :

* transfered (by the NFT owner) : the new NFT owner will be able to withdraw streamed (an unclaimed) funds
* canceled (by the stream creator) : the stream creator will be able to withdraw the un-streamed funds, setting the stream in a state where it won't stream funds anymore (the current owner is still able to withdraw the already streamed funds)

Here is a realistic scenario where a malicious user can honeypot other users to steal their funds.

*note that this scenario can be crafted in a more malicious way but for the sake of simplicity, we'll keep it this way*

1. Attacker creates a linear stream that lasts for **5 days**, holds **5,000 USDC** and sets the recipient to himself
2. Attacker lists the stream on Opensea for the equivalent of **1,000 USDC**
3. User sees he can make a **4,000 USDC** profit and buys the stream for **1,000 USDC**
4. Attacker was monitoring the mempool and **frontruns** the user's transaction to **withdraw** the already streamed tokens (he can do it since he is the stream recipient) and **cancel** it to recover the un-streamed tokens (he can do it since he is the stream creator)
5. The marketplace executes the trade, sends the **1,000 USDC** equivalent to the attacker and sends the stream to the user

At the end of the scenario, the attacker basically stole **\$1,000** from the user

### Recommendations

Considering the protocol design, the issue can't be patched easily.

In order to prevent one part of the attack, the stream would need to be `UNCANCELABLE` but this can't be enforced as it is a core functionality.

The second part of the attack can't really be countered because while the NFT is listed on the marketplace, it might still stream funds which can be claimed by NFT owner at any time.

One idea would be to add a requirement on the NFT approval function that checks if the address approved is part of a list of allowed marketplaces. If that is the case, the stream would need to be `canceled` (and maybe have `0 streamed tokens`?)

Adding functions to make the process easier for users would be beneficial.
