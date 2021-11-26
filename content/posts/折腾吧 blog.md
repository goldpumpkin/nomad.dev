![jenga-ga5f995546_1920](https://qiniu.5ires.top/uPic/jenga-ga5f995546_1920.jpeg)

其实开始有搞一个博客想法的初衷，就是想写一些技术文章，面试的时候可以说一说，体现一下自己的技术。但是到现在都没有坚持下来，很惭愧~不过平时对技术的积累，也都有笔记记录，自己看得懂就行，如果是写文章话，是要给别人看，还要好好的理顺语句和排版，这实在是对我是有些麻烦的，或者说驱动力还不够强烈。这篇主要是今天又折腾了一下博客，现在转到了`vercel`,原因是关注的博主说好，然后又发现都不清楚现在我的博客到底用到了啥玩意儿，正好现在有时间，就梳理一下我的博客历史做一个记录，主要是年龄大了记忆力不太行了。

##Born

`"created_at": "2017-05-20T03:09:19Z"`([查看 repo  信息api](https://api.github.com/repos/goldpumpkin/goldpumpkin.github.io)) --- 多么浪漫的日子啊! 当时的我， 虽然才算是一只脚踏进计算机行业超级小白，但是通过看别人的分享，一步一步按步骤搭建，过程还算顺畅吧。回忆一下，博客是建立在 [Github Pages](https://pages.github.com/) 平台上的，搭配 [Hexo](https://hexo.io/) 框架，无自主域名，默认的域名应该是`https://goldpumpkin.github.io/`， 图库用的是[七牛云](https://www.qiniu.com/)的免费空间，但现在免费空间还没用完（实在是没多少图片上传）。这样我的博客就诞生啦~ 

Summary：

+ 部署平台：Github Pages
+ 代码及文件：本地
+ 博客框架：Hexo
+ 图库：七牛云

关于博客文章，在这之后，迫于种种原因（懒），博客的文章基本没写过几篇，到现在也只有2020年写的了......

##混沌的折腾期

之后的一段时间， 博客很少触及，博客文章没几篇，倒是还想去设计 hexo theme，也是迫于种种原因，夭折。

博客框架应该是看了[少数派的博客](https://sspai.com/post/59904)，从 hexo 变成了 hugo，虽然我的文章很少，但是眼光要放长远一点。

托管平台跟随[这篇博客](https://www.bmpi.dev/dev/guide-to-setup-blog-site-with-zero-cost-5/)，经历了 Github Pages ---> [netlify](https://www.netlify.com/) ---> [vercel](https://vercel.com/) 的转变，首先抛弃 Github Pages 是因为 netlify 支持 hexo 和 hugo 部署脚本的，关联 github repo 可实现持续更新和自动化部署。而从 netlify 到 vercel 是因为 [vercel 有AWS HK 的 CDN](https://aozaki.cc/migrating-to-vercel/)，国内访问速度较快，当然要换到快的平台。

评论系统从 [Valine](https://vuepress-theme-reco.recoluan.com/views/plugins/comments.html) + [LeanCloud](https://www.leancloud.cn/) 转到了 [utteranc](https://utteranc.es/)，因为 utterances 是基于 GitHub issues，主要是不麻烦。

所以，现在(2021-11-26)看到的博客的构成是：

+ 部署平台：Vercel
+ 源代码：Github
+ 博客框架：Hugo
+ 图库：七牛云
+ 评论系统：utteranc
+ 域名：www.smallgolden.com

##Bolg 的各个时期

+ 2017/05/20 --- Born
+ Born ~ 2020/07/14 --- 假死状态
+ 2020/07/14 ~ Now --- 苦苦续命状态
+ Now ~ Future 别再假死了，疯狂呼吸好不好

##Future

写博客就是记录和分享的一种方式， 在写作中帮助自己思考，分享所见所得。

不只是技术（口号先喊起来！）。

