1. 大家好，我是比特大陆的姜家志，很高兴这次有机会给大家分享OP_GROUP以及Colored Coin相关的内容，如果有描述的不正确或者不清晰的地方欢迎大家给予指正。这次的分享会以文字的方式为主，主要考虑语音在描述技术问题时对于听众来说不是很容易理解。使用文字的好处是随时都可以向上查找内容，有利于理解。未来也希望有更多的技术人员能给大家分享BCH相关的内容。
	大家也知道BCH社区对于OP_GROUP的提议争议比较大，最近也有些朋友问我OP_GROUP的相关的问题，要解释这个问题是需要花费些时间才能说清楚，我觉得那就不如和大家一起分享下OP_GROUP和彩色币的相关东西，这样大家能够更加清晰的表达自己在彩色币上的观点。OP_GROUP相关的技术spec和实现还在讨论中，这里只是一个相对比较简单的分享。
	下面正式开始今天的分享内容。
	
	2. 这是今天要分享的内容简介，首先我们简单介绍下什么是彩色币，然后介绍下OP_GROUP是用来做什么的，再说下如何通过OP_GROUP实现token。最后在那OP_GROUP和ERC2.0,以及聊聊在BCH上的创新问题。
	3. 先来说说什么是彩色币，彩色币对于比特币来说是一个比较古老的话题了，13年的时候对于彩色币的讨论比较多，13年的时候在BitcoinX出来之后比特币社区就在讨论token这个话题，token在当时在比特币社区的讨论就比较广泛了，但是在比特币社区内这个理念一直都没有得到太好的实现，看看现在ETH上ICO的火爆程度大家就应该明白比特币上当年如果能够发行Token会是件多么重要的事情。当年没有实现在比特币上发token有很多的原因，其中一个就是在技术并没有一个好的技术实现能满足实现染色功能，我印象中当时是通过OP_RETURN和燃烧BTC的方式去实现，这两种方式都无法方便地在BTC上面发行Token,另一个就是BTC上面防尘交易问题，小于546聪的交易会被视为灰尘交易，这样就导致了彩色币最小的限制变成了546聪。现在在BTC上面的彩色币的研究会考虑在闪电网络上实现。
	上面说的是彩色币的一些历史，那彩色币本质上是什么呢？BCH上面实现的彩色币本质上还是BCH，只是会让部分UTXO具有了不同的标记，这些特殊的标记就是染色，染色之后的BCH具有了特殊的意义。而染色币的存储和转移也是在使用BCH的转账系统，保持低手续费以及快速确认对于染色币的使用也是非常有意义的。
	通过让某一些BCH的UTXO有了特殊的含义，那就可以定制化不通的权益证明，比如可以给出一段特殊的标记xxxx,那些被xxxx染色的UTXO就是某个公司的股东，这些地址的拥有者就可以享受到分红。类似的还可以实现token，还可以用在商品证书，产权证明等....
	
	4. 上面简单介绍了什么是彩色币，让大家对于彩色币有一个比较直观的认识，下面开始介绍下OP_GROUP以及如何使用OP_GROUP实现染色。
	
	OP_GROUP的提出是在BCH开发社区有计划地在启用操作码，这样可以让BCH的操作码更加完善，更具有图灵完备性，一个可编程的货币系统也应该具备更强大的功能。启用操作码的时候BU团队提出了OP_GROUP的操作码，现在已经提案到BUIP077了，是由Andrew Stone提出。
	
	简单来说OP_GROUP的作用是在堆栈上面存在的数据做为一个group identifier，同时还能关联input和output，实现group identifier的转移。在转移group identifier的同时必须满足一个条件就是对应的input和output的banlance必须为0(除了铸币和销毁彩色币之外)，具体的铸币、转帐以及销毁后面会详细的介绍。
	
	5. 在详细的介绍OP_GROUP如何实现铸币、转账以及销毁之前。需要简单的介绍下BCH现有的脚本的使用。这样更容易理解后面的OP_GROUP使用，在BCH的脚本系统中，发送的时候给出的output就是锁定脚本，要动用这笔UTXO的时候，就需要使用对应的解锁脚本。在锁定脚本上规定必须使用那个公钥，以及对应的私钥签名，这样才能解锁。而解锁脚本的任务就是给出这两个条件:签名和公钥。在验证的使用把解锁脚本放在前面而锁定脚本放在后面，运行结果通过该交易才是合法的否则就是非法交易，不能打包。
	
	脚本系统中有操作码和数据的区别，完整的脚本会运行在一个栈内，栈：简单理解就是一个特殊的数组。
	操作码是用来操作数据的，
	而数据就是签名，pubkey以及hash值等....
	脚本在执行的时候如果遇到数据就会放入栈内，遇到脚本就会执行对应的操作。
	图中的脚本在执行的时候会变成:
	<sig>, <pubkey>, OP_DUP, OP_HASH160, <pubkeyHash>, OP_EQUAL_VERIFY, OP_CHECKSIG
	
	使用<>的都是数据，而其他的都是操作码，脚本执行顺序安装从左到右执行。
	这个脚本在执行的使用会先把	<sig>, <pubkey>两个数据先放到栈内，遇到第一操作码OP_DUP时(它的含义是复制栈顶元素),就是复制pubkey。
	再执行操作码OP_HASH160，对于<pubkey>做hash160。执行完OP_HASH160之后，就遇到了一个数据<pubkeyHash>,再该数据放到栈顶内，这样栈内就有四个数据:
	<sig>, <pubkey>, <pubkey> (hash160之后的结果), <pubkeyHash>（锁定脚本中的内容）
	再往下执行就遇到了操作码OP_EQUAL_VERIFY,它的意思是对比栈顶两个元素的结果是否相等，简单来说就是对于锁定脚本给出的hash160值，是否和解锁脚本中的pubkey算出来的hash160值相同，如果相同就往下执行，不然就失败。
	OP_EQUAL_VERIFY执行完成之后栈内就剩下了两个数据(<sig>, <pubkey>），而最后一个操作码就是验证交易的签名。
	
	这是整个脚本的运行情况,有可能不是太容易理解,之所以先说下脚本的执行原理，是因为OP_GROUP也是一个操作码，他会在交易的锁定脚本中，大致了解整个脚本的运行原理更容易理解OP_GROUP的运行。
	


6. 这个就是OP_GROUP的使用方式，可以看到前部分的脚本和之前的普通交易的脚本的不同之处，多了两个：（<colored_hash>, OP_GROUP）
<colored_hash>是一段数据，是BCH的地址的hash160的表示，后面紧跟着是OP_GROUP的操作码，在脚本运行的时候，当遇到OP_GROUP操作码的时候，会识别该锁定脚本有token的操作，把前面的group identifier从堆栈内取出，然后再做对应的操作，可能出现的操作是：转账以及销毁染色币。这两个相关的操作会在后面有更加详细的介绍。先说下colored_hash。
7. colored_hash就是group identifier也就是token在BCH系统的标识，简单来说就是使用BCH的地址。现在的建议是地址的20bytes，也可能也会定为20-32bytes之间，这样更加有利于未来的扩展。
在BCH发token的一方需要使用一个地址用来铸币，该地址对于的hash160就成为了group identifier,在整个BCH网络里面可以同过该colored_hash追踪token的去向。地址的格式可以是p2sh,p2pkh都可以实现。当一个人想发行token的时候，首先要有一个用来铸币的地址，然后就可以开始铸币了，如何通过op_group铸币呢？
8. 铸币就是token的发行，将来OP_GROUP上线之后，可以使用有该支持该功能的钱包完成铸币。右边是一个铸币交易的一部分。
  来简单分析下这个交易，以及是如何通过op_group完成铸币的。
	其中的<blue>是铸币者的地址，地址的格式是hash160格式，未来他就是该token在BCH系统内的唯一标识。
	先来看input，BCH上发行的token最小的单位是用聪，要发行token，就必须要有对于的币量（以聪为单位）。也就意味着token能使用的最小单位聪，1聪就可以对应一个token代币。这里假设要发行1000000个token代币。就需要给<blue>地址上发送1000000聪的BCH，从input里面看到从某一个地址上面给<blue>转了1000000聪:
input中对应的utxo脚本:	OP_DUP OP_HASH160 <blue> OP_EQUALVERIFY OP_CHECKSIG

再看output部分，这里直接发token代币给某一个地址<pubkeyHash>，给了该地址发送了1000000聪的BCH，在锁定脚本中对其进行染色:
output中对应的锁定脚本 <blue> OP_GROUP OP_DUP OP_HASH160 <pubkeyHash> OP_EQUALVERIFY OP_CHECKSIG
这样就可以完成了染色token的发行，让<pubkeyHash>的UTXO具有了一个特殊标识，这个标记就是<blue>。

目前来看该output在BCH系统内任意地址都可以发出，如果一个用户在BCH上完成了铸币发行了token，而随便一个用户就能对他要发出的output进行染色，这样铸币的意义就没有了。

所以这里还需要增加一个限制在铸币的条件里，只有拥有对于 <blue>私钥的地址才能对output进行染色成 <blue>，或者是拥有解锁P2SH（多重签名）的脚本的用户才能染色。这样就保证了只有拥有该私钥的用户才能发行对应的token。
这里还有个限制就是交易手续费的问题，铸币的过程是需要发送一笔交易到BCH网络中，交易手续费是需要付的矿工的，不然矿工会有可能不打包。但是铸币的过程最好是保证input和output的平衡，不要从发行的input中产生手续费，这个时候需要手续费时可以使用其他的input来解决。

BCH上可以在发行token的时候建立一个描述性的文档，该文档仅仅是一个声明性的文档。

9. 铸币的描述文档放在OP_RETURN中，以字符串的方式进行描述，但是OP_RETURN中无法放入太多的字符串，比较折中的办法是放入一个URL该URL中会描述token的相关信息，比如谁创建的，创建的目的，限制，合约内容等......最后再附上一个签名

铸币完了之后，拥有彩色币的用户有可能出售彩色币，这就需要看下转账逻辑了

10. 彩色币的转账，简单来说彩色币转账是在复用BCH的转账系统，转账的时候input里面拥有的group identifier，转到想要转出的output上,而且需要要做到他的value在输入和输出中是平衡。该交易需要满足原子交换。从右边的input中里面可以看到<pubkeyHash>地址value是1000000,是被染色过的:
 <blue> OP_GROUP.... 
 对应的output里则可以看到，这笔染色币被转移到了另一个地址，他的锁定脚本中有了:
 <blue> OP_GROUP
 转账时的input和output之间的平衡需要矿工去验证，要保证彩色币不会增发。这里的例子是直接把1000000转移过去。另外当然根据BCH的转账原理，也可以做到给10个地址，每个地址上转100000代币。

 很多的token都存在销毁机制，销毁token也是一个比较大的需要，比如现在的很多的ICO都有在某一个时刻回收token的计划。
 
 11. 销毁彩色币的原理很比较简单，就是把已经染色的交易褪色成正常的交易，从带有标识的UTXO退化成不带有任何标识的UTXO，这样就可以让染色币褪色，从右边的交易中可以看到，input中的 <blue> OP_GROUP ，在output中已经没有了。
     比较标准的销毁染色币的流程是，直接把输入的染色币褪色成正常的utxo，输入和输出的值匹配的，而且还要使用 OP_RETURN <blue>作为记录。
    
    上边就是染色币的铸币、转账以及销毁的流程。很多人会有疑问op_group有没有安全问题，下面是我个人关于OP_GROUP的一些安全边界的看法，可以一起探讨
    
    12. OP_GROUP本质是使用一个特殊的标记对UTXO进行染色，但不会改变和控制UTXO，把OP_GROUP当做一个操作码来看的话，的确和以前的操作码的不一样，之前的操作码操作的都是堆栈中的数据，操作的范围没有超出堆栈，而OP_GROUP却超出了堆栈，但是也只是为了平衡input和output的校验而已.
    它所操作的集合是在现在BCH的允许范围之内，简单来说就是OP_GROUP能做的平衡作用只是现在的BCH的一种特例，并没有超出现有的BCH的转账集合。
    另一个就是OP_GROUP的确会增加一个操作码和一个地址的空间，这对于8M的BCH来说更是件小事。
    
    上述只是从我个人对于OP_GROUP一些安全上的理解，也有可能会有考虑不到的情况，如果大家在安全上有深入的思考欢迎一起交流。
    
    13. OP_GROUP的当前实现如何，这是一个大家比较关心的话题，在实现的影响范围会涉及到全节点以及钱包对应的开发，难度比较大的是op_group的开发，还有对应的RPC命令，几天前这些开发工作还没有完全就绪。如果开发早日就绪，就可以在测试环境上可以进行更多的测试，这样更容易理解OP_GROUP，以及更早的发现潜在的问题。
    
    14. 我们来看看实现OP_GROUP所带来的优点
    （1）能够完全复用BCH的交易系统，不用开发对应的转账
    （2）因为使用的脚本，未来BCH扩展出来的脚本功能在BCH的token上面也可用
    （3）它使UTXO具有了不同的属性，也许这些属性会有人延伸出更多的创新
    （4）SPV钱包是比较容易支持的
    
    上面列举了OP_GROUP的优点，并不代表着OP_GROUP没有缺点，比如上面所说的在增发上OP_GROUP能够限制非私钥拥有者的增发，那如果他自己增发怎么办呢？
    
    15. 第一解决的方式就是使用多重签名，大股东和token发行方一起控制多重签名的地址，这种方式可以防止token发行方单方面的增发，技术上想知道该token有没有增发可以对该地址进行监控。

    16. 第二种是一种比较好的方式，可以使用P2SH的地址替代P2PKH，使用类似于这样的脚本：
    
    OP_CHAINHEIGHT <505500> OP_LESSTHAN OP_VERIFY <pubkey> OP_CHECKSIG
    他的语义是在某一个高度时才会让脚本验证通过，这样就可以保证类似的脚本只能执行一次，对应的token也只能发行一次。目前并没有OP_CHAINHEIGHT这个操作码。希望以后能扩展出类似的操作码。
    
   另外ICO基本都是在ETH上面实现的，采用的是ERC2.0 , 那op_group和ERC2.0对比呢？
   
   17. ERC2.0现在基本上ETH上ICO的标准，ERC2.0是定义的一套发token的标准接口，实现这套接口的智能合约就可以发币了，相对于OP_GROUP来说，ERC2.0更加灵活，可定制化更强，ERC2.0是ETH上面的一个智能合约当然在合约内也就可以解决增发的问题。
   而OP_GROUP的token内含了BCH的价值，无法彻底的把BCH的价值和token的价值拆分，OP_GROUP需要使用聪做为单位也有一定的麻烦，一个BCH只能发1亿个代币，如果要发1000亿个代币就需要使用1000个BCH，对于发币方来说，隐含的成本有点高。
   还就有就是BCH内的灰尘交易的问题，也可以通过含有op_group的交易不再识别为灰尘交易这个方案来解决，但是这样就会难免会有人通过op_group估计发送灰尘交易的问题。
   ERC2.0的确会更加的灵活一些，但是BCH系统本身的低手续费也给彩色币带来了一些优势，在ETH生态内的转账手续费也已经很贵，BCH在低手续费上的确有些优势，如果BCH上能够更快的确认，无疑也能给BCH带来更多的优势。
BCH上的OP_GROUP没有ERC2.0灵活，本质上还有一个就是就是UTXO模式和账户模式的区别。

18. UTXO模式是比特币的一个伟大设计它让比特币在去中心的实现上面更加的容易,UTXO的优势:
(1)UTXO模式无需维护余额，每个地址的余额需要找到自己的UTXO算出来
(2)UTXO都是独立的数据记录，很容易就能做到脱离block单独存储，在验证交易的时候会提高速度
(3)UTXO模式下无需关心事务问题，只需要关心锁定脚本和解锁脚本

而账户是ETH采用的模式，它的好处有:
(1)账户余额容易计算出来，只需要通过查询就可以知道账户的余额
(2)账户更容易记录，包括余额以及账户状态
(3)最重要的是通过账户系统更容易实现图灵完备的智能合约，更容易实现有状态的智能合约

但即使是在BCH上面实现无状态的智能合约也已经可以满足不少需求了，上面举的例子就是一个无状态的智能合约:
OP_CHAINHEIGHT <505500> OP_LESSTHAN .....   
语义上来说就是在505500块的时候我要做什么事情，比如可以在BCH上面实现一个智能合约，猜某一个块的hash的会落在那个概率空间内，谁猜对了就能拿走对应的币，这样实现的智能合约，是不用记录合约账户的状态的，只需要在特定的条件下会执行一次。这样的智能合约也可以做很多创新的事情。

19. 下面是一些我看到的关于OP_GROUP的争论
（1）突破了以前脚本的限制，这个是存在的，但是并没有有安全溢出的问题，而且我个人认为BCH的脚本能够访问自身的状态是一件不错的事情，这样可以出现更多的合约，当然这个是也太容易开发。
（2）运行效果不如ERC2.0，这个我也认可，而且大家在ETH发ICO发的挺好的，突然让他们转到BCH上肯定会有些困难，而且OP_GROUP也没有ERC2.0灵活定制。但是越是这样BCH更应该发展自己，不然未来的差距会更大。
（3）何时上线的问题，这个问题也隐含了如何在BCH上部署上线的问题。如果参考ETH的开发模式，会发现ETH的开发者会集中地每隔一段时间（2-3个月）一起讨论下一步的开发，然后各司其职，当然是V神主导。对于新的功能我觉得提案的团队最起码先要在测试链上运行起来，这样其他人员可以加入测试或者讨论，这样是会比较快一些。

20. 个人支持OP_GROUP尽快上线code reveiew和测试做完的情况下，简单来说就是在安全风险不会外溢的情况下应该支持更多的创新，彩色币仍然存在者一个不小的市场。对于无状态的合约也是，它也是能产生很大的价值的。还有就是开发者应该更快的协议测试，试用，以及做对应的评估。
21. BCH社区应该支持更多的创新，加密货币社区竞争越来越激烈，BCH社区作为一个独立的社区，在实现点对点的现金交易这个远景下，应该做更多的创新。BCH社区应该向BTC社区学习，也要想ETH社区学习，还有其它的加密货币社区学习。BCH社区应该加快进步的步伐，跑的远也要跑的快。
22. 下面是我觉得BCH社区可能需要关注的技术点


