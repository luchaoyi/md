#malware
->基于改进随机森林算法的Android恶意软件检测|杨宏宇
创新点 随机森林投票改进为oob加权投票
->Android恶意软件静态检测模型|杨宏宇
特征属性选择
	每一个Android应用程序必须包含AndroidManifest.xml 文件,Mainfest文件提供应用基本信息
	从Mainfest文件中选择关键特征,如,运行时权限,应用间通信，动作(action)、类型(category)等
静态检测模型
	解包样本安装包，从AndroidManifest.xml提取特征 *
	从恶意样本和良性样本，选择缺失最少的n个特征，良性，恶性分开筛选后形成特征并集，得到特征并集 /使用频率筛特征，特征要保证缺失值少/
	使用信息增益选择特征                                          
	分类模型0|1	

->Maldetect : 基于 Dalvik 指令抽象的 Android 恶意代码检测系统|陈铁明  *
基于特征代码的检测方法
	检测文件是否拥有已知恶意软件的特征代码，快速 、 准确率高
	需要维护病毒特征数据库，法检测未知的恶意代码
基于行为的检测方法
	程序的行为与已知恶意行为模式进行匹配，可实现对未知恶意代码或病毒的检测，误报率高
	动态分析方法 沙盒或虚拟机来模拟运行程序,拦截或监控的方式分析运行时的行为特征,计算资源和时间消耗较大 ,且代码覆盖率低
	静态分析 
		通过逆向工程抽取程序的特征 , 分析函数调用 、 程序指令等序列 ,具有快速高效 、 代码覆盖率高等特点
		大部分的静态分析方法主要从 Android‐Manifes . xml 文件 、 lib 库文件( . so 文件) 、Java 源文件(通过反编译 APK 文件获得)等进行特征的提取	
本文 基于Dalvik 指令序列的静态分析模型
	apktool反汇编，官方Dalvik指令有230条指令,选取107种代表性指令,将指令分为10大类 *
	每类指令选择一个代表符号,将107指令转为10种指令符号,将Dalvik指令序列转换为符号编码序列 * 
	样本指令符号序列编码n-gram特征 
	分类模型
	
->基于Bagging・SVM的Android恶意软件检测模型|谢丽霞
恶意样本种类少且数量有限,导致收集到的数据集不平衡* 
文献[8]提出一种基于多级集成的恶意软件检测模型,提取Dalvik指令、权限和API等静态属性作为特征，基于J48决策树的三层集成分类器。 /不使用集成考虑多分辨率端到端模型/*
本文
	manifest提取Android软件的权限(Permission)、意图(Intent)和组件(Component)等静态属性量化后作为特征
	IG-ReliefF混合筛选算法特征选择
		通过IG和ReliefF算法为特征评分，得到特征子集A      /无监督特征选择/
		通过对分类器交叉验证，根据分类器对特征评分，根据评分对特征子集A做特征选择得到特征子集B		/监督下特征选择/
	bagging bootstrap抽样,构造样本数目相等的恶意数据集和良性数据集合并二者组成平衡数据集* /通过抽样构造多个平衡数据集/	
	svm分类
	
->基于 CNN 和 LSTM 混合的 Android 恶意应用检测 |王聪 
静态反编译 
	通过BakSmali程序进行反编译dex文件来获得多个Smali文件,合并Smali文件,去除具体的操作数部分,只留下操作指令序列，操作指令序列由cnn训练
	指令序列构造一个字典(<256个指令),采用one-hot编码将指令序列转换为一个0-1矩阵 *
动态运行监控某些指标，运行时特征具有时序性，因此使用lstm
cnn+lstm融合使用了全连接层 */去除FC使用reshape的趋势/

->基于多特征和 Stacking 算法的 Android 恶意软件检测方法|盛杰
选取权限、API、SO 库作为特征,运用 Stacking 集成学习算法将多个弱学习器组合

->基于多特征融合的安卓恶意应用程序检测方法|王勇 
静态特征提取 apktool反编译apk *
	从AndroidManifest.xml 文件提取权限和组件特征
	从smali文件正则匹配，提取函数调用和 API调用特征,从API调用序列生成n-gram特征
	文件结构特征 提取apk解包后的文件目录结构特征、
	脚本信息特征 find 命令在样本中查找 js 脚本文,件将找到的文件名作为特征
	字符串信息特征 字符串可能包含网络 http 请求信息和一些关键数据   /恶意程序访问某些网址下载.../
	APK 签名,取签名的 MD5 码或 SHA1 码为特征
动态特征提取 
	virtualbox 上运行安卓系统沙盒,使用 MobSF 工具提取了系统调用特征、关键路径和数据访问特征、http请求特征和恶意吸费特征,linux命令执行特征	*	
	
信息增益率筛选特征
基于Simhash的特征融合*
	LSH 两个文本在原有的数据空间是相似的, 在经过哈希之后他们的相似性仍然存在，simhash是一种LSH，被Google用于海量网页文本去重
	文档的simhash值可保持相似性，是文本的一种降维方法
	步骤 
		文本分词，去除停止词 doc={w0,w1,w2,...}
		每个词wi计算一个权重vi(可使用tf-idf,词频等)
		w0v0
		w1 v1 
		...
		wn vn		                         
		采用任意hash函数映射wi为固定长度0-1串,如100110=hash(wi)，根据0-1串生成等长数值向量,1位置vi,0位置-vi,如100110=hash(wi),wi对应vi,则为wi生成的数值向量为[vi,-vi,-vi,vi,vi,-vi]
		每一个词产生1*n的向量，m个词产生m*n的矩阵,列求和产生1*n的向量
		列求和向量，元素>0则产生1,<0则产生0 ,产生的0-1串为doc的simhash值,如[13, 108, -22, -5, -32, 55]产生110001		
		
->基于非用户操作序列的恶意软件检测方法|罗文琰*
广度优先遍历构建API调用图 *
提取敏感API
提取非用户操作序列
基于编辑聚类度量序列相似性
/方法细致,复杂度高/

->基于灰度图纹理指纹的恶意软件分类|张晨斌 *
2011年 Nataraj等提出了将待分类恶意代码的可执行文件转换为对应的灰度图像,基于图像纹理特征生成特征向量利用KNN算法实现对恶意文件的分类*
灰度图生成
	恶意软件以8bit生成一个[0-255]的像素值，以固定宽度生成一行,本文以128bits为一行，即图片一行128/8=16个像素,末尾，补齐填充0
灰度图纹理特征的提取 *
caffe构建分类器，本文使用了多个分类器，包括cnn,cnn表现不错
参考文献*

->基于机器学习的勒索软件检测方法|项子豪
特征 基于动态提取，沙箱运行后提取特征
	Windows API调用,实验环境基于windows7
	注册表项操作,例如读、写、新建和删除等
	文件和目录的操作
	样本中的字符串
then PCA+BP

->基于卷积神经网络的Android恶意应用检测方法|郗桐
提取操作码，按类别划分; 相比于API调用序列,Opcode操作码可以从底层解释应用程序的行为
使用嵌入层将操作码序列转为矩阵 -) cnn *
实验 
	操作码分类粒度，6类，24类，256类;粒度小准确率提升
	cnn层数 2层和3层效果接近，但2层时间更优

->基于决策树的Android恶意应用检测方法的研究|李秀
提取权限作为特征
使用Apriori挖掘恶意样本和正常样本，得到频繁项集特征 /通过挖掘频繁项筛选特征/*

->基于纹理指纹与活动向量空间的Android恶意代码检测|罗世奇
实验数据*
特征 灰度图+权限融合特征 /DL特征融合常用方法 ，向量累加，向量连接/
分类器 SAE

->基于语义特征的恶意代码检测综述|戴超*
->机器学习安全性问题及其防御技术研究综述|李盼
对机器学习系统的攻击
GAN 对抗样本 *
->恶意代码反分析与分析综述*
->一种改进贝叶斯模型的 Android 恶意软件流量特征分析技术|吴非
以app运行后为特征，
笔者发现恶意软件普遍是周期性的接收和发送数据

->一种混合的Android恶意应用检测方法|姜海涛*
混合 
	静态检测方法和动态检测方法结合 
	权限特征、敏感API调用 and 运行时采集系统调用序列完成特征抽取,获得系统调用依赖关系
系统调用图表示依赖关系，每个节点是一个API,定义系统调用距离，将调用图转为表示调用距离的数值矩阵* /调用图的量化/

->一种恶意代码汇编级行为分析方法|戴超
限长空位字符串匹配*
	模式串可以包含限定长度的空位,这些空位可以匹配限定长度范围内的任意字符
	空位在限定范围内相当于通配符
将操作码建模为字符表,操作码匹配问题则可以转化为为字符串匹配问题,汇编级指令分析可以建模为限长空位字符串匹配

->像素归一化方法在恶意代码可视分析中的应用|任卓君
图像分析在恶意代码分析的发展*

/->恶意代码检测综述|逯超毅
静态分析 
	指令序列,敏感API序列特征        /NLP/
		可使用rnn处理
		提取n-gram特征后使用分类器
		频率特征 
		构造流图，图结构可使用图挖掘算法处理
	指令二进制编码为灰度图,指令序列词嵌入映射为矩阵可使用cnn处理	 /CV/
	权限组合特征使用分类器处理
动态分析
	沙箱内运行，监控文件读写，网络连接，产生的网络流量，短信发送，电话拨打等行为
	动态执行缺点 不稳定，不同执行环境收集的某些信息可能不同
/

->基于静态特征的 Android 恶意代码检测|贾蕴哲
静态分析有效率高、不受环境和特定执行过程限制的优点
特征 权限和敏感函数调用;系统权限（72 维）和敏感 API（67 维）
	Android Manifest.xml 获取app权限信息
	invoke-static”“invoke-direct”“invoke-super”三个基本指令，分别对应调用静态函数、调用一般函数以及调用父函数
分类器 MLP

->基于纹理特征的恶意代码检测方法测试|汪应龙
映射代码为灰度图，使用Gabor(小波)滤波器提取图像特征，新样本检测通过匹配纹理数据库 /Gabor提取特征，KNN检测,改进可使用小波神经网络/

->安卓恶意软件的静态检测方法|陈红闵
Android安全机制
	权限机制
	签名机制 数字签名建立作者和app之间的信任关系

->基于多特征信息增益的安卓恶意软件检测|周梦婷
将apk按不同功能类别归类分别提取数据，生成不同的特征集合，then 求并集*,信息增益选择特征

#arxiv
->A survey of malware behavior description and analysis|Bo YU
静态方法 提取API调用,动态方法可以提取 API调用+参数+返回值+环境变量等信息
受污染分析技术 比较运行前系统状态和运行后系统状态
恶意软件指令集  is a new representation of system calls with input and output arguments*
一些用于描述恶意软件行为的平台无关语言被提出
可视化技术 
	相似性图，序列图，和树图
行为数据 
	level 1 API调用，函数参数，环境变量等 
	level 2 控制流，数据流
	level 3 行为属性，系统资源

对抗学习 
	通过攻击机器学习算法，特征空间，训练和测试数据，对抗机器学习可以影响基于机器学习的方法的有效性
	恶意攻击者可以小心操纵测试数据，以利用学习算法的特定漏洞
		
->一种基于程序切片的 Android 恶意代码检测技术
动态执行缺点 不稳定，不同执行环境收集的某些信息可能不同*
程序切片 指定函数及其参数 、 常量字符串的分析，分析否执行了某种行为

->一种基于支持向量机的安卓恶意软件新型检测方法|张超钦*
使用马尔科夫链建模系统调用序列
	Android4. 0. 4 196 个系统调用，每一个系统调用作为一个状态
	每个软件由strace提取系统调用系列,从序列计算转移概率矩阵,pij=cij/di，i转移到j的概率为，序列中系统调用ij相邻数量/系统i出现的次数,即i和j相邻出现的次数/i出现的次数
	概率转移矩阵reshape为一维作为特征向量
	
->Survey of Machine Learning Techniques for Malware Analysis|Daniele Ucci *
for windows PE 文件，PE特征分为八种类型：字节序列，API /系统调用，操作码，网络，文件系统，CPU寄存器，PE文件特征和字符串
任务 恶意软件识别(样本是否是恶意的)，归类，相似性分析
基于自身 静态 ;基于环境/运行 动态
挑战 
	反分析技术 
		恶意软件可以很容易地理解它是否在沙箱中运行
		PE文件的加壳等
	操作集
		特征筛选，影响准确性	
	数据集 *
		老化，过时问题 
		数据不平衡，良性样本多余恶性样本
		样本集过小
研究趋势
	恶意软件归属/溯源，归类/
	分类，相似度分析，新恶意软件归类到相似样本类别，更容易利用已有知识，分析，防御
	变体预测，模拟恶意软件家族进化中的模式并进行预测未来的变种* /生成模型,进化模型/
	其它特征 
		内存访问
		函数长度 函数中包含的字节数
		抛出异常
		
->Measuring the Effectiveness of Generic Malware Models|Naman Bagga*
本文讨论
	特征 操作码序列和n-gram
	准确性与恶意软件类型数量 
		/恶意软件定义几个类别,感觉是个聚类问题，all一个类别则为二分类，良性or恶性/
		为每一类恶意软件建立模型，精度高但很复杂，为所有恶意软件建立一个模型，通用可能导致无用* /多类vs二类/
实验 
	特征 2,3,,4-gram 
	The techniques used are — SVM, 卡方 test, k-NN, and random forests
	特征选择 恶意和良性 top频率n字组合得的并集
	k=5交叉验证
	模型评价标准 Balanced Accuracy*,使用平衡准确性背后的基本原理是处理恶意软件和良性集中样本数量的不平衡*

结论 
	合并恶意软件种类，更通用的模型，准确度下降
	对于通用模型rf表现最好 /为数据拟合一个参数的svm表现不好，将空间切分为不同区域的rf最好，局部响应的体现!/*
	
->A Review on The Use of Deep Learning in Android Malware Detection|Abdelmonim Naway
F. Martinelli et al. [7]  恶意软件分类，和文本情绪分析建立联系，使用nlp情感分析方法分析恶意软件* /x通用nlp迁移模型是否能在恶意软件分类上用到x/
挑战
	缺乏标准数据集
	深度模型可解释性
	深度学习安全性问题*	
未来方向
	静态分析 
		结构分析分为基于文本，控制流图（CFG），调用图（CG）和程序间控制流图（ICFG）。
		敏感性分析分为流敏感，上下文敏感和路径敏感
	动态分析 
		系统调用，网络连接，功耗，用户等特性交互和内存利用
		跟踪java方法调用的应用程序级别;内核级别收集系统调用使用内核模块;虚拟机拦截模拟器中发生的事件
	针对对抗性攻击的深度学习模型*
		▪蒸馏:
			假设已有神经网络F可以将训练集x分类为目标类y并生成概率分布作为输出,该输出用作标记以训练第二模型F'	
		▪再培训：使用对抗生成的实例重新训练分类器。 再培训的目的是修改模型的抽象。 在选择适当数量的对抗训练实例是一个挑战
		
->Deep-Net: Deep Neural Network for Cyber Security Use Cases|Vinayakumar R*
本文证明了深度学习算法在三个task上的优越性
dnn vs xgboost，在Android恶意软件分类为0.940和0.741，事件检测为1.00和0.997,欺诈检测分别为0.972和0.916
数据集*

->MALWARE DETECTION TECHNIQUES FOR MOBILE DEVICES|Belal Amro *
背景，现状 Ios and Android

->IoT Security Techniques Based on Machine Learning|Liang Xiao
IOT安全
	身份验证
	访问控制
	Secure IoT offloading
	恶意软件检测 
机器学习安全检测应用在IOT系统
	IOT设备 带宽，内存，算力电量有限
	ML算法可能工作不稳定，需要备份策略
	
->Behavioral Malware Classification using Convolutional Recurrent Neural Networks|Bander Alsulami
Behavioral 运行时的行为作为特征  such as 文件系统活动，内存分配，网络通信和系统在执行程序期间调用
Microsoft Windows Prefetch Files
	预取文件是windows操作系统为加速app启动而生成的，包含了exe执行时行为的一个摘要 
	WCM监控程序启动后一段时间的活动，生成记录文件，包含存储信息，加载资源，访问目录等信息
本文模型使用从Windows Prefetch文件夹中存储的Prefetch文件收集的信息	
	l1 输入是exe资源文件名列表，嵌入层转换为矩阵
	l2 卷积，池化提取局部特征，特征进一步抽象
	l3 双向lstm提取特征直接的序列关系
	l4 全局池化+softmax分类
	
->An investigation of a deep learning based malware detection system|Mohit Sewak
Related work on Malware Detection*
本文 
	data 使用objdump生成 .asm
	生成操作码词典，然后生成操作码频率向量作为特征向量
	使用AE提取特征，降维
	模型 FC DNN
->Comparison of Deep Learning and the Classical Machine Learning Algorithm for the Malware Detection|Mohit Sewak
dnn vs rf in malware detection
Related Work*

->Malware Dynamic Analysis Evasion Techniques:A Survey|Amir Afianian /动态分析逃避技术
恶意软件分析
	反汇编，反编译
	手动动态分析 在调试器的帮助下进行
	自动动态分析 Sandbox
	
反debug 
	搜索调试器存在的痕迹,搜索断点
	检测到调试器后，通过多线程等隐藏恶意代码或阻碍调试器的工作
	分析利用特定调试器的漏洞
	无文件恶意软件，完全驻留在内存中增加分析难度
	
沙箱检测规避
	分析环境和痕迹来确定是否处于沙箱
	反图灵测试 检测人机互动等行为来确定是否处于sandbox
	特定目标检测，检测是否处于特定条件下或发现特定目标，若条件满足才表现恶意性
	规避
		延迟执行，睡眠
		触发式执行
		
->Examining Adversarial Learning against Graph-based IoT Malware Detection Systems|Ahmed Abusnaina
提取应用的CFG(Control FlowGraph)，利用图论方法分析物联网恶意软件
从CFG提取23个特征 有中介中心性，亲密度中心性，度中心性，最短路径，密度，边缘数和节点数等

->Machine Learning Aided Static Malware Analysis: A Survey and Tutorial|Andrii Shalaginov*
32bit PE静态分析
Fig. 1 timeline * 
Fig. 5 

->DeepOrigin: End-to-End Deep Learning for Detection of New Malware Families|Ilay Cordonsky
transfer learning*
1>使用一个和目标无关的恶意软件数据集训练一个分类模型后去掉softmax层，作为一个特征提取器，特征提取器输入样本可生成一个样本的低维表示
2>使用特征提取器获取目标数据集低维表示，然后训练新的分类器

->Emulating malware authors for proactive protection using GANs over a distributed image visualization of dynamic file behavior|Vineeth S. Bhaskara
/使用GAN模拟恶意软件作者的主动保护/	
局部表示 
	不同可执行文件共享相同的API子集，但整体行为是不同的[API序列是不同的],单个API特征分布到像素上，则不同的API序列构成不同的纹理*,单个ngram字代表一个像素位置，这些字的tf-idfs作为它们的强度，这种图像表示是局部的，像素仅对应于单个API调用ngram,tf-idf特征是稀疏的，可视化大部分为位置是黑的	
	
本文分布式图像表示*
	API调用ngram编码到频率空间,three channels对应4-grams, 3-grams, and 1,2-grams
	通过对对应通道的逆DFT（IDFT）get分布式图像表示
	分布式表示的每个通道的API信息分布在图像上，形成纹理，而且跨通道的信息是混合的	
	
训练wgan-gp模型，训练后生成新图片代表新的恶意样本
Radford[23]*** 
	噪声向量的输入空间以微妙的方式对最终生成的图像的语义信息进行建模，使得能够利用输入的简单向量算法来操纵所生成的图像
	G(~z man,glasses − ~z man,no glasses + ~z woman,no glasses)resembled that of a woman with glasses 
	生成的图像类似 woman with glasses
	G(~z man,glasses) − G(~z man,no glasses) + G(~z woman,no glasses)
	
	
#对基于AI算法检测器攻击和反攻击的研究
->Exploring Adversarial Examples in Malware Detection|Octavian Suciu*
本文基于可执行文件（PE）文件的原始字节值，每个字节被转换为8维的嵌入向量

对抗样本-构造可以逃避检测的样本
	与图像逃避检测不同，改变PE文件原始字节的攻击必须保持语法的语法和语义保真度
	二进制文件的严格语义不允许输入空间中的任意扰动。这是因为相邻字节之间存在结构相互依赖性，并且对字节值的任何更改都可能会破坏可执行文件的功能
	Kolosnjaji et.2018 在二进制的末尾附加对抗性噪声来避免这个问题,此类攻击利用了MalConv架构中存在的与位置无关的特征检测器中的某些漏洞 
	/末尾附加其它良性软件特征抵消或压倒恶意软件特征来逃避检测，利用了cnn没有存储位置信息的特性,即不具有时序性/
	/时序性来源 rnn或transformer位置向量/*

Random Append攻击都失败了malconv不受随机噪声影响	
Slack FGM
	允许攻击算法在不破坏PE的情况下自由修改现有二进制文件中的字节
	虚拟地址与磁盘上块大小的乘数之间的不对齐，编译器会插入间隙,解析可执行部分标头来提取可执行文件的相邻PE部分之间的间隙
	所有间隙的索引组成Slack区域，执行FGM后更新Slack区域的字节	


->Adversarial Malware Binaries: Evading Deep Learning for Malware Detection in Executables|Bojan Kolosnjaji*
基于梯度的攻击，生成填充字节，填充字节可填充到任意不影响可执行功能的位置来回避检测,本文实验在恶意软件样本末尾填充字节

1 byte is 0-255 ,嵌入矩阵M为256x8,用8维向量表示一个1byte字符
随机初始填充字节xj,xj经过嵌入矩阵M得到向量zj,(注 训练填充字节时嵌入矩阵M是固定的已有的)
f(x) is MalConv ,min f(x),为zj计算一个负梯度方向wj
假设将zj沿方向wj移动，使它最大程度接近每一个Mi,可最接近的Mi对应1byte 数字i，将xj设置为i，完成对填充字节xj的一次迭代

结论
	攻击逃避率和训练时间和填充字节数正相关
	逃避率60%
	在原始字节上训练的基于深度学习的恶意软件检测器存在严重漏洞
	空间不变性 生成的注入字节在任意位置都是有效的

Deceiving End-to-End Deep Learning Malware Detectors using Adversarial Examples|Felix Kreuk
本文和上文Bojan Kolosnjaji类似
不同点 
	损失函数=对抗loss+zj到all Mi的距离loss
	只有对抗loss的实验中发现有时绕动的zj已经和嵌入矩阵的all Mi失去了相似性,因此增加zj到all Mi的距离loss使生成zi更新更接近all Mi


#Open Set Recognition
->Learning a Neural-network-based Representation for Open Set Recognition *
开放集识别问题*
	例如，新的恶意软件类会定期出现; 因此，恶意软件分类系统除了区分已知类之外，还需要识别来自未知类的实例类
	系统需要适当地处理来自新颖/未知类别的实例，对已知类别正确分类，并识别未知类别
构造一个神经网络训练一个低维表示z
	g(x)->z->ii-loss
	ii-loss定义 
		得到的z空间需要满足类内接近(min)，类间离散(max is -min)
		在z空间计算类中心u,类内距离=所有样本到各类的中心的平均距离，类间聚类=类中心最近的两个类的距离
		ii-loss=min(类内距离-类间距离)
	g(x)将x投影到z空间尽可能接近类的平均值,因此离最近的类平均值越远，实例就越可能是异常值,在z空间计算样本的outlier score
阈值估计 
	全局估计 假设异常值百分比为1%,对all样本outlier score降序排序，则选择1%位置的outlier score为阈值
	局部估计 每个类估计一个阈值
	we observe that global threshold(更优) consistently gives more accurate results than per-class threshold
预测 
	预测label,z空间中阈值内最接近的类,阈值外k+1类
	预测概率分布，使用指数函数+归一化转换距离为概率分布
	
Using Randomness to Improve Robustness of Machine-Learning Models Against Evasion Attacks ∗|Fan Yang
引入随机性在训练和预测过程，增加不确定性为减少攻击者的信息量使ml模型更具有鲁棒性
RELATED WORK*
	
训练 构建一个模型池，包含多个模型
预测 随机选择一个模型子集，预测

#数据集
->Microsoft Malware Classification Challenge|https://www.kaggle.com/c/malware-classification
20个字符的散列值唯一标识一个恶意软件,带类标签整数label标识类别,共9个类别恶意软件
每个文件 原始数据文件二进制内容的十六进制表示，以及使用IDA反汇编工具从二进制文件中提取的函数调用，字符串等

->EMBER: An Open Dataset for Training Static PE Malware Machine Learning Models|https://github.com/endgameinc/ember
win PE 1.1M (M is million百万个),训练样本（300K恶意，300K良性，300K未标记）和 200K测试样品（100K恶意，100K良性)
发布了从其它二进制文件中提取特征的src，用于扩充数据集
标记挑战 图像，文本和语音可以相对快速地标记，恶意软件标记需要更加专业的人耗费一定的时间

数据集特征是从原始二进制文件中直接提取出来的
	sha256 可允许研究人员从VirusShare或VirusTotal等获取原始二进制文件
	特征hash 可转换字符串为数值向量
		#Feature Hashing for Large Scale Multitask Learning.
		#将M维特征转换为固定N长度的向量,对样本X
		def hashing_vectorizer(Xi,N):
			Xti=[0,...,0] #转换后向量Xti
			for Xi in X:
				h=hash(Xi) #对第i个特征Xi hash
				idx=h%N #hash值当作索引
				Xti[idx]+=1 
			return Xti
			

##生成模型
/
nn哲学:不容易计算的使用nn拟合
/
#gan
->生成式对抗网络研究进展|王万良
模型
	GAN
		logstic回归
			p(y=1|x)=h_theta(x)
			l=E{yi*log h_theta(xi) + (1-yi)*log(1-h_theta(xi)))}=E{[log h_theta(xi)|yi=1] + [log(1-h_theta(xj)|yj=0]},学习参数theta使h_theta(xj|yj=0)=0,h_theta(xi|yi=1)=1

真实样本x(来自生成模型G需要模仿的真实数据集),生成样本G(z),z是随机噪声
	判别模型D学习参数d时生成模型G的参数g固定不动,使D(x)->1,D(G(z))->0; 固定G的参数下最大化 log(D(x)+log(1-D(G(z))
	生成模型G学习参数g时固定判别模型D参数d不动,使D(G(z))->1,即欺骗D; 固定D的参数下不同最大化log(D(G(z))) 
	特点
		使用D的对抗来使生成器生成更加接近真实的样本，如果去掉D，直接使用样本x监督G则G成为了一个自编码器		
		判断器D相当于生成器的损失函数，对于各种场景不需要针对性设计损失函数，而是对抗学习一个损失函数
		训练不稳定，G生成不好的样本，但D给出了错误的判断，导致G更加表现差		
		
生成式模型 能够反映数据内在概率分布规律并生成全新数据的模型
生成器和判别器可以是任意可微函数
文献[22]提出,通过最大化 logD(G(z))而不是最小化log(1−D(G(z))来训练 G 是一个更好的策略
实际应用中,判别器 D 的参数更新 k 次后,生成器 G 的参数才更新一次	

DCGAN
	CNN的池化层并不可微，DCGAN用步进卷积网络(strided convolution)及其转置结构分别实现D和G	
	生成器中的全连接层用反卷积代替
	
条件生成式对抗网络 将条件变量 y 作为模型的附加信息以约束生成过程
	cGAN 同时对 z 和条件变量 y 进行采样
		直接从训练数据中获取条件变量
		在训练过程中基于训练样本的条件变量值构造核密度估计,对条件变量进行随机采样
	LAPGAN [45] 和 GRAN [32]  条件变量是上一级所生成的图片,前一步得到的生成结果进行训练
	InfoGAN		
		InfoGAN的生成器有两个输入随着噪声z和隐含编码c, 输出为G(z,c),	InfoGAN的目标函数为原始 GAN 的目标函数加上条件变量与生成样本间的互信息(最大化)
		InfoGAN的判断器要鉴定真伪，同时要最大化隐变量 c 和生成分布的互信息I(c;G(z,c)) ,互信息表示两个随机变量的依赖程度			
		
双向生成式对抗网络
自编码生成式对抗网络
	VAE-GAN 自编码机可学习隐变量表示的性质对GAN 进行了改进
组合生成式对抗网络  对朴素GAN 进行堆叠、平行或相互反馈,来调整D和G的组合方式
	GAP
		同时训练几组 GAN ,并令每个判别器周期性地与其他 GAN 的生成器进行对抗训练 /一对一对抗，变多对多混合对抗/
		将 GAP 视为正则化手段	
WGAN Wasserstein 距离解决了梯度消失问题	
WGAN-GP 添加梯度惩罚的方式 [83] , 进一步提高了网络的稳定性 , 在多种网络结构上都可实现收敛 , 是目前性能最佳 , 使用最广泛的 GAN 变种之一



->人工智能研究的新前线:生成式对抗网络|林懿伦
背景
	无监督学习是AI领域下一步要着重解决的问题
	早期生成模型过拟合,研究者提出随机反向传播方法，通过加入额外的独立于模型的随机输入z
	缺点
		训练过程不稳定
		模式坍塌 生成器倾向生成重复但会被判别器认为真实的少数几种甚至一种样本
		原始GAN只能用于生成连续数据 , 无法生成离散数据
		GAN 的评价问题困难
		/
		FCN-score [29]思想
			真实图像训练，对生成图像进行分类;对于图像生成，图像生成质量是看细节，而分类太抽象,存在噪声影响视觉效果，but可能不影响分类.		
		/
		
		
正则方法
	BN
	WN 权值规范化 常用的方法是将网络权值除以其范数,在 GAN 网络中使用 WN 可以取得比 BN更好的效果	
	SN 权值W/权值矩阵奇异值可以极大地提高 GAN 的生成效果
	Minibatch discrimination
集成学习
	AdaGAN 训练 T 个生成器模型，前一次未能成功生成的模式会被加大权重
	Stack GAN  网络叠加，将上层生成器的输出作为隐变量输入下层生成器
	CoGAN 
		Bagging 思想的集成方法主要针对模式坍塌 使用多个网络 , 每个网络针对不同的模式进行训练 , 之后再将这些网络的输出进行整合
		
GAN的应用设计模式
	定义一个模型用于将某一空间中的数据映射至另一空间,再定义一个模型用于评估这一映射的质量
	通过迭代训练两模型得到理想的映射模型或评价模型 
	
总结与展望
	新的研究范式中 , 模型从分析的工具变为了数据的 “ 工厂 ”
	平行思想是指 , 通过将真实系统与人工系统融合 , 在两个平行的系统中迭代实现对另一系统的描述、预测与引导
	
->生成对抗网络理论模型和应用综述|徐一峰
Improved GAN
	特征匹配 让生成器产生的样本与真实样本在判别器中间层尽可能一致 /中间层，最前层太细节，最后一层太抽象/
	最小批量判断  
		生成网络模式单一问题，判别器每次判定一个样本，对相似的点，指出相似的优化方向
		因此不是判别单张图片，而是判别一批，并考虑点与点的相似性
	历史平均 D和G的loss添加||θ−1/t∑[i=1->t]θ[i]||^2
	单边标签平滑 0-0,1->0.9 /标签平滑表示真的不是绝对真，假的不是绝对假，考虑创新点引入随机平滑策略1->[阈值，1]随机值/ 
	VBN 每次将当前数据批次x加入参考集合构建一个新的虚拟的批量数据进行BN
	判定器最后一层sigmod判定真假，改变为softmax,引入数据集的标签，标签共k类,将以前的fake作为第k+1类
	
->生成式对抗网络 : 从生成数据到创造智能|王坤峰
平行学习-新型的机器学习理论框架
	首先从原始数据中选取特定的 “ 小数据 ”, 输入到软件定义的人工系统中 , 并由人工系统产生大量新的数据 ;
	然后这些人工数据和特定的原始小数据一起构成解决复杂问题所需要学习的 “ 大数据 ” 集合 
	通过计算实验和平行执行来设计优化机器学习模型 , 得到应用于某些具体场景或任务的 “ 精准知识 ”
基于对抗训练策略的语言模型数据增强技术
	语言模型的数据增强问题实质上是离散序列的生成问题
	该方法将离散序列生成问题表示为强化学习问题 , 利用判别模型的输出作为奖励对生成模型进行优化 , 采用蒙特卡洛搜索算法对生成序列的中间状态进行评价
	
基于 Regression GAN 的原油总氢物性预测方法
	/
		GAN的G可以生成样本，则G的中间层可以提取样本的潜在特征
		GAN的D可以判定样本，则D的中间层可以提取样本潜在特征	
	/
	增加了一个回归模型,通过对抗学习,使得D提取了原油物性核磁共振氢谱谱图的一系列潜在特征回归模型和D共享首层潜在特征 , 即样本空间的浅层表达 , 有利于提高回归模型的预测精度及稳定性通过在生成模型增加互信息约束 , 并采用回归模型的均方误差损失函数来估计互信息下界,使得生成模型产生更加接近于真实的样本.

->NIPS 2016 Tutorial:Generative Adversarial Networks|Ian Goodfellow
GAN		
	生成器G 
		x=G(z),G一般由DNN表示,z从一些先验分布中采样,z不一定从网络第一层输入，可以在网络结构任何位置引入,设计上的限制非常少	
	判定器D
		D作为生成器的cost,GAN是使用监督学习来逼近难以处理的cost函数的生成模型,Boltzmann机器使用马尔可夫链来估计其cost,VAE使用变分下限来近似其cost\
		
基于最大似然估计的深度生成模型分类
		大多数生成模型基于最大似然估计，最大化模型在数据集m个样本的似然性,最大化m个样本的似然性等于最小化KL散度
	显式模型 
		构建概率密度来表示数据分布
		显式密度模型中，密度可以是计算上易处理的，或者它可能是难以处理的，这意味着为了最大化可能性，必须进行可变近似或蒙特卡罗近似
			密度可以是计算上易处理的
			vs深度信念网络 可并行
			vs非线性ICA  ICA g要可逆,x和z维度必须相同
	难以处理的
			vs VAE
				数学上 近似后验分布太弱或先验分布太弱时，即使完美优化算法和无限训练数据模型也不能很好的逼近真实数据分布
				主观上认为生成质量不佳
			vs Boltzmann机 Markov链通常无法扩展到高维空间，并且增加了使用生成模型的计算成本
隐式模型
	模型没有明确表示数据所在空间的概率分布,提供了从分布中采样的能力(即，可生成样本)
	vs  generative stochastic network Markov链通常无法扩展到高维空间，并且增加了使用生成模型的计算成本

->协作式生成对抗网络
构建两个 ( 或更多 ) 生成器G, 一个判别器D,两个G在，一个D指导下同步训练,G采用不同网络结构/增加G的差异性，防止模式塌缩/
协作 
	s=D(G1(z))-D((G2(z))
	s>0,G1更真，拉近G2->G1
	s<=0,G2更真,拉近G1->G2	
		
->生成式对抗网络在抑郁症分类中的应用|刘宁
	ICA提取特征
	相关系数排序选择特征
	C-DCGAN扩充数据
	高维小样本采用SVM分类

->基于生成对抗网络的恶意域名训练数据生成|袁辰
DGA 域名生成算法技术
	利用随机字符来生成C&C域名，从而逃避域名黑名单检测的技术手段
	
本文实验
	数据集 正常域名|恶意域名，特征从域名中构造
	用GAN扩充恶意域名数据集	
	域名的统计特征 域名长度、n-gram 频率(n=2、3、4、5)、n-gram 正态分 [5] (n=2、3、4、5)、域名元音频率和域名辅音频率
	基于统计特征构造数据集 使用分类器分类
	
->基于生成对抗网络的信息隐藏方案|王耀杰
目的 以图片为载体隐藏信息,利用生成对抗网络生成为更合适信息隐藏、更安全的载体图片
实验	
	利用本文提出的生成对抗网络的Stego-WGAN和其它生成对抗网络生成图片
	使用相同的隐写算法将隐藏信息写入原始图像和生成图像
	使用隐写分析提取隐藏信息，生成图像的分析准确率更低，是更安全的载体
	
结论 
	本文提出的生成对抗网络的Stego-WGAN生成的图片更能抵抗隐写分析，优于已有的方案
/
最优传输理论 分布变换理论，寻找分布变换T:X->Y,将分布X变换到分布Y,并最小化传输代价c(x,T(x))的期望的理论
生成模型和特征提取涉及分布变换和特征解耦
/

#vae
/
ae x-z-x'
vae x->z ,z'->x'    z'采样自p(z|xk) 
假设隐变量z的分布p(z|x)服从正态分布,kl loss KL(p(z|x),N(0,I))使隐变量z分布尽量接近N(0,I)
标准正态分布各分量是独立的，vae使隐变量z接近N(0,I),解耦隐变量特征
vae生成图片效果模糊更多用作图像特征提取器
vae使用x,x'重构误差拉近样本距离，gan使用判定x,x'度量生成拉近分布距离
散度,对抗都可拉近分布的距离,分布的距离和样本距离的区别是分布距离样本的无序性
/
->PM->GAN|郑华滨 	
PM模型 
	解耦隐变量z=[z0,z1,z2,...],ae隐变量z增加预测器P,如果P(z1,z2,...)-z0越大则预测越不准说明特征解耦，更不相关
	编码器E解耦特征使P预测不准，因此E和P存在对抗,E在P的对抗以及D的监督下(减小重构误差)学习到解耦并能更好表示样本的隐特征
对抗自编码器
	ae x-z-x',将通过判别器D的对抗让E学习到的z接近来自N(0,I),即解耦z(vae使用kl loss解耦)
Gan 
	使用D判定x,x'度量生成，拉近分布距离使生成更加逼真	
InfoGan
	z->x'->z'
	D判断生成样本x'和真实样本x,要求生成的x'更加真实(D)
	x'->z'为重构器,要求生成样本x'能很好重构z,若能很好重构z,则说明x'和z有很大的相关性
						
#end#

#activation
->SEARCHING FOR A CTIVATION FUNCTIONS|Google Brain
INTRODUCTION
	RELU使用广泛，其它relu的改进往往通用性不强，大多人工改进适用于解决的问题
	组合穷举和强化学习的自动搜索技术，发现了多个新的激活函数，其中表现最好的为f（x）= x·sigmoid（βx),称为swish	
METHODS
	基本组件 简单常见的一元函数和二元函数,通过基本构件构造激活函数	
	搜索算法构造一个激活函数
	将激活函数替换到某个子网络,训练子网络利用子网络性能评价激活函数，更新搜索算法
	/
		类似遗传编程框架,考虑使用遗传算法构建选择网络基本组件构建网络
	/
SEARCH FINDINGS
	搜索产生了一些函数得到了一些结论
		简单的更好，可能简单更容易优化
		除法不好
		某些周期函数表现不错,如sin,cos等		
SWISH
	无上界有下界,平滑且非单调的函数,在更深网络中略由于RELU
RELATED WORK
	使用搜索技术来发现传统的手工设计组件是最近复兴的元学习子领域的一个实例
	元学习已被用于寻找一次性学习的初始化，适应性强化学习,以及生成模型参数		
	
#parallel(平行学习) learning
->平行学习 — 机器学习的一个新型理论框架|李力
引言		
	强化学习属于主动学习 (Active learn-ing) [17] 的一种 , 我们可以选取特定的行动来兼顾优化目标函数和探索输入数据集合 X(行为影响状态).
	监督|无监督学习，采取的行动不影响数据获取，被动的获取数据
	
平行学习
	数据处理
		从原始数据中选取特定的 “ 小数据 ”, 输入到软件定义的人工系统中 , 并由人工系统产生大量新的数据 . 
		这些人工数据和特定的原始小数据一起构成解决问题所需要学习的 “ 大数据 ” 集合
	从人工合成大数据中学习 , 并将学习到的知识存储在系统状态转移函数中
	学习的知识指导平行控制和平行决策将引导系统进行特定的数据采集 , 获得新的原始数据
	
三大特色方法
	软件定义的人工系统
		构建人工场景来模拟和表示复杂系统的特定场景 , 并将选取的特定 “ 小数据 ” 在平行系统中演化和迭代 , 以受控的形式产生更多因果关系明确、数据格式规整、便于探索利用的大数据 , 再把大数据浓缩成小知识、小智慧和小定律
	预测学习和集成学习
		允许多个智能体共同学习
		每个智能体获取的数据和采取行动的次数和时间均独立.
		以平行世界的角度来看待系统状态的演化过程,将新获得的数据映射到平行空间中
	指示学习
		对抗学习通过构造相互竞争的生成器和辨别器来提高学习的效率,在系统和行动的互动之中 , 达成知识的"泛化"
		对偶学习
			两个智能体分别从事原任务 ( 从集合X 到集合 Y 的学习任务 ) 和对偶任务 ( 从集合 Y 到集合 X 的学习任务 )
			把集合 X 用第一个智能体的模型 F 映射成集合 Y 的子集 Y 0 , 再利用第二个智能体的模型 G 把集合 Y 0 映射成集合X 的子集 X 0 
			比较集合 X 和 X 0 , 可以获得非常有用的反馈信号来改进映射模型 F 和 G

总结 使用预测学习解决如何随时间发展对数据进行探索 ; 使用集成学习解决如何在空间分布上对数据进行探索 ; 使用指示学习解决如何探索数据生成的方向	

##cv&cnn
从“特征工程”到“网络工程”
/
vgg 相同模块堆叠
Inception 
	split - f(x)- merge
	小卷积核 1x1卷积	
Xception dw卷积 - 将通道(C方向)的卷积与空间(HW方向)的卷积分离
Resnet shorcut bottleneck
/
->Resnet
short cut 
	H(X) + X 需要H(X)和X通道数相同 identity mapping的策略有 
		简单补0
		1*1 conv
bottleneck block 1x1,64 -> 3x3,64 -> 1x1,256 
1x1 conv
	起到 reduce and expand c 的作用，降维，增维
	跨通道信息流动
DPN = Resnet + Densenet

->Resnext~多路同构分支结构堆叠+shorcut
vgg 相同结构(block)的堆叠，简单的规则减少了超参数的个数 /2,8定理集中注意力优化重要的/
inception split-transform-merge策略
resnet shorcut
创新点
	多路同构分支，在通道上分裂，相当于Group卷积，
	concat/add 不同分支的输出也是一种集成学习的表现
			
->SENET
conv 在局部感受野 将空间上（spatial）的信息和特征维度上（channel-wise）的信息进行聚合
SE 
	考虑特征通道之间的关系,显式地建模特征通道之间的相互依赖关系* 
	为不同通道加权，加权是一种注意力机制，也是一种隐式降维(提升有用的特征并抑制对当前任务作用不大的特征)
	Sequeeze+Excitation+Scale即 压缩HW，生成通道weight,对feature map不同通道 reweight	
	
->SKNET
看不同尺寸不同远近的物体时，视皮层神经元感受野大小会根据刺激来进行调节
SK
	split 使用不同尺寸的卷积核计算多个feature map,代表不同感受野的特征提取 
	fuse 通过类似SE block 为不同大小感受野的卷积输出生成权重，
	select 根据权重加权求和，权重是注意力机制，是一种对感受野的筛选，相当于自适应调节感受野		
不同尺寸感受野体现在split分支，自适应体现在赋权重，累加，被抑制的感受野相当于不存在，即变相的实现了根据loss自动调节
注意力机制就是赋权，就是降维，就是关注，就是兴奋与抑制

#轻量化cnn
/
信息融合/集成(多->1)方法 特征向量 +/concat
通道间信息混合(多<->多)方法 shuffle 1*1 conv 

直接设计
	空间分离 3x3 conv -> 3x1 conv - 1x3 conv 
	通道分离 conv -> depthwise conv(逐通道conv)  + pointwise conv (1x1 conv)
							  通道分离提取特征               +/-C，通道间信息混合
	group conv  group外分离，group内跨通道
	shuffle 通道信息流动
	Bottleneck结构 维度 Squeeze and Expand，大->小->大，更多的非线性，保证最有用信息通过
	空间是HW方向，通道是C方向
	
automl 基于当前被证明有用的策略，搜索组合
模型压缩 
	参数剪枝/共享，去冗余
	权值量化 
	压缩
	矩阵分解
	知识蒸馏
	低比特网络
	
/
->SqueezeNet
Squeeze layer 使用1*1 conv降维，feature map数量减少 ，1×1降维大大压缩参数数量 //降维又升维，形成了bottleneck结构
Expand layer 使用1*1 conv 和 3*3 conv 增加通道 ，通过pad 0使1x1conv和3x3conv的输出尺寸一样
->Mobilenet V1
//当前的卷积基本都是pad卷积，通道stride=2缩小HW尺寸,stride=1时HW不变
3x3 Conv+BN+ReLU  替换为 3x3 Depthwise Conv+BN+ReLU + 1x1 Pointwise Conv+BN+ReLU
Depthwise 提取通道特征，Pointwise通道信息融合
计算量集中在1x1 conv,1x1 conv可使用高度优化的通用矩阵乘法函数GEMM实现
全局超参数
	宽度乘法器（width multiplier) C方向 所有卷积层kernel的数量统一乘以缩小因子 
	分辨率乘法器（resolution multiplier) HW方向 

->Mobilenet V2
残差块 1x1(压缩压缩)->3x3(卷积)-> 1x1(升维)
倒残差 1x1(升维)->3x3(dw conv+relu)->1x1(压缩降维+线性变换) 
	dw减少了参数数量提取到的信息变少，再压缩特征更少，会降低精度。因此，在逐通道深度卷积之前先升维扩展数通道，是为了丰富特征数量，提高精度。
	dw conv后压缩通道数，自动选择有用的特征，减少参数数量
	ReLU对channel数较低的张量会造成较大的信息损失，因此在每一个压缩降维的1x1conv后取消RELU
	
->Shufflenet V1
Resnet的改造
	1x1 conv -> 分组 1x1卷积(pointwise group convolution) 
	channel shuffle 不同组的信息流动 //工程实现效率不理想
	shorcut
		dw conv stride=1时输入与输出shape一致则shortcut采用add,x+f(x,stride=1)
		dw conv stride=2时hw时，对输入stride=2的池化后concat,{avgpool(x,stride=2),f(x,stride=2)}
->Shufflenet V2
衡量模型复杂度通用指标是FLOPs，指multiply-add数量，but它不完全等同于速度
MAC (memory access cost)不能忽略 //!本文的关注点
四条原则
	深度可分离卷积中复杂度集中在1x1 conv,输入、输出channels数目相同时，conv计算所需的MAC(memory access cost)最为节省
	组卷积，分组过多会增加MAC
	网络碎片化会降低并行度,如多split结构
	element-wise操作，如ReLU和Add，FLOPs较小，却有较大的MAC

​	

##nlp
#语言模型
->Language Modeling with Gated Convolutional Networks
cnn+gate机制用于nlp,使用词嵌入提取特征

/->语言模型|lcy
语言模型 is 根据上下文去预测下一个词，不需要人工标注语料，so语言模型can learn 丰富的语义知识 from 无限制的大语料库

v={w0,w1,...wn}是词表,句子是词的排列,语言模型为句子计算概率
如果语料集合包含所有可能出现的句子则,p(x)=c(x)/N，一般不可能 
context 1, 独立假设,词频 tf-idf
context n, n-gram word2vec   
	n-gram是通过频率来计算p(w|context(w)),p(w|context(w))=c(w)/c(context(w))
	神经模型/基于机器学习的模型是使用模型计算概率p(w|context(w);	rp(w|context(w))=F([w,context(w)],theta),通过训练得到theta
context all rnn，注意力机制
/

#注意力机制
Attention机制是输出y利用权权重信息对输入x的筛选降维
->Attention is all you need|Google Brain(Transformer)
encoder是双向的
decoder是left-to-right的,通过mask不看到未来输入

self-attention
	rnn通过隐藏状态将上文信息输入到下文中,p(xi|xi-1.xi-2,...)
	双向rnn通过双向隐藏状态[<-hi,hi->]得到词i的上下文信息
	transformer 通过self-attention得到词i的上下文信息，通过计算句子自身的attention每个词得到自身的上下文信息,即self-attention输出的向量zi包含的信息相当于双向rnn的[<-hi,hi->]，表示词i的上下文信息
	另一种理解 zi包含了词i自身的信息和句子其它所有词的相似性信息
		
Positional Encoding	
	Transformer相对于rnn丢失了词的位置信息，因此将位置信息编码为向量累加到词嵌入向量作为输入，即每个输入词都是带位置信息的词
	偶数位置，使用正弦编码，在奇数位置，使用余弦编码
	
p(yt|yt-1,yt-2,...;xn,xn-1,...) 中文序列X，对应英文序列Y,翻译出单词yt时，需用到中文序列X以及英文序列（y1,y2,……yt-1）
	encoder提取原句子信息，可并行,xn,xn-1,...信息通过self-attention生成
	decoder是一步生成一个词语yt，非并行,yt-1,yt-2,...的信息通过decoder输入的引入,xn,xn-1,...信息通过context attention引入
		
#迁移学习
->Universal Language Model Fine-tuning for Text Classification/ULMFiT
/
word2vec在通用语料库上在受限窗口内预测中心词语获取词的向量表示，向量蕴含了语义,word2vec训练出的向量是一种固定表示，而词语存在一词多意，在不同句子词语的语义可能是不同的
ELMo 通过训练双向lstm语言模型，训练后保留模型(不是保留词向量)，在迁移到不同任务时，从语言模型可以得到句子相关的词表示，词语在不同句子可产生不同的词向量
ELMo用语言模型（language model）来获取词嵌入，ULMFiT 拿掉最后一层预训练模型
ELMo比ULMFiT更加谨慎,只专注于使用语言模型获取更好的词嵌入  
语言模型预训练是在无监督学习，因此可以利用大量的无标注样本提升监督学习任务的性能
/
三个步骤 
	通用领域语料预训练  通用域
	目标领域语料微调   目标域
		低层的学习率递减，高层要避免overfitting使用更大学习率
		坡度三角学习率 学习率随迭代变化，迅速上升，缓慢下降
	目标任务训练   目标任务
		仅使用用最后一个时间步上的隐变量hT会有信息的丢失，因此hc={hT,maxpooling(H),meanpooling(H)},H={h1,h2,...hT}
		逐渐解冻 
关于Hypercolumns
	指结合多个中间层的输出和输入层的embedding做engineering，提供给下游任务,ELMo采用此方法，
	思想是把低层的语法和高层的语义信息结合起来
	ULMFiT作者不需要此方法也可以/意思是他牛皮/
				
->Improving Language Understanding by Generative Pre-Training|openAI GPT
/与ULMFiT思想类似，ULMFiT使用RNN语言模型,GPT使用transformer decoder模型/
L1(C)是非监督预训练loss,任务通过前k个词预测当前词
L2(C)是监督目标任务loss,在监督训练中加入L1 loss辅助训练，L3(C)=L2(C)+λL1(C)

->BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding
GPT
	使用 transformer decoder模块
	从左到右预测下一个词为目标任务,是单向的
BERT
	使用 transformer encoder模块是双向的，
	bert使用了两个不同的非监督预训练任务，不同于Elmo,GPT,ULMFiT
		Mask LM 完形填空
		下一个句子预测					
/
bert以及迁移学习以及多任务学习的启示,他山之石可以攻玉，其它(其它目标，其它数据，其它...)目标任务的训练学习得到的模型可以为另一个模型生成特征表示
/	

-> Beam Search 
最大化归一化对数似然函数
	1/t^alpha SUM {log p(yt|...y1,X)}	,alpha=[0,1]
	log是避免概率乘数值下溢
	1/t^alpha,t是句子长度，避免倾向生成短句子
	
T=步长，B=束宽，即每一步保存最好结果的数量
每一步t=range{0,T-1}
	从上一步最好的B个结果出发，生成下一个单词 
	最后达到指定步长停止搜索，
使用最大化归一化对数似然函数评价T * B个搜索得到的结果选择，arg max作为翻译结果 

#ml 
->AUC Optimization vs. Error Rate Minimization|Corinna Cortes
二元分类问题
	预测   +1 , -1
真实 +1    TP    FN
     -1    FP    TN
ROC曲线
	真阳率y =TP/(TP+FN),所有+1样例中预测为+1的比率
	假阳率x =FP/(FP+TN),所有-1样例中预测为+1的比率 
	AUC是ROC曲线下的面积,反映了分类器平均性能,随机猜测为0.5,完美分类器为1
	
根据Lemma 1
	AUC可以被视为基于两类分类之间的成对比较的度量
	完美的排名，所有正面例子的排名都高于负面排名，A = 1，任何偏离该排名都会降低AUC
		
我们证明，在某些条件下，RankBoost算法优化的全局函数恰好是AUC  

/rankboost学习一个排名,对排序r1>r2>r3>r4,可构成pair：(r1,r2)(r1,r3),(r1,r4),(r2,r3)(r3,r4),label赋予1
而其余的pair,如(r2,r1)lable赋予-1或0;这样将排序巧妙的转换为分类/		

#归纳偏置
/
归纳偏置即假设集合，对数据集做的假设决定使用怎样的模型,假设也是一种先验知识
/
	