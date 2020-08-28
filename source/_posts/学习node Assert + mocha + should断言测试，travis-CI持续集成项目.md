---
title: 学习node Assert + mocha + should断言测试，travis-CI持续集成项目
date: 2019-03-16 17:22:51
tags: Node.js
categories:
  - Node.js
---
## node Assert api学习

1. assert.strictEqual() / assert.notStrictEqual() 判断期望值与实际值是否相等或不相等
2. assert.deepStrictEqual() / assert.notDeepStrictEqual() 判断对象是否完全相等或不相等（深度遍历对象的可枚举属性，且递归对比子对象的可枚举属性）
3. assert.throws() / assert.doesNotThrow() 判断执行的函数是否抛出错误
4. assert.rejects() / assert.doesNotReject() 判断执行异步函数是否返回拒绝状态的Promise
5. assert.ok() 判断是否为真值
6. assert.ifError() 判断实际值是否为undefined 或 null
7. assert.fail() 默认抛出AssertionError提供的错误提示，如果是Error实例，则抛出实例错误提示

以上为个人对Assert类的api理解，第三方断言库的语法基本和此接近，用法上可能更方便，语义化更好。

## mocha + should

[mocha](https://mochajs.org/)单元测试框架

[should](http://shouldjs.github.io/)BDD风格第三方断言库，mocha同时也支持其他第三方断言库，选择哪看个人喜好

![](https://user-gold-cdn.xitu.io/2019/3/15/16981e8c40ebddc7?w=1920&h=656&f=png&s=134977)

根据mocha的Github文档说明安装mocha，


```
npm install -g mocha
```

增加一个名为mocha.opts的配置文件，文件内容为

```
--require should
```

可直接默认添加依赖，无需手动required

*提示：使用最新版本的mocha在package.json文件所在目录下执行mocha命令不会加载opts文件，降低版本可解决这个问题*

以下代码为should文档使用的例子

引入should依赖后自动绑定should的api到Object.prototype原型上，如果是Object.create(null)创建的则使用should(params)来使用should的api，具体api参考should文档

```
var should = require('should');

var user = {
    name: 'tj'
  , pets: ['tobi', 'loki', 'jane', 'bandit']
};

user.should.have.property('name', 'tj');
user.should.have.property('pets').with.lengthOf(4);

// If the object was created with Object.create(null)
// then it doesn't inherit `Object.prototype`, so it will not have `.should` getter
// so you can do:
should(user).have.property('name', 'tj');

// also you can test in that way for null's
should(null).not.be.ok();

someAsyncTask(foo, function(err, result){
  should.not.exist(err);
  should.exist(result);
  result.bar.should.equal(foo);
});
```

下面代码为一个测试用例，测试一个大数相加的函数，输入与输出是否一致:

```
let add = require('../lib/add')

describe('大数相加add方法', function () {
    it('字符串"42329"加上字符串"21532"等于"63861"', function () {
        add('42329', '21532')
            .should.equal('63861')
    })

    it('"843529812342341234"加上"236124361425345435"等于"1079654173767686669"', function () {
        add('843529812342341234', '236124361425345435')
            .should.equal('1079654173767686669')
    })
})
```

describe()定义了一组测试用例，describe()内可重复嵌套describe()，第一个参数为测试用例命名，第二个参数为执行的函数

在项目目录下运行mocha命令执行测试
![](https://user-gold-cdn.xitu.io/2019/3/15/16981fc41dbbb857?w=1278&h=340&f=png&s=34723)
大数相加的函数输出值与我们输入的期望值则通过测试。

## Karma + mocha + travis-CI

[Krama](https://karma-runner.github.io/1.0/index.html) 一个基于Node.js的JavaScript测试执行过程管理工具

[travis-CI](https://www.travis-ci.org/) 持续集成构建项目

首先根据krama Github文档安装

```
npm install -g krama
```

安装完成后在我们的项目中执行

```
karma init
```

执行命令后按照官网文档说明初始化部分配置

```
$ karma init

// 选择使用的测试框架
Which testing framework do you want to use?
Press tab to list possible options. Enter to move to the next question.
> jasmine

// 是否使用require.js
Do you want to use Require.js?
This will add Require.js plugin.
Press tab to list possible options. Enter to move to the next question.
> no

// 是否自动绑定浏览器
Do you want to capture a browser automatically?
Press tab to list possible options. Enter empty string to move to the next question.
> Chrome
> Firefox
>

// 输入你的项目依赖源和测试的文件，一个一个输入，最后生成一个数组
What is the location of your source and test files?
You can use glob patterns, eg. "js/*.js" or "test/**/*Spec.js".
Press Enter to move to the next question.
> *.js
> test/**/*.js
>

// 排除部分文件
Should any of the files included by the previous patterns be excluded?
You can use glob patterns, eg. "**/*.swp".
Press Enter to move to the next question.
>

// 监听文件并且当文件改变时候执行测试
Do you want Karma to watch all the files and run the tests on change?
Press tab to list possible options.
> yes

```

最后本人项目生成的karma配置文件如下

```
// Karma configuration
// Generated on Tue Mar 12 2019 21:15:08 GMT+0800 (China Standard Time)

module.exports = function (config) {
    config.set({

        // base path that will be used to resolve all patterns (eg. files, exclude)
        basePath: '',


        // frameworks to use
        // available frameworks: https://npmjs.org/browse/keyword/karma-adapter
        frameworks: ['mocha'],


        // list of files / patterns to load in the browser
        files: [
            'https://cdn.bootcss.com/jquery/2.2.4/jquery.js',
            'node_modules/should/should.js',
            'test/**.js'
        ],


        // list of files to exclude
        exclude: [],


        // preprocess matching files before serving them to the browser
        // available preprocessors: https://npmjs.org/browse/keyword/karma-preprocessor
        preprocessors: {},


        // test results reporter to use
        // possible values: 'dots', 'progress'
        // available reporters: https://npmjs.org/browse/keyword/karma-reporter
        reporters: ['progress'],


        // web server port
        port: 9876,


        // enable / disable colors in the output (reporters and logs)
        colors: true,


        // level of logging
        // possible values: config.LOG_DISABLE || config.LOG_ERROR || config.LOG_WARN || config.LOG_INFO || config.LOG_DEBUG
        logLevel: config.LOG_INFO,


        // enable / disable watching file and executing tests whenever any file changes
        autoWatch: true,


        // start these browsers
        // available browser launchers: https://npmjs.org/browse/keyword/karma-launcher
        browsers: ['Chrome'],

        // Continuous Integration mode
        // if true, Karma captures browsers, runs the tests and exits
        singleRun: true,

        // Concurrency level
        // how many browser should be started simultaneous
        concurrency: Infinity
    })
}

```

把我们的项目接入travis-CI持续集成，首先登陆travis-CI网站[https://www.travis-ci.org/](https://www.travis-ci.org/)

*提示：需要有自己的github账号，并且以github账号授权登陆travis-ci网站*

登陆后可以看到自己github上所有项目仓库，选择你需要持续集成的项目仓库

![](https://user-gold-cdn.xitu.io/2019/3/16/1698468b9abfcecc?w=1382&h=654&f=png&s=95231)
按照文档说明要在项目目录下新增一个.travis.yml文件，配置文件内容参考文档[https://docs.travis-ci.com/user/languages/javascript-with-nodejs/](https://docs.travis-ci.com/user/languages/javascript-with-nodejs/)

下面是简单的配置文件说明

```
language: node_js
node_js:
  - "lts/*"
before_script:
  - export DISPLAY=:99.0
  - sh -e /etc/init.d/xvfb start
```

language: 定义我们测试的语言

node_js: 定义node的版本，可以指定某个特定的版本
![](https://user-gold-cdn.xitu.io/2019/3/16/169845b68d5ea417?w=560&h=302&f=png&s=23934)
按照文档说明，travis-CI执行测试默认执行npm install安装依赖包，安装完依赖后默认执行npm test命令执行测试任务。

before_script:在script配置命令前执行

根据karma官网文档说明接入travis-CI的配置里说明[http://karma-runner.github.io/1.0/plus/travis.html](http://karma-runner.github.io/1.0/plus/travis.html)

![](https://user-gold-cdn.xitu.io/2019/3/16/169846092e4584de?w=2042&h=636&f=png&s=108274)

新增.travis.yml文件后并上传到github项目仓库中，在travis-ci网站中打开开关，travis-ci检测到配置文件会自动执行测试命令

![](https://user-gold-cdn.xitu.io/2019/3/16/169846e9815e2c76?w=3336&h=766&f=png&s=180559)

在job log标签页中可以看到执行任务的日志

![](https://user-gold-cdn.xitu.io/2019/3/16/169847192a1e0f81?w=2728&h=1766&f=png&s=430910)

根据任务日志可以看到我们的测试任务执行情况，4个测试用例均通过测试。

在集成travis-ci到项目中遇到了不少坑，翻了文档看了很多配置，一开始以为是chrome-launch的问题，在before_script中手动安装也是报错；另外一开始是看到文档里是对firefox配置，以为这是专门针对firefox的配置，后来照着karma接入travis-ci的文档配置尝试一下，居然成功了，这结果来得太意外。

![](https://user-gold-cdn.xitu.io/2019/3/16/1698476b924555b9?w=2042&h=636&f=png&s=108274)

另外在karma.conf的文件中，singleRun的配置也需要配置成true，不然travis-ci任务会一直执行。

![](https://user-gold-cdn.xitu.io/2019/3/16/169847b76a941a40?w=497&h=73&f=png&s=9990)

以上所述为第一次探索自动化测试的过程，确实学习到了很多东西，但是很多高级的用法并不太了解，也有不足的地方没有接入到真实项目中配合使用。
