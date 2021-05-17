### 用途
- 精简词表，Bert自带的词表有些冗余，精简可以略微提升速度，缓解强迫症

- 扩充词表，Bert的词表缺少一些token，如中文的双引号，且有时想要加入一些领域任务的高频词，
  训练一个字词混合的Bert，各种优点可参考苏神的wobert

- Huggingface的Bert实现了resize_token_embeddings函数，但只是简单暴力地进行末位删除或扩充，
  而实际我们修改词表时，新词表可能对旧词表有复杂的增删，新词表内token的顺序可能也有很大变化，
  我们希望修改后能尽可能完整地保留原模型的权重
  
### 修改方式  
- Bert的MLM层的weights和word_embeddings层是共享的，但独有自己的bias，所以修改词表后，
  需要处理的主要就是word_embeddings层和MLM层的bias，二者参数的key分别为：
  bert.embeddings.word_embeddings.weight、cls.predictions.decoder.bias
  
- 分两种情况处理
  - 旧词表中需要保留的token，复制相应权重
  - 旧词表中找不到的token，一般是长度>=2的词，用旧词表的字将它tokenize成一个个的字（可能存在[UNK]但没关系），取这些字的平均权重
  - 词表修改后的模型，需要重新微调。即使只是精简token且后序不会用上那些token，但只要用上了MLM层，
    结果也会和原先不一致，因为MLM层的softmax是作用于所有token的，token变化了结果也会变化

### 使用方式
- modelPath为旧模型目录，放置好config文件、词表、模型文件
- newModelPath为新模型目录，最初只需放置新的词表（token顺序随意），修改后的模型、config文件将保存到此目录
- 顺序执行jupyter