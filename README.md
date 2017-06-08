###项目中有这样一种需求，给html5网页中图片添加点击事件，并且弹出弹出点击的对应的图片，并且可以保持图片到本地
###应对这样的需求你可能会想到很多方法来实现。
    1. 最简单的方法就是在html5中添加图片的onClick方法，并把图片的src和index传过来。这种方法虽然能够很好的解决这个问题，但是还需要前端代码的支持
    2.使用WebviewJavascripBridge添加objc和js交互的方法，通过调用方法来实现效果，缺点是需要用到第三方,并且同样也需要前端代码的支持
    3.第三种也就是今天这里要着重介绍的方法：objc中动态注入JavaScript代码，动态给img标签添加onClick事件。话不多说，先上代码。
html代码
```
<!DOCTYPE html>
<html>

    <head>
        <meta charset="utf-8" />
        <title>测试demo</title>
        <style type="text/css">
            img {
                width: 100%;
            }
        </style>
    </head>

    <body>
        <p>内容测试我是详情内容1</p>
        <img src="http://img1.cyzone.cn/uploadfile/2017/0602/20170602094901601.jpg">
        <p>内容测试我是详情内容2</p>
        <p>内容测试我是详情内容3</p>
        <img src="http://img1.cyzone.cn/uploadfile/2017/0602/20170602094902133.jpg">
        <p>内容测试我是详情内容4</p>
        <img src="http://img1.cyzone.cn/uploadfile/2017/0602/20170602094902734.jpg">
        <p>内容测试我是详情内容5</p>
        
    </body>

</html>
```
objc代码
```
//
//  ViewController.m
//  JSInsertDemo
//
//  Created by Tiny on 2017/6/8.
//  Copyright © 2017年 LOVEGO. All rights reserved.
//

#import "ViewController.h"

@interface ViewController ()<UIWebViewDelegate>

@property (nonatomic,strong) UIWebView *webView;
@property (nonatomic,strong) NSMutableArray *imgsArr;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    self.imgsArr = [NSMutableArray array];
    
//    NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:@"http://192.168.20.14:8020/JsToObjc/index.html"]]
    //加载本地html
    NSString *path = [[NSBundle mainBundle] pathForResource:@"index.html" ofType:nil];
    NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL fileURLWithPath:path]];
    [self.webView loadRequest:request];
}

-(UIWebView *)webView{
    if (_webView == nil) {
        _webView = [[UIWebView alloc] initWithFrame:self.view.bounds];
        _webView.delegate = self;
        _webView.scrollView.scrollsToTop = NO;
        _webView.backgroundColor = [UIColor whiteColor];
        [self.view addSubview:_webView];
    }
    return _webView;
}


#pragma mark - webViewDelegate
-(BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType{
    if([request.URL.scheme isEqualToString:@"image-preview"]){
        //触发点击事件  -- >拿到是第几个标签被点击了
        NSString *clickImg = request.URL.resourceSpecifier;
        //遍历数组，查询查找当前第几个图被点击了
        NSInteger selectIndex = 0;
        for(int i = 0; i< self.imgsArr.count;i++){
            NSString *imgUrl = self.imgsArr[i];
            if ([imgUrl isEqualToString:clickImg]) {
                selectIndex = i;
                break;
            }
        }
        //拿到当前选中的index  -- > 使用图片浏览器让图片显示出来
        NSLog(@"当前图片%@ 选中index:%zi",clickImg,selectIndex);
        return NO;
    }
    return YES;
}

-(void)webViewDidFinishLoad:(UIWebView *)webView{
    //加载完成之后开始注入js事件
    [self.webView stringByEvaluatingJavaScriptFromString:@"\
     function imageClickAction(){\
        var imgs=document.getElementsByTagName('img');\
        var length=imgs.length;\
        for(var i=0;i<length;i++){\
            img=imgs[i];\
            img.onclick=function(){\
                window.location ='image-preview:'+this.src;\
            }\
        }\
     }\
     "];
    //触发方法 给所有的图片添加onClick方法
    [self.webView stringByEvaluatingJavaScriptFromString:@"imageClickAction();"];
    
    //拿到最终的html代码
//    NSString *HTMLSource = [self.webView stringByEvaluatingJavaScriptFromString:@"document.getElementsByTagName('script')[0].innerHTML"];
    //调用html代码
    [self.webView stringByEvaluatingJavaScriptFromString:@"sendMsg('我是objc传入的值');"];

    //拿到所有img标签对应的图片
    [self handleImgLabel];
}

-(void)handleImgLabel{
    //要拿到所有img标签对应的图片的src
    //1.拿到img标签的个数
    NSUInteger length = [[self.webView stringByEvaluatingJavaScriptFromString:@"document.getElementsByTagName('img').length"] integerValue];
    //2.遍历标签，拿到标签中src的内容
    for (int i =0 ; i < length; i++) {
        NSString *jsStr = [NSString stringWithFormat:@"document.getElementsByTagName('img')[%zi].src",i];
        NSString *img = [self.webView stringByEvaluatingJavaScriptFromString:jsStr];
        //3.将标签中src内容存入数组
        [self.imgsArr addObject:img];
    }
}
@end

```
