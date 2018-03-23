---
layout:     post
title:      " OpenGLES入门学习1-GLKit "
subtitle:   "Make a refresh effect like NetEase News"
date:       2017-03-01 12:00:00
author:     "PeterKwok"
header-img: "img/home-bg-o.jpg"
catalog: true
tags:
    - iOS开发
    - openGL-ES
    - 视频开发
---

## 前言

**2017对于我来说是一个转折点，自己从大公司到了一个创业公司，对于iOS开发的了解也很深刻了，想学习的地方更多，现在公司是做直播app，自己也参与了视频开发相关的项目，就开始openGL的学习，希望可以在这方面有更深的发展**

***

## 正文
只是整个OpenGL学习的起点，目标是通过GLKit，尽量简单地实现把一张图片绘制到屏幕。

### 效果显示
![avatar](https://upload-images.jianshu.io/upload_images/1049769-3ac4ee1f0a331d32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

### 具体细节
####1、新建OpenGL ES上下文
``` objc 
- (void)setupConfig {
    _pkContext = [[EAGLContext alloc]initWithAPI:kEAGLRenderingAPIOpenGLES2];
    GLKView *view = (GLKView *)self.view;
    view.context = _pkContext;
    view.drawableColorFormat = GLKViewDrawableColorFormatRGBA8888;
    [EAGLContext setCurrentContext:_pkContext];
}
```

<code>GLKView* view = (GLKView *)self.view;</code>这里需要在storyboard里面把view的类设置成GLKView，其他代码是OpenGL ES上下文的创建。

####2、顶点数组和索引数组
``` objc 
    //顶点数据，前三个是顶点坐标，后面两个是纹理坐标
    GLfloat vertexData[] =
    {
        0.5, -0.5, 0.0f,    1.0f, 0.0f, //右下
        0.5, 0.5, -0.0f,    1.0f, 1.0f, //右上
        -0.5, 0.5, 0.0f,    0.0f, 1.0f, //左上
        
        0.5, -0.5, 0.0f,    1.0f, 0.0f, //右下
        -0.5, 0.5, 0.0f,    0.0f, 1.0f, //左上
        -0.5, -0.5, 0.0f,   0.0f, 0.0f, //左下
    };
```
顶点数组里包括顶点坐标，OpenGLES的世界坐标系是[-1, 1]，故而点(0, 0)是在屏幕的正中间。
纹理坐标系的取值范围是[0, 1]，原点是在左下角。故而点(0, 0)在左下角，点(1, 1)在右上角。
索引数组是顶点数组的索引，把squareVertexData数组看成4个顶点，每个顶点会有5个GLfloat数据，索引从0开始。

####3、顶点数据缓存
``` objc 
    //顶点数据缓存
    GLuint buffer;
    //glGenBuffers申请一个标识符
    glGenBuffers(1, &buffer);
    //glBindBuffer把标识符绑定到GL_ARRAY_BUFFER上
    glBindBuffer(GL_ARRAY_BUFFER, buffer);
    //glBufferData把顶点数据从cpu内存复制到gpu内存
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertexData), vertexData, GL_STATIC_DRAW);
    
    glEnableVertexAttribArray(GLKVertexAttribPosition); //顶点数据缓存
    //指定了渲染时索引值为 index 的顶点属性数组的数据格式和位置
    //void glVertexAttribPointer( GLuint index, GLint size, GLenum type, GLboolean normalized, GLsizei stride,const GLvoid * pointer);
    //index(指定要修改的顶点属性的索引值)
    //size(指定每个顶点属性的组件数量。必须为1、2、3或者4。初始值为4。（如position是由3个（x,y,z）组成，而颜色是4个（r,g,b,a))
    //type(指定数组中每个组件的数据类型。可用的符号常量有GL_BYTE, GL_UNSIGNED_BYTE, GL_SHORT,GL_UNSIGNED_SHORT, GL_FIXED, 和 GL_FLOAT，初始值为GL_FLOAT)
    //normalized(指定当被访问时，固定点数据值是否应该被归一化（GL_TRUE）或者直接转换为固定点值(GL_FALSE))
    //stride(指定连续顶点属性之间的偏移量。如果为0，那么顶点属性会被理解为：它们是紧密排列在一起的。初始值为0)
    //pointer(指定第一个组件在数组的第一个顶点属性中的偏移量。该数组与GL_ARRAY_BUFFER绑定，储存于缓冲区中。初始值为0)
    glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, sizeof(CGFloat) * 5, (GLfloat *)NULL + 0);//读取顶点数据
    glEnableVertexAttribArray(GLKVertexAttribTexCoord0); //纹理数据缓存
    glVertexAttribPointer(GLKVertexAttribTexCoord0, 2, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * 5, (GLfloat *)NULL + 3);//读取纹理数据
```
这是本章节的核心内容。

* glGenBuffers申请一个标识符
* glBindBuffer把标识符绑定到<code>GL_ARRAY_BUFFER上</code>
* glBufferData把顶点数据从cpu内存复制到gpu内存
* glEnableVertexAttribArray 是开启对应的顶点属性
* glVertexAttribPointer设置合适的格式从buffer里面读取数据

####4、纹理贴图
``` objc 
- (void)uploadTexture {
    //纹理贴图
    NSString* filePath = [[NSBundle mainBundle] pathForResource:@"for_test" ofType:@"jpg"];
    NSDictionary* options = [NSDictionary dictionaryWithObjectsAndKeys:@(1), GLKTextureLoaderOriginBottomLeft, nil];//GLKTextureLoaderOriginBottomLeft 纹理坐标系是相反的
    GLKTextureInfo* textureInfo = [GLKTextureLoader textureWithContentsOfFile:filePath options:options error:nil];
    //着色器
    _pkBaseEffect = [[GLKBaseEffect alloc] init];
    _pkBaseEffect.texture2d0.enabled = GL_TRUE;
    _pkBaseEffect.texture2d0.name = textureInfo.name;
}
```
* GLKTextureLoader读取图片，创建纹理GLKTextureInfo
* 创建着色器GLKBaseEffect，把纹理赋值给着色器

基础

代码带了很多注释，百度下相应的概念，会有更多解释。
如果对OpengGL ES感兴趣，但是却毫无图形学基础的，可以看看[LearnOpenGL教程](https://learnopengl-cn.github.io)。

Demo：[GitHub](https://github.com/handmuch/RefreshControl_News)



