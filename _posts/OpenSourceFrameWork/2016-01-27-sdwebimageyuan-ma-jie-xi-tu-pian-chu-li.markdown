---
layout: post
title: "SDWebImage源码解析---图片处理"
date: 2016-01-27 18:47:31 +0800
comments: true
category: OpenSourceFrameWork
 
---
```
	本篇文章主要是介绍了，在SDWebImage源码查看过程中的一些对于图片处理的方法，这些方法可能在平时的工作中也会用到，所以单独拿出来一篇文章来介绍这些方法。

	关于图片处理的方法个人本身了解的并不深，初期阶段可能只是简单的介绍一下，添加一些自己的理解，如果有问题，还请大家不吝赐教。
	
```

#### 图片的解码

```
+ (UIImage *)decodedImageWithImage:(UIImage *)image {
    // while downloading huge amount of images
    // autorelease the bitmap context
    // and all vars to help system to free memory
    // when there are memory warning.
    // on iOS7, do not forget to call
    // [[SDImageCache sharedImageCache] clearMemory];
    
    @autoreleasepool{
        // do not decode animated images
        if (image.images) { return image; }
    
        CGImageRef imageRef = image.CGImage;
    
        CGImageAlphaInfo alpha = CGImageGetAlphaInfo(imageRef);
        BOOL anyAlpha = (alpha == kCGImageAlphaFirst
        	 			||alpha == kCGImageAlphaLast
        	 			||alpha ==kCGImageAlphaPremultipliedFirst
        	 			||alpha==kCGImageAlphaPremultipliedLast);
    
        if (anyAlpha) { return image; }
    
        size_t width = CGImageGetWidth(imageRef);
        size_t height = CGImageGetHeight(imageRef);
    
        // current
        CGColorSpaceModel imageColorSpaceModel = 
        CGColorSpaceGetModel(CGImageGetColorSpace(imageRef));
        
        CGColorSpaceRef colorspaceRef = 
        						CGImageGetColorSpace(imageRef);
        
        bool unsupportedColorSpace = (imageColorSpaceModel == 0
         	|| imageColorSpaceModel == -1
			|| imageColorSpaceModel == kCGColorSpaceModelCMYK 
			|| imageColorSpaceModel==kCGColorSpaceModelIndexed);
		
        if (unsupportedColorSpace)
            colorspaceRef = CGColorSpaceCreateDeviceRGB();
    
        CGContextRef context = CGBitmapContextCreate(NULL, 
        						width,height,
						CGImageGetBitsPerComponent(imageRef),
                               0,colorspaceRef,
                               kCGBitmapByteOrderDefault | 								kCGImageAlphaPremultipliedFirst);
    
        // Draw the image into the context and retrieve the new 
        	image, which will now have an alpha layer
        	
        CGContextDrawImage(context, CGRectMake(0, 0, width,
        					 height), imageRef);
        					 
        CGImageRef imageRefWithAlpha = 
        					CGBitmapContextCreateImage(context);
        					
        UIImage *imageWithAlpha = [UIImage 
        					imageWithCGImage:imageRefWithAlpha 
        					           scale:image.scale 
        					orientation:image.imageOrientation];
    
        if (unsupportedColorSpace)
            CGColorSpaceRelease(colorspaceRef);
        
        CGContextRelease(context);
        CGImageRelease(imageRefWithAlpha);
        
        return imageWithAlpha;
    }
}

```


#### 由NSData转换为UIImage

```

+ (UIImage *)sd_imageWithData:(NSData *)data {
    if (!data) {
        return nil;
    }
    
    UIImage *image;
    NSString *imageContentType = [NSData 
    						sd_contentTypeForImageData:data];
    						
    if ([imageContentType isEqualToString:@"image/gif"]) {
        image = [UIImage sd_animatedGIFWithData:data];
    }
#ifdef SD_WEBP
    else if ([imageContentType isEqualToString:@"image/webp"])
    {
        image = [UIImage sd_imageWithWebPData:data];
    }
#endif
    else {
        image = [[UIImage alloc] initWithData:data];
        UIImageOrientation orientation = [self 
        				sd_imageOrientationFromImageData:data];
        				
        if (orientation != UIImageOrientationUp) {
            image = [UIImage imageWithCGImage:image.CGImage
                                        scale:image.scale
                                  orientation:orientation];
        }
    }


    return image;
}

```

#### 根据图片的NSData判断这个图片的类型

```
+ (NSString *)sd_contentTypeForImageData:(NSData *)data {
    uint8_t c;
    [data getBytes:&c length:1];
    switch (c) {
        case 0xFF:
            return @"image/jpeg";
        case 0x89:
            return @"image/png";
        case 0x47:
            return @"image/gif";
        case 0x49:
        case 0x4D:
            return @"image/tiff";
        case 0x52:
            // R as RIFF for WEBP
            if ([data length] < 12) {
                return nil;
            }

            NSString *testString = [[NSString alloc] 
            initWithData:
            	[data subdataWithRange:NSMakeRange(0, 12)] 
            				  encoding:NSASCIIStringEncoding];
            				
            if ([testString hasPrefix:@"RIFF"]
             && [testString hasSuffix:@"WEBP"]
             )
             {
                return @"image/webp";
            }

            return nil;
    }
    return nil;
}

```

#### NSData转Gif

```
+ (UIImage *)sd_animatedGIFWithData:(NSData *)data {
    if (!data) {
        return nil;
    }

    CGImageSourceRef source =
    					 CGImageSourceCreateWithData((__bridge 										CFDataRef)data, NULL);

    size_t count = CGImageSourceGetCount(source);

    UIImage *animatedImage;

    if (count <= 1) {
        animatedImage = [[UIImage alloc] initWithData:data];
    }
    else {
        NSMutableArray *images = [NSMutableArray array];

        NSTimeInterval duration = 0.0f;

        for (size_t i = 0; i < count; i++) {
            CGImageRef image = 
            			CGImageSourceCreateImageAtIndex(source, 
            									      i, NULL);

            duration += [self sd_frameDurationAtIndex:i 
            									source:source];

            [images addObject:[UIImage imageWithCGImage:image 
            					scale:[UIScreen mainScreen].scale 
            			  orientation:UIImageOrientationUp]];

            CGImageRelease(image);
        }

        if (!duration) {
            duration = (1.0f / 10.0f) * count;
        }

        animatedImage = [UIImage animatedImageWithImages:images
        									  duration:duration];
    }

    CFRelease(source);

    return animatedImage;
}

```

#### 讲一个webP转换为UIImage

```
+ (UIImage *)sd_imageWithWebPData:(NSData *)data {
    WebPDecoderConfig config;
    if (!WebPInitDecoderConfig(&config)) {
        return nil;
    }

    if (WebPGetFeatures(data.bytes, data.length,
    					 &config.input) != VP8_STATUS_OK) {
        return nil;
    }

    config.output.colorspace = config.input.has_alpha ?
    								 MODE_rgbA : MODE_RGB;
    config.options.use_threads = 1;

    // Decode the WebP image data into a RGBA value array.
    if (WebPDecode(data.bytes, data.length, &config) != 
    									VP8_STATUS_OK)
    {
        return nil;
    }

    int width = config.input.width;
    int height = config.input.height;
    if (config.options.use_scaling) {
        width = config.options.scaled_width;
        height = config.options.scaled_height;
    }

    // Construct a UIImage from the decoded RGBA value array.
    CGDataProviderRef provider =
    CGDataProviderCreateWithData(NULL,
     		config.output.u.RGBA.rgba,
     		 config.output.u.RGBA.size,
     		  FreeImageData);
     		  
    CGColorSpaceRef colorSpaceRef = 
    							CGColorSpaceCreateDeviceRGB();
    							
    CGBitmapInfo bitmapInfo = config.input.has_alpha ? 
    kCGBitmapByteOrder32Big | kCGImageAlphaPremultipliedLast
     : 0;
    
    size_t components = config.input.has_alpha ? 4 : 3;
    
    CGColorRenderingIntent renderingIntent =
    								 kCGRenderingIntentDefault;
    								 
    CGImageRef imageRef = CGImageCreate(width, height, 8, components * 8, components * width, colorSpaceRef, bitmapInfo, provider, NULL, NO, renderingIntent);

    CGColorSpaceRelease(colorSpaceRef);
    CGDataProviderRelease(provider);

    UIImage *image = [[UIImage alloc] initWithCGImage:imageRef];
    CGImageRelease(imageRef);

    return image;
}

```

#### 其他格式

```
其他格式的图片的转换比较简单，直接使用
[[UIImage alloc] initWithData:data];
就可以

```
