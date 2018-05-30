# Lock Contract Tutorial

A lock contract specifies a certain timestamp, before which no one is allowed to withdraw assets from the contract. Once the time stated is reached, the contract owners can then withdraw the assets.

The current time obtained by the contract is the time of the latest block in the blockchain (the error is about 15 seconds). For details, refer to [Blockchain class](../reference/fw/dotnet/neo/Blockchain.md), [Header class](../reference/fw/dotnet/neo/Header.md).

This section describes how to deploy a lock contract in the blockchain, so that it can be called by others. 

In addition, this tutorial is based on the demo of Smart Contract 2.0. Please download the latest **test network client** from GitHub: [Neo GUI ](https://github.com/neo-project/neo-gui/releases)

> [!Note]
> The following operation will run in the **test network**. Because the main network has not yet deployed Smart Contract 2.0, the following operation in the main network will fail.
> In order to use the test net you have to make two changes in the config files:
1. Extract Neo GUI client to your folder. You will notice the files config.json, config.mainnet.json, config.testnet.json, protocol.json, protocol.mainnet.json, protocol.testnet.json. By default, `config.json` and `protocol.json` are identical to the Mainnet versions.
2. You need to copy the code from the testnet files into the `config.json` and `protocol.json` files so that you can access the Testnet rather than the Mainnet (i.e. copy and paste `config.testnet.json` into `config.json`, and `protocol.testnet.json` into `protocol.json`).

## Create a wallet

This step is very basic. Open the PC version of the client, click `wallet`, `create the wallet database `, select the wallet storage location and set the wallet name and password.

![](../../../assets/lock2_1.png)

## Get the public key

The newly created wallet will automatically generate a standard account. Right-click on the account, view the private key, and copy the public key from the second line, as shown in the figure:

![](../../../assets/lock2_2.png)

> [!Caution]
> Please note: Do not divulge your private key.

Here we write a local program to turn the public key into a byte array, C# code is as follows:

```c#
namespace ConsoleApp1
{
    class Program
    {
        static void Main(string[] args)
        {
            // 这里替换为上一步复制的公钥
            byte[] b = HexToBytes("0285eab65f4a0126e4b85b4e5d8b7e303aff7efb360d595f2e3189bb90487ad5aa");
            foreach (var item in b)
            {
                Console.Write($"{item}, ");
            }
            Console.ReadLine();
        }

        static byte[] HexToBytes(string hexString)
        {
            hexString = hexString.Trim();
            byte[] returnBytes = new byte[hexString.Length / 2];
            for (int i = 0; i < returnBytes.Length; i++)
            {
                returnBytes[i] = Convert.ToByte(hexString.Substring(i * 2, 2), 16);
            }
            return returnBytes;
        }
    }
}
```

After running it, the screen will display the byte array created from the public key. Copy this down as we will be using it later.

## Write a smart contract

Create a smart contract project and write the following smart contract. 

```c#
using Neo.SmartContract.Framework;
using Neo.SmartContract.Framework.Services.Neo;

namespace Neo.SmartContract
{
    public class Lock : SmartContract
    {
        public static bool Main(byte[] signature)
        {
            Header header = Blockchain.GetHeader(Blockchain.GetHeight());
            if (header.Timestamp < 1499328600) // 2017-6-6 18:10
                return false;
            // Paste the public key byte array here
            return VerifySignature(signature，new byte[] { 2, 133, 234, 182, 95, 74, 1, 38, 228, 184, 91, 78, 93, 139, 126, 48, 58, 255, 126, 251, 54, 13, 89, 95, 46, 49, 137, 187, 144, 72, 122, 213, 170 });
        }
    }
}
```

The lock contract has two important variables to change: the public key, and the lock time.

1. In the contract code, paste the previous copy of the public key byte array

2. Change the lock time in the sample code, which is a Unix timestamp. Calculate it yourself, you may want to use an online tool. [Unix timestamp online conversion](https://unixtime.51240.com/).

After replacing the two variables, compile the contract to get a Lock.avm file.

## Deploy lock Contract

To deploy the contract, we first need to obtain the contract script. There are many ways to do this. We can utilize the C# code below to read the .avm in order to get the bytecode.

```c#
byte[] bytes = System.IO.File.ReadAllBytes("Test.avm");
string str = System.Text.Encoding.Default.GetString(bytes);
```

If you think writing a script for this is troublesome, the client's `Deploy Contract` function has a simple way to obtain the bytecode:

Click on `Advanced`, `Deploy Contract`, click on the `Load` button on the bottom right corner. Choose the `Lock.avm` file generated earlier. You should see the contract script displayed in the `Code` box, as seen in the figure. Copy this down again.

![](../../../assets/lock2_5.png)

In the client, under the `Account` tab, right click on the whitespace, select `Create Contract Add.`, `Custom`, and paste the contract script into the box:

![](../../../assets/lock2_7.png)


Here, we need to choose an associated account (to be specific, we are associating a pair of public/private keys). The association means that if the smart contract requires a signature operation, the client will use the associated private key to sign. In this step, we have to select the same public key as the first step, otherwise the signature does not match and execution of the contract will fail. Because there is a signature parameter in our contract, fill in 00 in the form of the parameter entry(To understand what to fill in for parameters, refer to [Parameter](../Parameter.md)), and fill in the script code as shown earlier. Once done, we will see the contract address as shown in the figure.

![](../../../assets/lock2_8.png)



## Test

The following is a test of the smart contract authentication account. In transferring assets from a smart contract authentication account, the consensus node will execute the smart contract when verifying the transaction. If the contract validation is successful (the result is true), the transaction is confirmed. Otherwise the transaction will always be unconfirmed. Our testing method will be to first transfer some assets into the account address, then transfer them out.

> [! Note]
> In order to ensure the accuracy of the test, it is best not to have any other assets in the wallet, as you may not know if the assets are coming from a standard address or a contract address, unless you understand the client's change finding algorithm and know which transaction is coming from the contract address.

### Transfer assets to contract address

Open a wallet with assets on **testnet** and transfer a certain amount of assets to the contract account.

### Transfer assets out of contract address

Transfer assets from your smart contract account:

![Transfer contract amount](../../../assets/lock2_11.png)

If the above operation is correct, the following happens when the asset is transferred:

When the current time is less than the lockout time, the transfer will not be confirmed, ie the transfer will fail.

After clicking `Rebuild Index`, after about 5 minutes, the unacknowledged transfer will disappear and the assets will return to the previous state.

If the current time is greater than the lock time, the transfer will be successful.