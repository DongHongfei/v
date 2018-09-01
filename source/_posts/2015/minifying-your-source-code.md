title: 压缩你的Ionic App源码【译】

date: 2015-01-26 18:17:56

tags:

---

*译者注：本人翻译功力有限，所以文中难免有翻译不准确的地方，凑合看吧，牛逼的话你看[英文版](http://ionicframework.com/blog/minifying-your-source-code/)的去，完事儿欢迎回来指正交流（^_^）。*

*这篇文章是邀请[Nic Raboy](http://blog.nraboy.com/)来写的，他是在Android， AngularJS， Ionic， Java， SQL和 Unity3D等多个领域都很有建树的应用开发者。Nic经常写一些关于Ionic和如何创建非常棒的hybrid应用的文章。*

先前我介绍过丑化你的Apache Cordova应用源码的重要性。如果你读过我[之前]() 的文章你就会知道，hybrid应用非常容易被反编译，所以混淆你的代码会给那些恶意用户增加不少难度。

<!-- more -->

我还写过一篇[类似](https://blog.nraboy.com/2014/12/use-grunt-lint-uglify-javascript-project/)的文章，介绍如何使用Grunt任务来丑化你的代码。然而，对于Apache Cordova项目来说，Grunt并不是最理想的解决方案，因为它在丑化代码的同时会产生很多错误。

在这篇教程中，我们就来看看对于Apache Cordova项目，在构建之前如何有效的缩减代码(译者注：原文是lint and minify，但在这里实在不知道怎么翻译了%>_<%)。这篇教程同样适用于[Phonegap](http://phonegap.com/)和[Ionic Framework](http://www.ionicframework.com/)项目。

我们就先从创建一个新的Apache Cordova的Android 和 iOS项目开始吧：

```
cordova create TestProject com.nraboy.testproject TestProject
cd TestProject
cordova platform add android
cordova platform add ios
```
注意，只有在Mac下才可以添加或构建ios平台。

这篇教程会分解成两部分讲述：

高亮化项目中的JavaScript错误
为了混淆而丑化代码
一旦你跟着这些步骤走下来，你会发现你的项目变得更好了(译者注：much better shape咋翻译丫%>_<%)。

高亮化项目中的JavaScript错误

为了完成这个任务，我博客的一个订阅者推荐我去看看[Cordova Linter](https://www.npmjs.com/package/cordova-linter)，我去看了，但是根本搞不明白这玩意儿怎么用，我执行完之后它一直告诉我项目没有错误，包里面也没有文档来说明它使如何工作的。

这时，我决定要自己创建Apache Cordova hook。如果你读过我之前关于hooks的[文章](https://blog.nraboy.com/2015/01/hooks-apache-cordova-mobile-applications/)，你就会大概知道我们接下来要干什么了。

创建 hooks/before_prepare/02_jshint.js，如果你在使用Linux或者Mac，确保给了它执行权限。根据文件名你可能会猜到我们要用[JSHint](http://jshint.com/docs/)来linting(咋翻译哇%>_<%)了。打开02_jshint.js，然后添加下面的代码：

```
#!/usr/bin/env node

var fs = require('fs');
var path = require('path');
var jshint = require('jshint').JSHINT;
var async = require('async');

var foldersToProcess = [
    'js'
];

foldersToProcess.forEach(function(folder) {
    processFiles("www/" + folder);
});

function processFiles(dir, callback) {
    var errorCount = 0;
    fs.readdir(dir, function(err, list) {
        if (err) {
            console.log('processFiles err: ' + err);
            return;
        }
        async.eachSeries(list, function(file, innercallback) {
            file = dir + '/' + file;
            fs.stat(file, function(err, stat) {
                if(!stat.isDirectory()) {
                    if(path.extname(file) === ".js") {
                        lintFile(file, function(hasError) {
                            if(hasError) {
                                errorCount++;
                            }
                            innercallback();
                        });
                    } else {
                        innercallback();
                    }
                } else {
                    innercallback();
                }
            });
        }, function(error) {
            if(errorCount > 0) {
                process.exit(1);
            }
        });
    });
}

function lintFile(file, callback) {
    console.log("Linting " + file);
    fs.readFile(file, function(err, data) {
        if(err) {
            console.log('Error: ' + err);
            return;
        }
        if(jshint(data.toString())) {
            console.log('File ' + file + ' has no errors.');
            console.log('-----------------------------------------');
            callback(false);
        } else {
            console.log('Errors in file ' + file);
            var out = jshint.data(),
            errors = out.errors;
            for(var j = 0; j < errors.length; j++) {
                console.log(errors[j].line + ':' + errors[j].character + ' -> ' + errors[j].reason + ' -> ' +
errors[j].evidence);
            }
            console.log('-----------------------------------------');
            callback(true);
        }
    });
}
```
上面这段脚本只会监视 www/js 目录，但随时可能会添加更多的目录。这个目录下所有的文件都会被过滤，如果是JavaScript，那么这个文件就会被放进JSHint。如果任何文件包含错误，那么（错误）就会被呈现到屏幕上，并且脚本会停止接下来所有程序的执行。这意味着如果你使用 cordova build [platform] 来执行脚本，如果包含错误应用是不会继续执行的。

为了实现这个功能，02_jshint.js依赖了两个NodeJS库。在你的项目根目录执行下面命令来安装：

```
$ npm install jshint
$ npm install async
```
为了混淆而丑化代码

对于混淆方面，我的另外一个订阅者建议我去看看[Cordova Uglify](https://www.npmjs.com/package/cordova-uglify)，并不像Cordova Linter一样，这个NPM包像介绍的一样棒。在你的Apache Cordova项目根目录执行下面的命令：

$ npm install cordova-uglify
当安装执行完毕之后，你会发现 hooks/after_prepare/uglify.js 已经被创建好了。在Linux或Mac你需要赋执行权限，否则不会生效。

你可以通过执行 `cordova prepare` 或者 `cordova build [platform]` 来进行测试。

结论

默认情况下Apache Cordova不会在构建之前检查你代码里的错误，这意味在执行之前你不会知道代码里是不是存在错误。Linting你的代码可以减少很多误差造成的拼写错误或丢失/额外的括号。

默认情况下你辛辛苦苦写出来的代码很容易就被反编译出来了，所以在发布之前丑化你的代码来达到混淆的目的是个很棒的主意。

这篇文章下面还有两条视频：

[第一部分](https://www.youtube.com/watch?v=qQiYE6x7cFk)

[第二部分](https://www.youtube.com/watch?v=hoy3MESySWQ)

Ps:下面的视频是放在YouTube上得，需要翻墙才能看，推荐个不错的Chrome插件[红杏](http://honx.in/i/U-LXTOz5NFxwRetD)
