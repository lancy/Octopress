---
layout: post
title: 如何做一个Letterpress拼词器
tagline: 猫咪都能玩拼词游戏了！
comments: true
categories: 
  - ios
tags: 
  - iOS
  - 图像处理

---

<img src="{{ size.url }}/assets/post/wordhacker0.png" alt="Drawing" style="width: 200px;"/>

## 故事
哥哥家的猫咪有一天迷上了风靡全球的拼词游戏Letterpress，但是贪吃的小猫咪只认识“food”和“milk”这样的词语，所以经常被对面的玩家欺负。可怜的小猫咪向哥哥求助：“喵呜~哥哥~哥哥，他欺负我！”，于是充满爱心和正义感的哥哥就踏上了拯救猫咪的道路。

![image]({{ size.url }}/assets/post/wordhacker1.jpg)

## 开始拯救世界
唔，我们马上来做一个自动拼词器，拼词器必须实现这样的功能：

1. 猫咪只需要选择一张游戏截图，拼词器能自动识别游戏提供的字母。（记住：小喵掌是用不了键盘的哦
2. 拼词器根据识别出来的字母，自动拼出所有可能的单词，并按长度由长到短排序显示。（小猫咪就能方便的挑选单词啦

有了这样的工具，连猫咪都能玩拼词游戏啦！

全部的代码在Github开源托管：[点这里](https://github.com/lancy/letterfun)

## 正式的开始
我们会使用到Xcode5，并创建一个iOS7的应用。我将用到CoreGraph来做图像处理，你需要一些图像处理的基本常识，一些C语言的能力以及一点内存管理的知识。

现在开始吧！

首先创建一个新的Xcode工程，模板选择单页面即可，名字就叫LetterFun（或者任何你和你的猫咪喜欢的名字），设备选择iPhone，其他的选项让你家猫咪决定。

接下来创建一个继承自`NSObject`的类`CYLetterManager`，我们将用它来识别游戏截图里面的字母。在头文件加上这些方法：

```objective-c
// CYLetterManager.h
@interface CYLetterManager : NSObject

- (id)initWithImage:(UIImage *)image;           \\ 1
- (void)trainingWihtAlphabets:(NSArray *)array; \\ 2
- (NSArray *)ocrAlphabets;                      \\ 3

@end
```
1. 我们假定一个`CYLetterManager`的实例只处理一个图片，所以我们使用一个`initWithImage:`的方法，来确保需要我们处理的图片总是被事先载入。
2. `trainingWihtAlphabets:`是一个训练方法，我们人工载入识别后的字母来让其进行训练，以提供后续字母识别的样本。
3. `ocrAlphabets`从图片里识别字母。

接着开始实现`CYLetterManager`。首先申明一些需要使用的变量：

```objective-c
// CYLetterManager.m
@implementation CYLetterManager {
    CGImageRef *_tagImageRefs;
    UIImage *_image;
    CGImageRef *_needProcessImage;
}
```

其中`_image`是我们从`initWithImage:`里初始化得到的图像，其他两个变量，我会在后面用到的时候解释。

实现初始化方法：

```objective-c
- (id)initWithImage:(UIImage *)image
{
    self = [super init];
    if (self) {
        _image = image;
        [self getNeedProcessImages];
    }
    return self;
}
```

接着实现`getNeedProcessImages`，这个方法用来将原图片切分为25个字母的小块，并存入`_needProcessImage`数组内。

```objective-c
- (void)getNeedProcessImages
{
	// 1
	CGImageRef originImageRef = [_image CGImage];
	CGImageRef alphabetsRegionImageRef = CGImageCreateWithImageInRect(originImageRef, CGRectMake(0, CGImageGetHeight(originImageRef) - 640, 640, 640));
	CGFloat width = 640;
	CGFloat height = 640;
	CGFloat blockWidth = width / 5.0;
	CGFloat blockHeight = height / 5.0;

	// 2 create image blocks
	CGImageRef *imagesRefs = malloc(25 * sizeof(CGImageRef));
	for (NSInteger i = 0; i < 5; i++) {
		for (NSInteger j = 0; j < 5; j++) {
			CGRect alphabetRect = CGRectMake(j * blockWidth, i * blockHeight, blockWidth, blockHeight);
			CGImageRef alphabetImageRef = CGImageCreateWithImageInRect(alphabetsRegionImageRef, alphabetRect);
			imagesRefs[i * 5 + j] = alphabetImageRef;
		}
	}

	// 3 transform to binaryImage
	for (NSInteger i = 0; i < 25; i++) {
		CGImageRef binaryImage = [self createBinaryCGImageFromCGImage:imagesRefs[i]];
		CGImageRelease(imagesRefs[i]);
		imagesRefs[i] = binaryImage;
	}

	// 4
	_needProcessImage = imagesRefs;
	CGImageRelease(alphabetsRegionImageRef);
}
```

1. 我们观察游戏截图，发现字母所在的区域在下方的640 * 640。我们使用`CGImageCreateWithImageInRect`函数创建了`alphabetsRegionImageRef`。注意：你需要使用`CGImageRelease`来release这个对象（函数最后一行），而`originImageRef`是由`UIImage`的`CGImage`方法获得的，你并不持有它，故而不需要release。
2. 我们把`alphabetsRegionImageRef`裁剪成了25个小的方块，暂时存在`imagesRefs`数组。
3. 彩色图片包含的信息太多，为了方便我们后续的处理，我们将得到的字母小方块进行二值化。注意：这里我们使用了自定义的函数`createBinaryCGImageFromCGImage`创建了一个二值化的image，再将其替换到数组里前，需要将数组里存在的旧对象release。
4. 最后我们将`imagesRefs`赋值给`_needProcessImage`，并release不需要imageRef。

再来看如何进行图像二值化，先将这几个常数加到`initWithImage:`方法的上面：

```objective-c
const int RED = 0;
const int GREEN = 1;
const int BLUE = 2;
const int ALPHA = 3;
```

之后来实现`createBinaryCGImageFromCGImage`方法，从这里开始我们将涉及到像素的操作:

```objective-c
- (CGImageRef)createBinaryCGImageFromCGImage:(CGImageRef)imageRef
{
	NSInteger width = CGImageGetWidth(imageRef);
	NSInteger height = CGImageGetHeight(imageRef);
	CGRect imageRect = CGRectMake(0, 0, width, height);

	// 1
	UInt32 *pixels = (UInt32 *)malloc(width * height * sizeof(UInt32));
	CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
	CGContextRef contextA = CGBitmapContextCreate(pixels, width, height, 8, width * sizeof(UInt32), colorSpace, kCGBitmapByteOrder32Big | kCGImageAlphaPremultipliedLast);
	CGContextDrawImage(contextA, imageRect, imageRef);

	// 2
	for (NSInteger y = 0; y < height; y++) {
		for (NSInteger x = 0; x < width; x++) {
			UInt8 *rgbaPixel = (UInt8 *)&pixels[y * width + x];
			NSInteger r = rgbaPixel[RED];
			NSInteger g = rgbaPixel[GREEN];
			NSInteger b = rgbaPixel[BLUE];
			if (r + g + b > 255) {
				rgbaPixel[RED] = 255;
				rgbaPixel[GREEN] = 255;
				rgbaPixel[BLUE] = 255;
			} else {
				rgbaPixel[RED] = 0;
				rgbaPixel[GREEN] = 0;
				rgbaPixel[BLUE] = 0;
			}
		}
	}

	// 3
	CGImageRef result = CGBitmapContextCreateImage(contextA);
	CGContextRelease(contextA);
	CGColorSpaceRelease(colorSpace);
	free(pixels);
	return result;
}
```

1. 使用`CGBitmapContextCreate`创建了一个 bitmap graphics context，并将 pixels 设为其 data pointer，再将 image 绘制到 context 上，这样我们可以通过操作 pixels 来直接操作 context 的数据。该方法的其他参数可以参考文档，参数会影响数据，在这里请先使用我提供的参数。
2. 我们遍历了图像的每个像素点对每个点进行二值化，二值化有许多种算法，大体分为固定阀值和自适应阀值两类。这里我们观察待处理图片可知，我们需要提取的字母部分是明显的黑色，这样使用固定的阀值255，即可顺利将其提取，而有颜色的部分会被剔除。
3. 使用`CGBitmapContextCreateImage`来从context创建处理后的图片，并清理数据。

注意：由于c没有autorelease池，你应当在函数（方法）的命名上使用create(或copy)来提醒使用者应当负责 release 对象。

至此，我们已经完成了字母方块的提取和二值化。为了防止我们没出问题，来检查一下成果。

1. 将一张游戏截图"sample.png"拖进Xcode proj内。
2. 在`CYViewController`的`viewDidLoad`里使用该图片实例化一个`CYLetterManager`。
3. 在`CYLetterManager`的`getNeedProcessImages`里的任意地方加上断点，可以是二值化前后，也可以是切小字母块前后。
4. 运行！然后隆重介绍Xcode5的新功能之一，快速预览，当当当当！

以本文最开始的截图为例：

![image]({{ size.url }}/assets/post/wordhacker2.png)

可以看到我们已经成功的截出了第一个字母，并把其转为二值化图片。

## 下一步
载入了需要的图片和进行了预处理之后，我们来进行识别的前奏：获得识别用的样本。为此我们实现 `trainingWihtAlphabets` 方法：

```objective-c
- (void)trainingWihtAlphabets:(NSArray *)array
{
	for (NSInteger i = 0; i < 25; i++) {
		if (array[i]) {
			[self writeImage:_needProcessImage[i] withAlphabet:array[i]];
		}
	}
	[self prepareTagImageRefs];
}
```

该方法接受一个字母数组，里面应该包含着，我们之前载入图片里的，从左到右，从上到下的字母队列。比如`@[@"t", @"e", @"j", ... , @"h"]`;

我们使用 `writeImage:withAlphabet:` 方法，将该图片设为标准样本，写入到文件中。读写 `CGImageRef` 的方法如下：

```objective-c
@import ImageIO;
@import MobileCoreServices;

- (NSString *)pathStringWithAlphabet:(NSString *)alphabet
{
    NSString *imageName = [alphabet stringByAppendingString:@".png"];
    NSString *documentsPath = [@"~/Documents" stringByExpandingTildeInPath];
    NSString *path = [documentsPath stringByAppendingString:[NSString stringWithFormat:@"/%@", imageName]];
    return path;
}

- (CGImageRef)createImageWithAlphabet:(NSString *)alphabet
{
    NSString *path = [self pathStringWithAlphabet:alphabet];
    CGImageRef image = [self createImageFromFile:path];
    return image;
}

- (CGImageRef)createImageFromFile:(NSString *)path
{
    CFURLRef url = (__bridge CFURLRef)[NSURL fileURLWithPath:path];
    CGDataProviderRef dataProvider = CGDataProviderCreateWithURL(url);
    CGImageRef image = CGImageCreateWithPNGDataProvider(dataProvider, NULL, NO, kCGRenderingIntentDefault);
    CGDataProviderRelease(dataProvider);
    return image;
}

- (void)writeImage:(CGImageRef)imageRef withAlphabet:(NSString *)alphabet
{
    NSString *path = [self pathStringWithAlphabet:alphabet];
    [self writeImage:imageRef toFile:path];
}

- (void)writeImage:(CGImageRef)imageRef toFile:(NSString *)path
{
    CFURLRef url = (__bridge CFURLRef)[NSURL fileURLWithPath:path];
    CGImageDestinationRef destination = CGImageDestinationCreateWithURL(url, kUTTypePNG, 1, NULL);
    CGImageDestinationAddImage(destination, imageRef, nil);
    if (!CGImageDestinationFinalize(destination)) {
        NSLog(@"Failed to write image to %@", path);
    }
    CFRelease(destination);
}
```

`prepareTagImageRefs` 方法将磁盘里保存的样本图片摘出来，存在_tagImageRefs数组里面，用于之后的比对。实现如下: 

```objective-c
- (void)prepareTagImageRefs
{
    _tagImageRefs = malloc(26 * sizeof(CGImageRef));
    for (NSInteger i = 0; i < 26; i++) {
        char ch = 'a' + i;
        NSString *alpha = [NSString stringWithFormat:@"%c", ch];
        _tagImageRefs[i] = [self createImageWithAlphabet:alpha];
        if (_tagImageRefs[i] == NULL) {
            NSLog(@"Need sample: %c", ch);
        }
    }
}
```

将 `[self prepareTagImageRefs]` 加到 `initWitImage:` 方法里面，这样我们每次实例化的时候，都会自动从磁盘里读取标记好的样本图片。

**非常需要注意的是**：我们添加dealloc方法（用惯了arc的开发者可能会不习惯），但这是c，是需要我们自己管理内存的。在dealloc里面释放我们的成员变量吧：

```objective-c
- (void)dealloc
{
    for (NSInteger i = 0; i < 26; i++) {
        if (_tagImageRefs[i] != NULL) {
            CGImageRelease(_tagImageRefs[i]);
        }
    }
    free(_tagImageRefs);
    for (NSInteger i = 0; i < 25; i++) {
        CGImageRelease(_needProcessImage[i]);
    }
    free(_needProcessImage);
}
```

接下来，我们需要载入足够多的包含了26个英文字母的sample图片，做好训练，将26个样品图片就都裁剪好的存入磁盘啦！（哥哥写不动了，训练代码在CYViewController里面，翻到最下面看源码啦）

## 识别字母！
OCR技术从最早的模式匹配，到现在流行的特征提取，有各种各样的方法。我们这里不搞那么复杂，而使用最简单粗暴的像素比对。即我们之前将其转化为二值化图像了之后，直接比对两个图片相同的像素点比例即可。

我们使用标记过的`_tagImageRefs`作为比对样本，将要识别的图像与26个标准样本进行比对，当相似度大于某个阀值的时候，我们即判定其为某个字母，实现如下：

```objective-c
- (NSString *)ocrCGImage:(CGImageRef)imageRef
{
    NSInteger result = -1;
    for (NSInteger i = 0; i < 26; i++) {
        CGImageRef tagImage = _tagImageRefs[i];
        if (tagImage != NULL) {
            CGFloat similarity = [self similarityBetweenCGImage:imageRef andCGImage:tagImage];
            if (similarity > 0.92) {
                result = i;
                break;
            }
        }
    }
    if (result == -1) {
        return nil;
    } else {
        char ch = 'a' + result;
        NSString *alpha = [NSString stringWithFormat:@"%c", ch];
        return alpha;
    }
}

// suppose imageRefA has same size with imageRefB
- (CGFloat)similarityBetweenCGImage:(CGImageRef)imageRefA andCGImage:(CGImageRef)imageRefB
{
    CGFloat similarity = 0;
    NSInteger width = CGImageGetWidth(imageRefA);
    NSInteger height = CGImageGetHeight(imageRefA);
    CGRect imageRect = CGRectMake(0, 0, width, height);
    
    UInt32 *pixelsOfImageA = (UInt32 *)malloc(width * height * sizeof(UInt32));
    UInt32 *pixelsOfImageB = (UInt32 *)malloc(width * height * sizeof(UInt32));
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    CGContextRef contextA = CGBitmapContextCreate(pixelsOfImageA, width, height, 8, width * sizeof(UInt32), colorSpace, kCGBitmapByteOrder32Big | kCGImageAlphaPremultipliedLast);
    CGContextRef contextB = CGBitmapContextCreate(pixelsOfImageB, width, height, 8, width * sizeof(UInt32), colorSpace, kCGBitmapByteOrder32Big | kCGImageAlphaPremultipliedLast);
    CGContextDrawImage(contextA, imageRect, imageRefA);
    CGContextDrawImage(contextB, imageRect, imageRefB);
    
    NSInteger similarPixelCount = 0;
    NSInteger allStrokePixelCount = 0;
    for (NSInteger y = 0; y < height; y++) {
        for (NSInteger x = 0; x < width; x++) {
            UInt8 *rgbaPixelA = (UInt8 *)&pixelsOfImageA[y * width + x];
            UInt8 *rgbaPixelB = (UInt8 *)&pixelsOfImageB[y * width + x];
            if (rgbaPixelA[RED] == 0) {
                allStrokePixelCount++;
                if (rgbaPixelA[RED] == rgbaPixelB[RED]) {
                    similarPixelCount++;
                }
            }
        }
    }
    similarity = (CGFloat)similarPixelCount / (CGFloat)allStrokePixelCount;
    
    CGColorSpaceRelease(colorSpace);
    CGContextRelease(contextA);
    CGContextRelease(contextB);
    free(pixelsOfImageA);
    free(pixelsOfImageB);
    
    return similarity;
}
```

有了上面两个识别的方法，我们再实现`ocrAlphabets`方法就很容易了：

```objective-c
- (NSArray *)ocrAlphabets
{
    NSMutableArray *alphabets = [NSMutableArray arrayWithCapacity:25];
    for (NSInteger i = 0; i < 25; i++) {
        NSString *alphabet = [self ocrCGImage:_needProcessImage[i]];
        if (alphabet) {
            [alphabets addObject:alphabet];
        } else {
            [alphabets addObject:@"unknown"];
        }
    }
    return [alphabets copy];
}
```

## 开始拼词
首先，我们需要准备一个词典。你可以在Unix（或者Unix-like）的系统里找到words.txt这个文件，他一般存在 `/usr/share/dict/words, or /usr/dict/words`

将这个文件拷贝出来，并添加到我们的工程里。我们将创建一个 `CYWordHacker` 类来做拼词的事情，实现传入一组字符，返回所有合法单词按长度降序排列的数组的接口，如下：

```objective-c
@interface CYWordHacker : NSObject
- (NSArray *)getAllValidWordWithAlphabets:(NSArray *)alphabets;
@end
```

具体实现从略，可参照源码。

## 界面
做成下面这样就可以了：

![image]({{ size.url }}/assets/post/wordhacker3.png)

界面细节大家就去看源码吧~写不动了~哥哥要和猫咪玩乐去了~

## 最终成品

全部的代码在Github开源托管：[点这里](https://github.com/lancy/letterfun)

<img src="{{ size.url }}/assets/post/wordhacker4.png" alt="Drawing" style="width: 200px;"/>
<img src="{{ size.url }}/assets/post/wordhacker5.png" alt="Drawing" style="width: 200px;"/>

## 还有一件事

这个东西其实到这里并不是就完了，我们将图片二值化后其实去掉了图片的很多信息，比如当前游戏的状态。有兴趣的筒子，可以根据字块的颜色，来识别出游戏的状态，写出更智能更强力拼词器。实现诸如：占有更多对方的格子或者做出最大的block区域等强力功能，甚至求出最优解策略。这就涉及到人工智能的领域啦。


## 联系我
* 写邮件：lancy1014#gmail.com
* 关注我的[微博](http://weibo.com/lancy1014/)
* Fo我的[Github](http://github.com/lancy)
* 在这里写评论留言

Lancy

20 Oct.




