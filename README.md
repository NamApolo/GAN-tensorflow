# GAN

## 01 Basic GAN
 * [Basic GAN](https://github.com/stesha2016/GAN/blob/master/tensorflow_GAN_basic.ipynb)
 * 最基础的GAN网络，全部使用FC进行网络连接
 * D网是［None, 784］ -> [1]
   G网是［None, 100］ -> [784]
   G网的input是随机生成的100个数字，output是通过G网后生成的784（28＊28）的图片G_fake。loss为将G_fake送入D网得到的数字与label为1之间的差值
   D网的input是G_fake和mnist中的图片D_real，output是一个评估是否为真实图片的数字。loss有两部分，第一部分G_fake生成的数字与label为0之间的差值，另一部分是D_real生成的数字与label为1之间的差值，两部分相加。
 * D和G同时训练20000次，虽然开始出现数字的雏形，但是效果并不是太理想。
 ![0次迭代](https://github.com/stesha2016/GAN/blob/master/image/01_00.png)
 ![1800次迭代](https://github.com/stesha2016/GAN/blob/master/image/01_01.png)

## 02 DCGAN
 * [DCGAN MNIST](https://github.com/stesha2016/GAN/blob/master/tensorflow_DCGAN_MNIST_02.ipynb)
 * 引入了卷积运算，通过实验得出一套效果相对不错的模型
 * D网［None, 64, 64, 1］ -> [None, 32, 32, 128] -> [None, 16, 16, 256] -> [None, 8, 8, 512] -> [None, 4, 4, 1024] -> [None, 1, 1, 1]
   G网[None, 1, 1, 100] -> [None, 4, 4, 1024] -> [None, 8, 8, 512] -> [None, 16, 16, 256] -> [None, 32, 32, 128] -> [None, 64, 64, 1]
 * 关键点：
   1. 每一层除了out层，都必须加上batch normalization.
   2. Loss函数与原始GAN一样.
   3. 使用lrelu效果会更好.
   4. 训练一次D网，训练两次G网效果会好一些.
 * 缺点：
   1. 训练过程不稳定，很有可能D网的loss会降到特别小，而G网的loss上升，D网约束不了G网了。
   2. 有时候G网生成的图片多样性不足。
   3. 没有一个值衡量结果的好坏，Dloss与Gloss似乎要达到一种平衡才可以。
 * 效果：
 
 ![15次epoch的效果](https://github.com/stesha2016/GAN/blob/master/image/DCGAN.png)

## 03 WGAN（Wasserstein GAN）
 * [WGAN](https://github.com/stesha2016/GAN/blob/master/tensorflow_WGAN_CIFAR-10_03.ipynb)
 * WGAN 作者通过数学推导，证明了GAN的缺陷，然后针对缺陷进行了改进，网络模型基本不变
 * 关键点：
   1. D网最后一层去掉sigmoid
   2. D网和G网的loss不取log
   3. 每次更新D网的参数后，把他们的绝对值截断到不超过一个固定常数c
   4. 不用Adam，用RMSProp或者SGD
 * 优点就是明显网络稳定很多，相对DCGAN更容易收敛，一般不会出现GAN网D_loss降到特别小而无法约束G网的情况
 * 缺点：对D网的参数范围进行限制后，很容易训练出D网的参数就集中在c值或者-c值上，而不是在-c到c之间，针对这点的改进就是WGAN-GP
 * 效果：
 
 ![初始图片](https://github.com/stesha2016/GAN/blob/master/image/WGAN-02.png)
 ![WGAN效果](https://github.com/stesha2016/GAN/blob/master/image/WGAN-01.png)
 
## 04 WGAN-GP
 * [WGAN-GP](https://github.com/stesha2016/GAN/blob/master/tensorflow_WGANGP_ANIME_04.ipynb)
 * 针对WGAN的D网训练出来的参数很容易集中在c值或者-c值上的缺点，而出现了WGAN-GP。取消了D网参数的截断，而对D网的loss增加了一个惩罚值
 * 关键点：
   1. 对D网的loss增加了一个penalty的值
      mix = real + epsilon*(fake-real)
      D_mix = discriminator(mix)
      grad = tf.gradients(D_mix, mix)[0]
      slopes = tf.sqrt(tf.reduce_sum(tf.square(grad), axis=[1, 2, 3]))
      penalty = tf.reduce_mean(tf.square(slopes - 1))
      D_loss = D_loss_fake - D_loss_real + 10*penalty
   2. D网不使用batch normalization
   3. 调试下来，不使用bias效果会更好
 * 迭代100多个epoch，有些动漫人物的脸部生成的效果很不错：
  ![图片1](https://github.com/stesha2016/GAN/blob/master/image/wgan-gp1.png)
  ![图片2](https://github.com/stesha2016/GAN/blob/master/image/wgan-gp2.png)
  
## 05 pix2pix
 * [pix2pix](https://github.com/stesha2016/GAN/blob/master/tensorflow_pix2pix_FACADES_05.ipynb)
 * 跟上面的GAN算法开始有了差别，这个算法不再是从噪点生成图片了，而是通过图片生成图片，这样可以学习两个图片之间关联的风格，因为有A和B两类图片，所以在设计G网和D网时需要同时考虑两种图片进行训练
 * D网的输入是将A和B两类图片concat后进行训练， ［None, 256, 256, 3］, [None, 256, 256, 3] -> [None, 256, 256, 6]， 最后的output是[None, 30, 30, 1]
 * G网是标准的UNET，用递归的方式实现是最合适的，因为在降维和升维过程中对应位置尺寸的数据还需要进行一次concat，input是[None, 256, 256, 3], output也是[None, 256, 256, 3]
 * 这个网络可以做很多有趣的事情，可以做风格迁移的效果。比如涂色，通过线条画图等等。
 * Loss就是普通的crossentropy，不过加了一个fake图片和realB的差值作为L1 loss。 L1 loss基本上可以比较清晰的衡量网络训练的效果。
 * 用batchnorm就不用bias，不用batchnorm就用bias。

## 06 CycleGAN
 * [CycleGAN](https://github.com/stesha2016/GAN-tensorflow/blob/master/tensorflow_CycleGAN_06.ipynb)
 * CycleGAN网络结构和pix2pix基本一样，增加了一个反向的过程。 A -> B, B > A
 * 网络有四个A->B的G网，B->A的G网，判定A的D网和判定B的D网
 * A --(GA)--> fakeB --(GB)--> recA
 * B --(GB)--> fakeA --(GA)--> recB
 * A&&fakeA&&recA一起用DA进行判定，B&&fakeB&&recB用DB进行判定
 * 这样前后互相约束生成的图片效果会比pix2pix好，而且A与B的图片并不需要是完全对应的图片。比如A1可以和B2进行对应，也可以和B3或者B4进行对应。
 * ![pic1](https://github.com/stesha2016/GAN-tensorflow/blob/master/image/c1.png) ![pic2](https://github.com/stesha2016/GAN-tensorflow/blob/master/image/c3.png) ![pic3](https://github.com/stesha2016/GAN-tensorflow/blob/master/image/c2.png) ![pic4](https://github.com/stesha2016/GAN-tensorflow/blob/master/image/c4.png)
 * 50个epoch的horse2zebra数据的效果，可以看到场景比较单一的情况下效果会好一些
  * ![pic1](https://github.com/stesha2016/GAN-tensorflow/blob/master/image/c5.png) ![pic2](https://github.com/stesha2016/GAN-tensorflow/blob/master/image/c6.png) ![pic3](https://github.com/stesha2016/GAN-tensorflow/blob/master/image/c7.png) ![pic4](https://github.com/stesha2016/GAN-tensorflow/blob/master/image/c8.png)
 * 100个epoch的monet2photo数据集的效果，前两张monet2photo,后两张photo2monet,monet生成photo的效果不如photo生成monet的效果。
