# music-top-recommend 
推荐系统思路：  
1、数据预处理（用户画像数据、物品元数据、用户行为数据）   
2、召回（CB、CF算法）    
3、LR训练模型的数据准备，即用户特征数据，物品特征数据  
4、模型准备，即通过LR算法训练模型数据得到w，b   
5、推荐系统流程：  
（1）解析请求：userid，itemid  
（2）加载模型：加载排序模型（model.w，model.b）  
（3）检索候选集合：利用cb，cf去redis里面检索数据库，得到候选集合  
（4）获取用户特征：userid  
（5）获取物品特征：itemid  
（6）打分(逻辑回归，深度学习)，排序  
（7）top-n过滤  
（8）数据包装（itemid->name），返回  

项目描述：
1.	对三个数据进行预处理，合并用户与物品相关信息，数据字段包含itemid、userid、用户信息(年龄、性别、收入、地区)、物品信息（名字、描述、时长、标签）、用户行为数据(收听时长)等。 
2.	粗排召回阶段使用CB算法，基于内容进行jieba中文分词，计算itemid对应分词的tfidf分数，整理训练数据；使用mr 协同 
过滤进行相关性计算，训练得到物品之间对应分数item-item；CF算法则通过协同过滤将UI矩阵转成II矩阵，格式化数据后将结果按k/v形式批量灌入redis数据库。 
3.	精排阶段利用LR进行推荐排序，得到权重w、b用于模型构建。结合用户与物品标签获取用户与物品特征训练数据。 
4.	推荐流程阶段加载特征数据及排序模型，检索redis数据库获取候选集，利用逻辑回归sigmoid函数打分并排序，最终利用可视化页面实现itemid->name进行top10评分相关推荐。 

一、数据预处理  python gen_base.py

总体思路：处理原始的数据：1、用户画像数据 2、物品元数据 3、用户行为数据，把三类数据统一到一个文件里面，供后面cb、cf算法进行计算权重。 
 
用户画像数据（user_profile.data）  
00ea9a2fe9c6810aab440c4d8c050000,女,26-35,20000-100000,江苏  
01a0ae50fd4b9ef6ed04c22a7e421000,女,36-45,0-2000,河北  
002db7d2360562dd16828c4b91402000,女,46-100,5000-10000,云南  
006a184749e3b3eb83e9eb516d522000,男,36-45,2000-5000,天津  
00de61c1d635ad964eef2aefa8292000,女,19-25,2000-5000,内蒙古  
字段：userid, gender, age, salary, location  

物品元数据（music_meta） 
029900100  徐颢菲《猫的借口》284 国内  
字段：itemid, name, desc, total_timelen, location, tags  

用户行为数据（user_watch_pref.sml）  
01e069ed67600f1914e64c0fe773094440903091011519  
01d86fc1401b283d5828c293be290e0861928091017512  
002f4b9c49be9a0b2c13e1c3c4f6a21c891510910138518  


二、召回阶段（粗排）：    
1.CB算法——候选--gen_cb_train.py    

总体思路：将初始化好的用户，物品，用户行为数据进行处理，目的是为了得到token，itemid，score，将itemName进行分词，得到tfidf权重，同时将desc进行分词，处理name和desc，元数据中tags已经分类好无需再次进行切分，只需要用idf词表查处权重即可，针对name、desc、tags三个分词结果，适当调整name权重比例，分别对这三类得出的分数再次进行分数权重划分，最后得到cb的初始数据。

物品元数据：name,desc,tag    
经过数据预处理，得到如下格式的cb训练数据：    
哲,4090309101,0.896607562717    
大连,4090309101,0.568628215367    
舞曲,4090309101,0.713898826298    
大美妞,4090309101,0.896607562717    
网络,4090309101,0.465710816584    
伤感,4090309101,0.628141853463    

(1)协同过滤算法ALS（求相似度的II矩阵）--mr.py     

总体思路：map阶段：把初始化后的结果进行map排序，为了后续两两取pair对，进行map操作输出即可     
         reduce阶段：经过map操作后以token，item，score输出，所以排序是token排好的序，求II矩阵，根据相同的token的item进行相似度计算，最后得到cb_train.data   
    思路： 
        1、统计user，若相同，把相同的user的item和score放入list里面   
        2、不相同，开始进行两两配对，循环该list，进行两两配对，求出相似度   

(2)批量灌入redis数据库--gen_reclist.py     

思路：将已经通过CB算法得到itemA，itemB，score，格式化数据后将结果按k/v形式批量灌入redis数据库。     
    以itemA为key与itemA有相似度的itemB，和分数，以value的形式存入内存库     
        1、创建一个字典，将key放入itemA，value 放入与A对应的不同b和分数     
        2、循环遍历字典，将key加上前缀CB，value以从大到小的分数进行排序，并且相同的item以“——”分割，item和score间用“：”分割    


三、推荐排序LR(精排)   
解析标签获取用户与物品训练数据-- gen_sampes.py     

排序模型rankmodel    
思路：为逻辑回归准备样本数据  
      1、取第一步merge_base.data,得到用户画像数据，用户信息数据，标签数据   
      2、收取样本标签，用户画像信息，物品信息   
      3、抽取用户画像信息，对性别和年龄生成样本数据   
      4、抽取物品特征信息，分词获得token，score，做样本数据   
      5、拼接样本，生成最终的样本信息，作为模型进行训练   

模型加载 --lr.py 
思路：这里我们要用到我们的数据，就需要我们自己写load_data的部分， 首先定义main方法入口，编写load_data其次调用该方法的到x，y训练和测试集，最后输出wegiht，和b偏置 

四、推荐流程阶段--main.py   
step1.解析请求userid、itemid.   
step2.加载模型w,b.   
step3.检索redis获得候选集.    
step4 精排前获取用户特征数据.   
step5：获取物品特征item_feature.data.   
step6:排序   
step7:过滤   
