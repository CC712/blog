# webpack 目录管理分析

      根据不同的项目类型和特点，项目文件目录的安排应该是不一样的。

## 多入口应用

      webpack在此类应用中，承担一个骨架的作用，链接所有的module与依赖。
    多入口应用的典型文件分布。

```
    index.html
    |__  js
    |     |__ index.js
    |     |__ vendor.js
    |__  css
    |     |__ index.less
    |     |__ index.css
    |__  img
          |__ img1.jpg
```

    webpack 需要做的工作

        1. 图片前处理 + 打包
        2. css 前处理 + 打包
        3. js/ts 打包 + 混淆 + treeSharking (?

      依赖webpack的js不能按照原生的引用形式。需要在webpack的config里面单独配置入口。html是以模板的形式出现的，如果多入口，意味着多模板多配置。

      一句话来说就是，所有资源都应当在入口js里面得到引用，无论是css还是img还是其他js文件，统统利用`import` `exports` 联系起来。

      这些其他格式的文件，就有各种loader去处理他，最后生成出来，可以利用各种包，得到想要的单独或者内联的形式。

### 如何保留 src 的路径目录结构

    entry的key对应dist文件名。
    htmlWebpackPlugin filename对应 dist里html的文件名。

    是否能自动化这个path命名过程？（必须自动化命名）

        应该是不容易的，因为js与html一般是分离的，在一个模块化的项目里面。
        ```
        a.html
        b.html
            |__  js
            |     |__ a.js
            |     |__ b.js
            |__  css
            |     |__ a.less
            |     |__ b.less
            |     |__ a.css
            |__  img
                |__ img1.jpg
        ```
        这种格式下，chunks还是必须用比较奇特的方法来命名。
        这种格式也是天然路由，对于多页面来说几乎很少人会去否定这种骨架结构。

### 入口极多，如何减少配置的复杂度

      公司主力项目的html文件达到了惊人的2087个，如果手动写entry 和htmlplugin 显然是很容易出错，并且存在很大问题。
      如果，目录的格式是规则的，那么可以用glob去遍历。做一个目录和入口js以及html的映射。
        比如我们希望能够得到图示的entry设置
![](https://raw.githubusercontent.com/CC712/blog/master/resource/img/entriesMap.png)

#### getEntry

```js
const glob = require("glob");

const getEntry = globPath => {
    let entries = {},
        basename,
        tmp,
        pathname;
    glob.sync(globPath).forEach(entry => {
        basename = path.basename(entry, path.extname(entry));
        let m = entry.match(new RegExp(`src/(.*)/${basename}[.]`));
        //首层 无嵌套情况
        m = m ? m[1] : "";
        pathname = m + "/" + basename;
        entries[pathname] = entry;
    });
    console.log(entries);
    return entries;
};
let entries = getEntry("./src/**/js/*.js");
```

#### getHTML
    同理，htmlplugin的设置同样是通过实际文件位置的映射完成的。
```js
const getHTML = globPath => {
    let htmls = getEntry(globPath);
    let res = [];
    for (let dir in htmls) {
        let chunk = (dir + "").split("/");
        chunk.splice(-1, 0, "js");
        // splice 不能链式调用
        chunk = chunk.join("/");
        res.push(
            Object.assign(
                {},
                {
                    hash: true,
                    filename: `${dir}.html`,
                    //chunks 需要对应entry js的key
                    chunks: [chunk],
                    template: `./src/${dir}.html`
                }
            )
        );
    }
    return res;
};
let htmlTpls = getHTML("./src/**/*.html").map(
    tpl => new htmlWebpackPlugin(tpl)
);
```

## 完整配置

```js
const path = require("path");
const glob = require("glob");

const htmlWebpackPlugin = require("html-webpack-plugin");
const MiniCssExtractPlugin = require("mini-css-extract-plugin"); //把css打包成单独的文件
const UglifyJsPlugin = require("uglifyjs-webpack-plugin");

const isDevMode = true;
const getEntry = globPath => {
    let entries = {},
        basename,
        tmp,
        pathname;
    glob.sync(globPath).forEach(entry => {
        basename = path.basename(entry, path.extname(entry));
        let m = entry.match(new RegExp(`src/(.*)/${basename}[.]`));
        //正则匹配可能为空
        m = m ? m[1] : "";
        pathname = m + "/" + basename;
        entries[pathname] = entry;
    });
    console.log(entries);
    return entries;
};
let entries = getEntry("./src/**/js/*.js");

const getHTML = globPath => {
    let htmls = getEntry(globPath);
    let res = [];
    for (let dir in htmls) {
        let chunk = (dir + "").split("/");
        chunk.splice(-1, 0, "js");
        // splice 不能链式调用
        chunk = chunk.join("/");
        res.push(
            Object.assign(
                {},
                {
                    hash: true,
                    filename: `${dir}.html`,
                    //chunks 需要对应entry js的key
                    chunks: [chunk],
                    template: `./src/${dir}.html`
                }
            )
        );
    }
    console.log(res, htmls);
    return res;
};
let htmlTpls = getHTML("./src/**/*.html").map(
    tpl => new htmlWebpackPlugin(tpl)
);
// return;
const config = {
    mode: isDevMode ? "development" : "production",
    // entry: {
    //     //左边是dist目录，右边是src目录
    //     'a/js/a': path.resolve(__dirname, 'src/a/js/a.js'),
    //     'b/js/b': path.resolve(__dirname, 'src/b/js/b.js'),
    // },
    entry: entries,
    output: {
        path: path.resolve(__dirname, "dist"),
        filename: "./[name].bundle.[chunkHash:5].js"
    },
    // optimization: {
    //     // minimizer: [
    //     //     new UglifyJsPlugin({ /* your config */ })
    //     // ],
    //     minimize: false
    // },
    plugins: [
        // new htmlWebpackPlugin({
        //     filename: "a/a.html",
        //     hash: true,
        //     chunks: ['a/a'],
        //     template: './src/a/a.html'
        // }),
        ...htmlTpls,
        //css
        new MiniCssExtractPlugin({
            filename: "[name].[hash].css",
            chunkFilename: "[id].[hash].css"
        })
    ],
    module: {
        rules: [
            {
                test: /\.(le|sc|c)ss$/,
                use: [
                    //顺序是反的，从下到上逐渐解析
                    isDevMode ? "style-loader" : MiniCssExtractPlugin.loader,
                    // MiniCssExtractPlugin.loader,  打包成单独的文件
                    "css-loader",
                    "less-loader"
                ]
            }
        ]
    },
    resolve: {
        extensions: [".js"],
        alias: {
            "@": path.resolve(__dirname, "src")
        }
    }
};
module.exports = config;
```

