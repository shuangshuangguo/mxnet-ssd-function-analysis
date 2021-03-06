# mxnet-ssd-function-analysis


这里主要针对mxnet在实现ssd时用到的函数   
#### 1.	MultiBoxPrior函数   
```python  
anchors = mx.contrib.symbol.MultiBoxPrior(data=from_layer, sizes=size_str, ratios=ratio_str, clip=clip, name="{}_anchors".format(from_name), steps=step)    
```
该函数在每层特征图的每个位置上生成对应anchor   
**(1). 输入参数解释：**    
data：输入特征图    
sizes：每层特征图对应的scales，一般大特征图对应小尺度即小物体。mxnet-ssd代码提供了计算每层size的function，但refinedet每层的scale为对应层stride的4倍，这些设置都根据每篇论文的设定而定。每层的size包含max_size和min_size，也可只包含min_size    
ratios：每个anchor对应几个不同的ratio，如设置为[1, 2, 0.5]。对于ratio=2, 0.5只有min_size，对于ratio=1有max_size和min_size    
clip：是否对超出边界的anchor进行clip，一般设为false   
name：该层特征图所有anchors的名字    
steps：每个anchor中心的步长   
**(2). 输出形式：**    
对于bacth里的每张图片而言，生成anchor的位置都是一样的，所以为了节省内存只存储一次，输出anchors的shape为：(1, num_anchors, 4)，num_anchors此处为该层anchor数目，每个anchor用4列ndarray表示，每一列分别表示anchor两个对角点的坐标(xmin, ymin, xmax, ymax)，坐标为归一化形式   
**(3). 后续操作：**
生成anchors的特征层后需要接上分类层和回归层用于refine anchor。相比于最开始的全连接层，在ssd里分类和回归都是使用3x3卷积实现的，通道大小分别为：num_anchors * (num_classes+1)和num_anchors * 4   

#### 2. MultiBoxTarget函数   
```python
tmp = mx.contrib.symbol.MultiBoxTarget(*[anchors, label, cls_preds], overlap_threshold=.5, ignore_label=-1, negative_mining_ratio=3, minimum_negative
_samples=0, negative_mining_thresh=.5, variances=(0.1, 0.1, 0.2, 0.2), name=" multibox_target")
```
multiboxprior生成大量anchor，且很多anchor与真实物体框的IoU非常小即负类样本，而我们只需保留那些最不确信是负类的。此函数根据每个anchor的分类置信度及给定的参数选择出用于训练的正负样本   
**(1). 输入参数解释：**    
anchors：所有层的MultiBoxPrior的输出，batch shared，shape为(1, num_anchors, 4)，num_anchors此处为所有层anchor数目总和   
label：ground truth，shape为(batch, 63, 6)(pascal voc第二维度大小为63)，第三维度的6列分别表示：(class_id, xmin, ymin, xmax, ymax, difficulity)    
cls_preds：每层特征图对应的分类层输出结果，shape为(batch, num_anchors x (num_classes+1))    
overlap_threshold：IoU阈值，若anchor和gt的IoU超过此阈值则matched    
ignore_label: 一般设为-1，gt的label若是-1的话则不考虑    
negative_mining_ratio：负/正样本比例    
minimum_negative_samples：负样本个数的最小值    
negative_mining_thresh：用于negative mining的阈值    
variances：除以variance就是对prediction和gt的误差进行放大，从而增加loss，增大梯度，加速收敛。variances四个值分别为(vx, vy, vw, vh)，分别对应anchor中心的x、y坐标和anchor长、宽    
name：操作名字    
**(2). 输出形式：**    
输出tmp包含三个值：  
odm_loc_target = tmp[0]: 预测的边框跟真实边框的偏移(具体公式见ssd论文)，大小是batch_size x (num_anchors x 4)    
odm_loc_target_mask = tmp[1]: 用来遮掩不需要的负类锚框的掩码，大小同上    
odm_cls_target = tmp[2]: 锚框的真实的标号，大小是batch_size x num_anchors    

#### 3. MultiBoxDetection函数    
```python
det = mx.contrib.symbol.MultiBoxDetection(*[cls_prob, loc_preds, anchors], nms_threshold=nms_thresh, force_suppress=force_suppress, variances=(0.1, 0.1, 0.2, 0.2), nms_topk=nms_topk, name=" detection")
```
预测阶段，主要是refine anchor + nms    
**(1). 输入参数解释：**    
cls_prob：cls_preds经过softmax函数的输出	    
loc_preds：回归层输出    
anchors：所有层的MultiBoxPrior的输出，batch shared    
nms_threshold：进行nms的IoU阈值    
force_suppress：为True表示：即使两个IoU大的anchor对应不同的类别，也去除一个    
variances：见MultiBoxTarget函数    
nms_topk：根据分类置信度选择nms_topk个anchors，在这些anchors里进行nms    
**(2). 输出形式：**
输出所有边框，每个边框由[class_id, confidence, xmin, ymin, xmax, ymax]表示。其中class_id=-1表示要么这个边框被预测只含有背景，或者被去重掉了。这些边框的排列顺序是按confidence降序排列的，框的坐标为归一化形式

#### 4. SoftmaxOutput函数：
```python
cls_prob =mx.symbol.SoftmaxOutput(data=cls_preds,label=cls_target, ignore_label=-1,use_ignore=True,grad_scale=1.,multi_output=True,normalization='valid',name="cls_prob")
```
(1). 该函数计算交叉熵损失对于softmax输出的梯度，返回cls_prob为softmax函数的输出    
(2). 最小化交叉熵损失时常用此语句，group包含cls_prob时即可最小化    
#### 5. MakeLoss函数
```python
loc_loss_ =
mx.symbol.smooth_l1(name="loc_loss_", data=loc_target_mask *
(loc_preds - loc_target), scalar=1.0)
loc_loss = mx.symbol.MakeLoss(loc_loss_, grad_scale=1., normalization='valid',
name="loc_loss")
```
(1). 自定义损失时常用此语句。上面的SoftmaxOutput函数其实是交叉熵损失和makeloss结合的版本，因其比较常用，故mxnet为它单独实现了    
(2). 当要group返回时，若返回的不是一个loss会报错，此时可通过makeloss解决：    
```python
anchors = mx.symbol.MakeLoss(data=anchors,grad_scale=0, name="anchors")
```
