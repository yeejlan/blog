Hexo blog
====

安装hexo
```bash
npm install hexo -g
```

新建post
```bash
hexo new "New Post"
```

产生静态页面
```bash
hexo clean
hexo generate
```

发布到github
```bash
hexo deploy
```

让html文件原样输出，不经过处理

首先找到hexo的安装位置
```bash
where hexo #windows
whereis hexo #linux
```
找到node_modules\hexo\lib\plugins\renderer\index.js，把html相关的处理代码注释掉
```javascript
//renderer.register('htm', 'html', html, true);
//renderer.register('html', 'html', html, true);
```



