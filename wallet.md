# 钱包概念

在区块链中，我们的数字资产都会对应到一个账户地址上，只有拥有账户的钥匙（私钥）才可以对资产进行消费（用私钥对消费交易签名）。

私钥和地址的关系如下：

<img src="BlockChain.assets/image-20211230144312667.png" alt="image-20211230144312667" style="zoom: 50%;" />

### 创建账号

创建账号关键是生成一个私钥， 私钥是一个32个字节的数， **生成一个私钥在本质上在1到2^256之间选一个数字**。
因此生成密钥的第一步也是最重要的一步，是要找到足够安全的熵源，即随机性来源，只要选取的结果是不可预测或不可重复的，那么选取数字的具体方法并不重要。比如可以掷硬币256次，用纸和笔记录正反面并转换为0和1，随机得到的256位二进制数字可作为钱包的私钥。

### BIP39

BIP32 提案可以让我们保存一个随机数种子（通常16进制数表示），而不是一堆秘钥，确实方便一些，不过用户使用起来也比较繁琐，这就出现了BIP39，它是使用助记词的方式，生成种子的，这样用户只需要记住12（或24）个单词，单词序列通过 PBKDF2 与 HMAC-SHA512 函数创建出随机种子作为 BIP32 的种子。

助记词生成的过程是这样的：先生成一个128位随机数，再加上对随机数做的校验4位，得到132位的一个数，然后按每11位做切分，这样就有了12个二进制数，然后用每个数去查BIP39定义的单词表，这样就得到12个助记词，这个过程图示如下：

<img src="BlockChain.assets/image-20211230144907783.png" alt="image-20211230144907783" style="zoom:50%;" />

### 助记词推导出种子

这个过程使用密钥拉伸（Key stretching）函数，被用来增强弱密钥的安全性，PBKDF2是常用的密钥拉伸算法中的一种。
PBKDF2基本原理是通过一个为随机函数(例如 HMAC 函数)，把助记词明文和盐值作为输入参数，然后重复进行运算最终产生生成一个更长的（512 位）密钥种子。这个种子再构建一个确定性钱包并派生出它的密钥。

密钥拉伸函数需要两个参数：助记词和盐。盐可以提高暴力破解的难度。 盐由常量字符串 "mnemonic" 及一个可选的密码组成，注意使用不同密码，则拉伸函数在使用同一个助记词的情况下会产生一个不同的种子，这个过程图示图下:

<img src="BlockChain.assets/image-20211230145130227.png" alt="image-20211230145130227" style="zoom:50%;" />



### 示例代码

```go
package main

import (
	"fmt"

	hdwallet "github.com/miguelmota/go-ethereum-hdwallet"
	"github.com/tyler-smith/go-bip39"
)

func main() {
	// generate mnemonic
	mnemonic, err := hdwallet.NewMnemonic(128)
	if err != nil {
		panic(err)
	}
	fmt.Println("mnemonic:", mnemonic)

	// generate seed from mnemonic
	seed := bip39.NewSeed(mnemonic, "")

	// generate from seed
	wallet, err := hdwallet.NewFromSeed(seed)
	if err != nil {
		panic(err)
	}

	// 通过这种（树状结构）推导出来的秘钥，通常用路径来表示，每个级别之间用斜杠 / 来表示，由主私钥衍生出的私钥起始以“m”打头。因此，第一个母密钥生成的子私钥是m/0。第一个公共钥匙是M/0。第一个子密钥的子密钥就是m/0/1，以此类推。
	// BIP44则是为这个路径约定了一个规范的含义(也扩展了对多币种的支持)，BIP0044指定了包含5个预定义树状层级的结构: m/purpose'/coin'/account'/change/address_index
	// m是固定的, purpose也是固定的44, coin代表的是币种，0代表比特币，1代表比特币测试链，60代表以太坊，account代表币的账户索引，从0开始，change常量0用于外部(收款地址)，常量1用于内部（也称为找零地址）。外部用于在钱包外可见的地址（例如，用于接收付款）。内部链用于在钱包外部不可见的地址，用于返回交易变更。 (所以一般使用0)
	path := hdwallet.MustParseDerivationPath("m/44'/60'/0'/0/0") //最后一位是同一个助记词的地址id，从0开始，相同助记词可以生产无限个地址
	account, err := wallet.Derive(path, false)
	if err != nil {
		panic(err)
	}

	address := account.Address.Hex()
	privateKey, _ := wallet.PrivateKeyHex(account)
	publicKey, _ := wallet.PublicKeyHex(account)

	fmt.Println("address0:", address)      // id为0的钱包地址
	fmt.Println("privateKey:", privateKey) // 私钥
	fmt.Println("publicKey:", publicKey)   // 公钥
}

```

