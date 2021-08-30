# My ported DApp to Polyjuice

## Contract
I ported an existing ethereum DApp that can be used to make donations in the future to an address.
The solidity contract used was Trust.sol.

```solidity
pragma solidity=0.8.3;

contract Trust {
    
    struct Kid {
        uint amount;
        uint maturity;
        bool paid;
        
    }
    mapping(address => Kid) public kids;
    
    
    address public admin;

    constructor(){
        admin = msg.sender;
    }
    
    
    function addKid(address kid, uint timeToMaturity) external payable {
        require(msg.sender == admin, 'only admin');
        require(kids[kid].amount == 0, 'kid already exists');
        kids[kid] = Kid(msg.value, block.timestamp + timeToMaturity,false);

        
    }
    
    function withdraw() external returns (uint) { 
        Kid storage kid = kids[msg.sender];
        require(kid.maturity <= block.timestamp, 'too early');
        require(kid.amount > 0, 'amount is 0'); 
        require(kid.paid == false, 'paid already');
        kid.paid = true;
        payable(msg.sender).transfer(kid.amount);
            return kid.amount;
        
        
    }
    
    
}

```

This simple contract only has two functions which are: payable addKid which takes an address and integer and sets the amount of tokens which are added to a mapping and locked in time, this data is used in withdraw function that can be used by the address in the mapping to unlock the donation later some time in the future. This contract can be extended to meet more requirements for the unlocked amount.


## Web3 Provider changes
We used ReactJs for the front-end. To port to Polyjuice is just needed a few changes on the front-end of original ethereum DApp. Like the web3 creation here where we set the polyjuice provider. In /src/ui/app.tsx.

The createWeb3 function was changed from this:
```javascript
async function createWeb3() {                                                             
    // Modern dapp browsers...                                                            
    if ((window as any).ethereum) {                                                       
        const web3 = new Web3((window as any).ethereum);                                  

        try {                                                                             
            // Request account access if needed                                           
            await (window as any).ethereum.enable();                                      
        } catch (error) {                                                                 
            // User denied account access...                                              
        }                                                                                 

        return web3;                                                                      
    }                                                                                     

    console.log('Non-Ethereum browser detected. You should consider trying MetaMask!');   
    return null;                                                                          
} 

```

to:
```javascript
async function createWeb3() {                                                             
    // Modern dapp browsers...                                                            
    if ((window as any).ethereum) {                                                       
        const godwokenRpcUrl = CONFIG.WEB3_PROVIDER_URL;                                  
        const providerConfig = {                                                          
            rollupTypeHash: CONFIG.ROLLUP_TYPE_HASH,                                      
            ethAccountLockCodeHash: CONFIG.ETH_ACCOUNT_LOCK_CODE_HASH,                    
            web3Url: godwokenRpcUrl                                                       
        };                                                                                

        const provider = new PolyjuiceHttpProvider(godwokenRpcUrl, providerConfig);       
        const web3 = new Web3(provider || Web3.givenProvider);                            

        try {                                                                             
            // Request account access if needed                                           
            await (window as any).ethereum.enable();                                      
        } catch (error) {                                                                 
            // User denied account access...                                              
        }                                                                                 

        return web3;                                                                      
    }                                                                                     

    console.log('Non-Ethereum browser detected. You should consider trying MetaMask!');   
    return null;                                                                          
}      

```

To set the provider we created the following object in /src/config.ts and imported in our /src/ui/app.tsx, also we need to import the PolyjuiceHttpProvider web3 implementation.

```javascript
export const CONFIG = {
    WEB3_PROVIDER_URL: 'https://godwoken-testnet-web3-rpc.ckbapp.dev',
    ROLLUP_TYPE_HASH: '0x4cc2e6526204ae6a2e8fcf12f7ad472f41a1606d5b9624beebd215d780809f6a',
    ETH_ACCOUNT_LOCK_CODE_HASH: '0xdeec13a7b8e100579541384ccaf4b5223733e4a5483c3aec95ddc4c1d5ea5b22'
};
```

```
import { CONFIG } from '../config';      
import { PolyjuiceHttpProvider } from '@polyjuice-provider/web3';
```
And we added a effect where we set the polyjuice address got from the Address Translator which is imported as well.

 ```javascript

    useEffect(() => {                                                                                           
        if (accounts?.[0]) {                                                                                    
            const addressTranslator = new AddressTranslator();                                                  
            setPolyjuiceAddress(addressTranslator.ethAddressToGodwokenShortAddress(accounts?.[0]));             
        } else {                                                                                                
            setPolyjuiceAddress(undefined);                                                                     
        }                                                                                                       
    }, [accounts?.[0]]);
```
           
```javascript
import { AddressTranslator } from 'nervos-godwoken-integration';      
```

## Contract Wrapper changes
In the contract calls wrapper in /src/lib/contracts/TrustWrapper.ts is important to send default options.

```javascript
const DEFAULT_SEND_OPTIONS = {
    gas: 6000000
};    
```

and include them in the transaction .send({....}) in those used in the wrapper.

```javascript

    async withdrawFunds(fromAddress: string) {                                                                                                                                    
        const tx = await this.contract.methods.withdraw().send({                                                                                                                  
                ...DEFAULT_SEND_OPTIONS,                                                                                                                                          
                from: fromAddress                                                                                                                                                 
        });                                                                                                                                                                       
        return tx.transactionHash;                                                                                                                                                
    }                                   

    async addTheKid(value: number, kid: string, timeInFuture: number, fromAddress: string) {
        const tx = await this.contract.methods.addKid(kid,timeInFuture).send({
            ...DEFAULT_SEND_OPTIONS,
            from: fromAddress,
            value
        });

        return tx.transactionHash;
    }

    async deploy(fromAddress: string) {
        const deployTx = await (this.contract
            .deploy({
                data: TrustJSON.bytecode,
                arguments: []
            })
            .send({`
                ...DEFAULT_SEND_OPTIONS,
                from: fromAddress,
                to: '0x0000000000000000000000000000000000000000'
            } as any) as any);

        this.useDeployed(deployTx.contractAddress);

        return deployTx.transactionHash;
    }

```
Also when sending the transactions we should take into account that the amount of CKB to send is 10^8 shannons.

## Test

And the Dapp runs smoothly in port 3000 running the following command.
```
yarn && yarn build && yarn ui
```


Here we deploy the contract.

<img src=./task7_1_deployed.png >

Then we can call the addkid function to add a donation to my own address.

<img src=./task7_2_signing_AddKid_tx.png >

Then we see the transaction has been processed and our balance has changed.

<img src=./task7_3_afterAddKid.png >

Later we withdraw the amount.

<img src=./task7_4_afterWithdraw.png >

This polyjuice DApp can be found in https://github.com/scentlxss/future_donations_ported_dapp
and original ethereum DApp in https://github.com/scentlxss/ethereum_dapp_example .
