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
![entry][entryimg]

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

[entryimg]: data:image/png;base64,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
