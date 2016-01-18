layout: post
date: 2015-06-26 13:20:12
title: iOS开发中的Markdown渲染
tags: 
- iOS
---

[BearyChat](https://bearychat.com/)的消息是全面支持Markdown语法的，所以在开发[BearyChat](https://bearychat.com/)的iOS客户端的时候需要处理Markdown的渲染。

主要是两套实现方案：
1. 直接将Markdown文本转换成`NSAttributedString`。
2. 先将Markdown文本转换成HTML，再将HTML转换成`NSAttributedString`。

方案1可用的第三方库有：[AttributedMarkdown](https://github.com/dreamwieber/AttributedMarkdown)，这个库是基于C语言的[peg-markdown](https://github.com/humblehacker/peg-markdown/)的封装，经过试验发现对[GitHub Flavored Markdown](http://github.github.com/github-flavored-markdown/)支持的不太好。

方案2可用的第三方库相对多一些：

将Markdown文本转换成HTML可用的第三方库有：[MMMarkdown](https://github.com/mdiep/MMMarkdown)，[GHMarkdownParser](https://github.com/OliverLetterer/GHMarkdownParser)。其中[GHMarkdownParser](https://github.com/OliverLetterer/GHMarkdownParser)对[GitHub Flavored Markdown](http://github.github.com/github-flavored-markdown/)支持比较好。

将HTML转换成`NSAttributedString`，在iOS 7之后`UIKit`为`NSAttributedString`增加了`initWithData:options:documentAttributes:error:`方法可以直接转换：
```objc
NSAttributedString * attributedString = 
[[NSAttributedString alloc] initWithData:[htmlString dataUsingEncoding:NSUTF8StringEncoding] 
                                 options:@{NSDocumentTypeDocumentAttribute: NSHTMLTextDocumentType,
                                           NSCharacterEncodingDocumentAttribute: @(NSUTF8StringEncoding)} 
                      documentAttributes:nil error:nil];
```
但是实测发现，这个方法的计算速度非常慢！google了一下，貌似因为这个方法渲染的过程是需要初始化`ScriptCore`的，每次渲染都要初始化一个`ScriptCore`肯定是不能忍的。
第三方库的替代方案：[DTCoreText](https://github.com/Cocoanetics/DTCoreText)，[NSAttributedString-DDHTML](https://github.com/dbowen/NSAttributedString-DDHTML)。二者之中，`DTCoreText`是一个比较成熟的第三方库，对样式的控制也比较灵活。

所以最终选择的方案是：首先用`GHMarkdownParser`讲Markdown转换成HTML，之后再用`DTCoreText`讲HTML转换成`NSAttributedString`最后交给`UILabel`等控件渲染。
最终的实现代码就比较简单了：
```objc
#import <GHMarkdownParser/GHMarkdownParser.h>
#import <DTCoreText/DTCoreText.h>

@interface MarkdownParser
@property GHMarkdownParser *htmlParser;
@property (nonatomic, copy) NSDictionary *DTCoreText_options;
- (NSAttributedString *)DTCoreText_attributedStringFromMarkdown:(NSString *)text;
@end

@implementation MarkdownParser

- (instancetype)init {
    self = [super init];
    if (self) {
        _htmlParser = [[GHMarkdownParser alloc] init];
        _htmlParser.options = kGHMarkdownAutoLink;
        _htmlParser.githubFlavored = YES;
    }
    return self;
}

- (NSString *)htmlFromMarkdown:(NSString *)text {
    return [self.htmlParser HTMLStringFromMarkdownString:text];
}

- (NSAttributedString *)attributedStringFromMarkdown:(NSString *)text {
    NSString *html = [self htmlFromMarkdown:text];
    NSData *data = [html dataUsingEncoding:NSUTF8StringEncoding];
    NSMutableAttributedString *attributed = [[NSMutableAttributedString alloc] initWithHTMLData:data options:self.DTCoreText_options documentAttributes:nil];    
    return attributed;
}

- (NSDictionary *)DTCoreText_options {
    if (!_DTCoreText_options) {
        _DTCoreText_options = @{
            DTUseiOS6Attributes:@YES,
            DTIgnoreInlineStylesOption:@YES,
            DTDefaultLinkDecoration:@NO,
            DTDefaultLinkColor:[UIColor blueColor],
            DTLinkHighlightColorAttribute:[UIColor redColor],
            DTDefaultFontSize:@15,
            DTDefaultFontFamily:@"Helvetica Neue",
            DTDefaultFontName:@"HelveticaNeue-Light"
        };
    }
    return _DTCoreText_options;
}

@end
```
到这里，绝大部分的问题都解决了，还有一点点小问题：把解析得到的`NSAttributedString`丢给`UILabel`的`attributedString`渲染的时候，在options里设置的链接的颜色是无效的，貌似`UILabel`对链接的渲染颜色是不可改的。继续寻找替代方案：用第三方的[TTTAttributedLabel](https://github.com/TTTAttributedLabel/TTTAttributedLabel)代替UILabel。`TTTAttributedLabel`是`UILabel`的派生类，为`UILabel`提供了更多对`NSAttributedString`的控制。通过为`TTTAttributedLabel`设置超链接的样式最终解决了Markdown渲染的相关问题。

```objc

#import <UIKit/UIKit.h>
#import <TTTAttributedLabel/TTTAttributedLabel.h>

@interface MarkdownLabel : TTTAttributedLabel <TTTAttributedLabelDelegate>
- (void)setDisplayedAttributedString:(id)text;
@end

@implementation MarkdownLabel

- (instancetype)initWithCoder:(NSCoder *)aDecoder {
    self = [super initWithCoder:aDecoder];
    if (self) {
        [self commonConfig];
    }
    return self;
}

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self commonConfig];
    }
    return self;
}

- (void)commonConfig {
    self.delegate = self;
    NSDictionary *linkAttributes = @{
         (id)kCTForegroundColorAttributeName:[UIColor blueColor],
         NSUnderlineStyleAttributeName:@(kCTUnderlineStyleNone),
         NSFontAttributeName:[UIFont fontWithName:@"HelveticaNeue-Light" size:15]
    };
    self.linkAttributes = linkAttributes;
    self.enabledTextCheckingTypes = 0;
}

- (void)setDisplayedAttributedString:(id)text {
    NSMutableArray *linksAndRange = [@[] mutableCopy];
    [self setText:[text string] afterInheritingLabelAttributesAndConfiguringWithBlock:^NSMutableAttributedString *(NSMutableAttributedString *mutableAttributedString) {
        [text enumerateAttributesInRange:NSMakeRange(0, [text length])
                                 options:0
                              usingBlock:^(NSDictionary *attrs, NSRange range, BOOL *stop) {
                                  if (attrs[NSLinkAttributeName]) {
                                      [linksAndRange addObject:@[attrs[NSLinkAttributeName], [NSValue valueWithRange:range]]];
                                  } else {
                                      [mutableAttributedString addAttributes:attrs range:range];
                                  }
                              }];
        return mutableAttributedString;
    }];
    
    for (NSArray *pair in linksAndRange) {
        [self addLinkToURL:pair[0] withRange:[pair[1] rangeValue]];
    }
}

@end
```