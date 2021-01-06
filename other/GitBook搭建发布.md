# 安装nodejs

直接安装10.21.0版本的，后续会省很多事。[安装包地址](https://nodejs.org/dist/v10.21.0/)

## windows

直接下载对应的msi安装即可

## Linux

1. 选择自己适合的tar.gz包进行下载

2. `tar -xzvf`解压安装包

3. 环境变量配置

   ```bash
   PATH=$PATH:/root/node-v10.21.0-linux-x64/bin
   export PATH
   source /etc/profile
   ```

下载成功后通过`node -V`可进行验证

```bash
#  node -v
v10.21.0
```

---

# 安装gitbook

直接通过`npm`安装即可

```
npm install -g gitbook-cli
```

---

# 入门

1. 创建一个存放文件的目录

   ```bash
   # mkdir gitbook-home
   ```

2. 初始化目录

   ```bash
   # cd gitbook-home & gitbook init
   ```

3. 启动服务进行测试

   ```bash
   # gitbook serve
   ...
   Starting server ...
   Serving book on http://localhost:4000
   
   ```

   访问本地4000端口即可查看到效果

---

# 上传文章

在`gitbook-home`目录下新建几个自己要上传文章的目录，方便文章分类。

```bash
# cat SUMMARY.md

* [前言](README.md)
* [Linux](linux/README.md)
    * [supervisor](linux/supervisor.md)
    * [vsftpd](linux/vsftpd.md)
* [Ansible](ansible/README.md)
    * [Playbook](ansible/Playbook.md)
    * [Roles](ansible/roles角色.md)
    * [安装介绍](ansible/安装介绍.md)
    * [常用模块](ansible/常用模块.md)
```

---

# 配置插件

在`gitbook_home`目录下编辑`book.json`，若不存在则新建

```json
{
    "title": "suhw gitbook",
    "description": "suhw 学习笔记",
    "author": "suhaowei",
    "language": "zh-hans",
    "root": ".",
    "plugins": [
        "chapter-fold",
        "expandable-chapters",
        "lightbox",
        "-lunr",
        "-search",
        "search-pro",
        "splitter",
        "code",
        "anchor-navigation-ex",
        "github",
        "-highlight",
        "prism"
    ],
    "pluginsConfig": {
        "anchor-navigation-ex": {
            "showLevel": false,
            "showGoTop": true
        },
        "github": {
            "url": "https://github.com/suhaow"
        },
        "prism": {
            "css": [
                "prismjs/themes/prism-okaidia.css"
            ]
        }
    }
}
```

插件用途

| 名称                 | 作用                                         |
| -------------------- | -------------------------------------------- |
| lightbox             | 图片放大                                     |
| code                 | 代码添加行号、添加复制按钮                   |
| search-pro           | 支持中文搜索，需去除默认的"lunr"以及"search" |
| github               | 右上角添加github图标                         |
| splitter             | 侧边栏宽度可调节                             |
| chapter-fold         | 左侧目录折叠                                 |
| expandable-chapters  | 左侧章节目录可折叠                           |
| prism                | 代码块高亮，去除默认的"highlight"            |
| anchor-navigation-ex | 悬浮目录                                     |

更详细的可参考：https://blog.csdn.net/qq_43514847/article/details/86598399

---

# 部署到github

1. github新建仓库。仓库名为 ` 用户名.github.io`

2. 克隆仓库到本地

   ```bash
   git clone git@github.com:suhaow/suhaow.github.io.git
   ```

3. 输出书籍

   ```bash
   cd suhaow.github.io.git & gitbook build ./
   ```

4. 上传书籍

   ```bash
   git add .
   git commit -m "add gitbook"
   git push
   ```

5. 稍等一下访问`htttps://suhaow.github.io`即可查看效果

---

# 参考

https://blog.csdn.net/meiko_zhang/article/details/81350924

https://www.bookstack.cn/read/gitbook-studying/publish-gitpages.md