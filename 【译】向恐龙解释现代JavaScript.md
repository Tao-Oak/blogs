<font face="Times New Roman">

因为公司的需求，我中途从Android转到了前端方向，并开始学习JavaScript。最让我感到困惑的是前端那众多的框架与工具，我不知道它们是干什么的，需不需要去学习。后来在medium上发现了[Peter Jang](https://medium.com/@peterxjang)的这篇文章，它系统地介绍了现在JavaScript的发展背景，以及众多前端工具分别用来解决什么样的问题。文章本身的内容非常地基础，但是它能够解惑，本着好东西就要拿出来分享的原则，我将它翻译成了中文。第一次翻译，肯定会存在很多的问题，还请各位老铁多多见谅。

原文链接：[https://medium.com/the-node-js-collection/modern-javascript-explained-for-dinosaurs-f695e9747b70](https://medium.com/the-node-js-collection/modern-javascript-explained-for-dinosaurs-f695e9747b70)

_申明：本译文中图片均来自原文，以及[@ryanqnorth](https://twitter.com/ryanqnorth)的[恐龙漫画](http://www.qwantz.com/)。侵删。_

分割线以下为译文：

---

<h1>向恐龙解释现代JavaScript</h1>
![](https://cdn-images-1.medium.com/max/1600/1*H8PH-HaV43gZyBJz0mJHxA.png)
中途转过来学习现代JavaScript是一件很难的事情，它的生态系统在快速地发展与改变，以至于你很难理解它众多的工具都是被用来解决什么样的问题。我从1998年开始编程，但是直到2014年，我才从真正意义上开始学习JavaScript。我记得当时我看到[Browserify](http://browserify.org/)，并盯着它的标语：

> “Browserify lets you require('modules') into the browser by bundling up all of your dependencies.”

我几乎不理解这句话中的任何一个词，却还要挣扎着去思考这句对作为开发者的我会有什么样的帮助。

本文的目的是提供JavaScript工具是如何一步步发展到今天(2017年)的一个历史背景。我们将从零开始，构建一个不借助任何工具，只需要简单的HTML和JavaScript的示例网站。接着我们将依次引入不同的工具，并解释它解决了什么样的问题。有了这种历史背景，你将能更好地学习和适应未来不断变化的JavaScript格局。让我们开始吧！

### JavaScript的"老派"使用方式
让我们从一个只使用HTML和JavaScript的“老派(old-school)”网站开始，它需要我们手动地下载和链接文件。这里是一个简单的 ```index.html``` 文件，并链接了一个JavaScript文件：

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>JavaScript Example</title>
  <script src="index.js"></script>
</head>
<body>
  <h1>Hello from HTML!</h1>
</body>
</html>
```

其中 ```<script src="index.js"></script>``` 这一行引用了一个与 ```index.html``` 在同一目录下名为 ```index.js``` 的JavaScript文件。

```javascript
// index.js
console.log("Hello from JavaScript!");
```

这就是你制作一个网站所需的一切。现在，假设我们想要添加一个第三方的库，如[moment.js](http://momentjs.com/)(一个可以将日期格式化为人类可读方式的库)。如下所示，你可以在JavaScript使用 ```moment``` 提供的功能：

```javascript
moment().startOf('day').fromNow();        // 20 hours ago
```

但是这需要你在你的网站中加入mement.js。在[moment.js](http://momentjs.com/)主页，你会看到以下说明：
![](https://cdn-images-1.medium.com/max/1600/1*ef7OX37jr--Jc38ZxO97Iw.png)

我们暂时先忽略右边的__Install__部分的内容。我们可以通过手动下载 ```moment.min.js```，并将其放到```index.html```所在的文件夹中，最后在```index.html```中引用```moment.min.js```的方式来给我们的网站添加moment.js功能，如下所示：

```javascript
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Example</title>
  <link rel="stylesheet" href="index.css">
  <script src="moment.min.js"></script>
  <script src="index.js"></script>
</head>
<body>
  <h1>Hello from HTML!</h1>
</body>
</html>
```

注意，```moment.min.js```在```index.js```加载之前被加载，这意味着我们可以在```index.js```中使用moment的功能，如下所示：

```javascript
// index.js
console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());
```

这就是以前我们使用JavaScript库制作网站的方式，它的好处是很容易理解；而坏的一面是，当JavaScript库更新时，我们需要重新寻找并下载新版本的JavaScript库，这很烦人。

### 使用JavaScript包管理器(npm)

大概从2010年开始，出现了几个相互之间具有竞争关系的包管理器，它们帮助开发者自动化地完成从中央存储库下载和升级第三方依赖包的过程。2013年前后，[Bower](https://bower.io/)毫无疑问是最受欢迎的包管理器，但最终在2015年左右，它被[npm](https://www.npmjs.com/)取代。(值得注意的是，从2016年年末开始，[yarn](https://yarnpkg.com/en/)作为npm的替代品，已经获得了很大的吸引力，但yarn的背后依然使用npm。)

请注意，npm最初是专门为node.js制作的包管理器，node.js是一个旨在运行在服务器而非浏览器上的JavaScript运行时。所以使用npm作为一个前端包管理器来管理运行在浏览器上的库，是一个非常奇怪的选择。

*__注意：__使用包管理器通常会涉及命令行的使用，而在过去，这是一个前端开发者从不需要掌握的技能。如果你从未使用过命令行，这里有一个很好的[入门教程](https://www.learnenough.com/command-line-tutorial)。无论好与坏，知道如何使用命令行是当代JavaScript开发的一个重要组成部分(同时它也为你在其他领域下的开发打开了一扇大门)。*

下面让我们来看看如何通过npm，而非手动下载的方式来安装moment.js。如果你已经安装了node.js，你也已经安装了npm，这意味着你可以在命令行中导航到```index.html```所在的文件夹下并输入如下命令：

```
$ npm init
```

这将提示你几个问题(一般情况下选默认值即可)并创建一个名为```package.json```的新文件。```package.json```是一个配置文件，npm用它来存储一些与工程相关的信息。默认情况下，```package.json```中的内容应该是这样的：

```
{
  "name": "your-project-name",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC"
}
```

要安装moment.js JavaScript库，我们可以按照moment.js主页上的指导，在命令行中输入如下命令：

```
$ npm install moment --save
```

这句命令做了两件事－－首先，它将moment.js包中的所有代码下载到一个名```node_modules```的本地文件夹中。其次，它自动修改 ```package.json```文件，并将moment.js作为工程的一项依赖记录下来。

```
{
  "name": "modern-javascript-example",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "moment": "^2.19.1"
  }
}
```

这在与他人共享工程时非常有用－－你只需要共享 ```package.json```文件，而非整个 ```node_modules```文件夹(它可能会变得非常大)，其他开发者可以通过 ```npm install```命令来自动安装工程所需要的软件包。

所以，现在我们再也不用从网站上手动下载moment.js了，我们可以通过使用npm来自动下载并更新它。查看 ```node_modules```文件夹，我们可以在 ```node_modules/moment/min```文件夹下找到 ```moment.min.js```文件，这意味着我们可以直接在 ```index.html```中链接通过npm下载下来的 ```moment.min.js```：

```
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>JavaScript Example</title>
  <script src="node_modules/moment/min/moment.min.js"></script>
  <script src="index.js"></script>
</head>
<body>
  <h1>Hello from HTML!</h1>
</body>
</html>
```

此时，好的一面是，通过命令行，我们可以使用npm来下载和更新我们的依赖包；坏的一面是，我们需要深入到 ```node_modules```文件夹内部来寻找每个依赖包的位置，并且需要手动将它们添加到HTML中。这非常麻烦，接下来我们将看看如何自动化地执行这一过程。

![](https://cdn-images-1.medium.com/max/1600/1*GeEETvRqyG4o7SZdbU2Guw.png)

### 使用JavaScript模块打包工具(webpack) 

大部分编程语言都提供了将代码从一个文件加载(import)到另一个文件的方法。JavaScript最初被设计成只能在浏览器中运行，出于安全的考虑，它不能访问客户端计算机的文件系统，因此JavaScript不支持这一特性。在很长的一段时间内，我们需要通过全局共享变量的方式来加载组织在不同文件中的JavaScript代码。

在上述moment.js例子中，我们就是这么干的－－整个 ```moment.min.js```文件被加载到HTML中，该HTML定义了一个叫 ```moment```的全局变量，然后在 ```moment.min.js```之后加载的任何文件都可以使用这个变量(无论这个文件是否需要使用到这个变量)。

在2009年，一个名为CommonJS的项目启动，它的目的是定义一个在浏览器之外运行的JavaScript生态系统。CommonJS的一个重要组成部分就是它的模块规范，它最终允许JavaScript能像大多数编程语言一样可以跨跃文件导入和导出代码，而不需要借助全局共享变量。CommonJS模块规范最著名的应用就是node.js。

![](https://cdn-images-1.medium.com/max/1600/1*xeF1flp1zDLLJ4j7rDQ6-Q.png)

正如上文提到的，node.js是一个运行在服务器上的JavaScript运行时。我们可以使用node.js的模块语法来改造前面的例子。此时，我们不需要使用HTML的script标签，就可以直接在JavaScript文件中加载 ```moment.min.js```，代码如下：

```javascript
// index.js
var moment = require('moment');
console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());
```

这就是node.js加载模块的方式，它效果很好，因为node.js是一门可以访问计算机文件系统的服务器端语言。同时node.js也知道每个npm模块路径的位置，所以加载模块时，我们可以简单地写成 ```require('moment')```，而不需要写 ```require('./node_modules/moment/min/moment.min.js')```。

这一切在node.js上都能很好地运行，但当你尝试在浏览器中运行上述代码时，浏览器会报出一个 ```require```未被定义的错误。加载文件必须以同步(会降低执行速度)或异步(可能有时间问题)的方式动态完成，而浏览器不能访问计算机的文件系统，这意味着以这种方式来加载模块会非常的棘手。

这时我们就需要使用到模块打包工具。JavaScript模块打包工具，通过添加一步叫做build(可以访问计算机文件系统)的操作，创建一个与浏览器兼容的最终输出(不需要访问文件系统)，来解决上述问题。在这种场景下，我们需要用模块打包工具来找到所有的 ```require```(这是无效的浏览器JavaScript语法)语句，并将它们替换成每个需要引入的文件的实际内容。最终的输出结果是一个打包好的JavaScript文件(没有require语法)。

[Browserify](http://browserify.org/)在2011年发布后，即成为最受欢迎的模块打包器，它率先在前端引入了node.js样式的require语句(这也是使npm成为前端包管理器之一的原因)。在2015年左右，受React前端框架普及的推动，[webpack](https://webpack.github.io/)打败Browserify成为最受欢迎的模块打包器，React框架充分利用了webpack的各种特性。

让我们来看看如何通过webpack来使 ```require('moment')```语句在浏览器中正常运行。首先，我们需要将webpack安装到工程中，webpack本身就是一个npm包，所以我们可以从命令行安装它：

```
$ npm install webpack --save-dev
```

请注意 ```--save-dev```参数，webpack将会被保存为开发依赖项，这意味着它是在开发环境而非生产环境所需要的软件包，你可以从 ```package.json```中看出这一区别：

```
{
  "name": "modern-javascript-example",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "moment": "^2.19.1"
  },
  "devDependencies": {
    "webpack": "^3.7.1"
  }
}
```

现在，我们将webpack软件包安装到了 ```node_modules```目录下，你可以在通过下述命令来使用webpack：

```
$ ./node_modules/.bin/webpack index.js bundle.js
```

这句命令会运行安装在 ```node_modules```文件夹下的webpack工具，它将从 ```index.js```文件开始，查找所有的 ```require```语句，并将它们替换为对应的代码，最后创建了名为 ```bundle.js```的单个输出文件。在浏览器中，我们不再使用包涵无效 ```require```语句的```index.js```文件了，相反，我们使用webpack输出的 ```bundle.js```文件，HTML文件的改动如下：

```
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>JavaScript Example</title>
  <script src="bundle.js"></script>
</head>
<body>
  <h1>Hello from HTML!</h1>
</body>
</html>
```

此时刷新浏览器，你会看到一切都像以前一样正常工作。

注意，每次修改 ```index.js```文件后，我们都需要运行上述webpack命令，这很枯燥。尤其当我们使用webpack的高级功能(例如[生成源码映射](https://webpack.js.org/guides/development/#using-source-maps)以帮助我们从转译后的代码调试原始代码)时，它会变得更加枯燥乏味。webpack可以从工程的根目录下一个名为 ```webpack.config.js```的文件中读取配置选项，在我们的例子中，```webpack.config.js```应该长这样：

```
// webpack.config.js
module.exports = {
  entry: './index.js',
  output: {
    filename: 'bundle.js'
  }
};
```

现在每次修改 ```index.js```文件后，我们只需要运行下述命令：

```
$ ./node_modules/.bin/webpack
```

我们不需要再指定```index.js```和 ```bundle.js```这两个选项了，因为webpack可以从 ```webpack.config.js```中读取这些选项。相比之前，这更简单一点，但每次修改代码后都需要输入这行命令依然很枯燥。接下来我们将会使这个过程变得更顺滑一点。

我们虽然没有做很大的改进，但这个工作流程却有一些巨大的优势。我们再也不用通过全局共享变量的方式来加载外部脚本了，任何新的JavaScript库都将通过JavaScript语言中的```require```语句来添加，而不是通过在HTML中添加新的\<script\>标签。相对加载多个JavaScript文件而言，加载单个打包好的JavaScript文件通常会有更好的性能表现。同时，既然我们已经在开发工作流程中添加了一个构建步骤，我们还可以添加一些其他的强大功能！

![](https://cdn-images-1.medium.com/max/1600/1*ee_ivxNTKgIJTjmEMC4-dg.png)

### 转译代码以支持新的语言特性(babel)

转译代码意味着将一种语言的代码转换为另一种语言的代码，这是前端开发的另外一个重要组成部分－－因为浏览器添加新功能的速度非常缓慢，而带有实验性功能的新语言可以被转译成浏览器兼容的语言。

对CSS来说，有[Sass](http://sass-lang.com/)，[Less](http://lesscss.org/)和[Stylus](http://stylus-lang.com/)等。而对于JavaScript，曾经一段时间内最受欢迎的转换器是CoffeeScript(2010年左右发布)，而现在，大部分人都使用[babel](https://babeljs.io/)或[TypeScript](http://www.typescriptlang.org/)。CoffeeScript是一门通过大量修改JavaScript语言来提升JavaScript能力的一门语言。Babel本身不是一门新的语言，但是它是一个将浏览器尚未全部兼容的下一代JavaScript(ES6及以上)转换为更兼容的JavaScript(ES5)的转换器。Typescript是一种与下一代JavaScript基本相同的语言，同时也增加了可选的静态类型。许多人选择使用babel，因为它最接近vanilla JavaScript。

下面让我们来看一个如何在现有的webpack构建过程中使用babel的例子。首先我们需要通过下面的命令将babel安装到我们的工程中：

```
$ npm install babel-core babel-preset-env babel-loader --save-dev
```

请注意，我们在开发依赖中安装了三个独立的软件包-- ```babel-core```是babel的主要组成部分，```babel-preset-env```预定义了需要转换的JavaScript新特性，```babel-loader```是一个能使babel与webpack一起工作的软件包。我们通过修改 ```webpack.config.js```来使用```babel-loader```：

```javascript
// webpack.config.js
module.exports = {
  entry: './index.js',
  output: {
    filename: 'bundle.js'
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['env']
          }
        }
      }
    ]
  }
};
```

这个语法可能会让人感到困惑，不过幸运的是，我们不需要经常去改动它。它的基本工作原理是，告诉webpack去查找所有.js文件(包括 ```node_modules```中的.js文件)，然后通过```babel-loader```与```babel-preset-env```来进行转换。你可以在[这里](https://webpack.js.org/configuration/)看到webpack的更多配置语法。

现在一切都已搭建好，我们可以在JavaScript中使用ES2015的新特性了。以下是一个使用[ES2015模版字符串](https://babeljs.io/docs/en/learn/#ecmascript-2015-features-template-strings)的例子：

```javascript
// index.js
var moment = require('moment');

console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());

var name = "Bob", time = "today";
console.log(`Hello ${name}, how are you ${time}?`);
```

我们还可以使用[ES2015 的import](https://babeljs.io/docs/en/learn/#ecmascript-2015-features-modules)语句来替代require语句加载模块的功能。今天，你可以在很多代码库发现这种写法。

```javascript
// index.js
import moment from 'moment';

console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());

var name = "Bob", time = "today";
console.log(`Hello ${name}, how are you ${time}?`);
```

在这个例子中，import语法与require语法并没有多大的区别，但对于更高级的场景，import语法会更灵活。因为修改了 ```index.js```文件，我们需要再次运行这句命令：

```
$ ./node_modules/.bin/webpack
```
刷新浏览器后你会看到期望的效果。在写这篇文章的时候，许多当代浏览器都已经支持了ES2015的特性，所以很难分辨出babel是否起了作用。你可以在IE9等旧版浏览器中进行测试，也可以在 ```bundle.js```查找转换后的这行代码：

```javascript
// bundle.js
// ...
console.log('Hello ' + name + ', how are you ' + time + '?');
// ...
```

你可以看到，babel将ES2015的模版字符串转换成了常规的JavaScript字符串拼接，以此来保证浏览器的兼容性。尽管这个例子可能不太令人兴奋，但是能转译代码是一个非常强大的功能。它让我们现在就可以使用JavaScript即将要支持的新特性，如[async/await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)等，来编写更好的代码。尽管转译过程有时候会很繁琐很痛苦，但它让我们能够在今天去测试将来的语言特性，这也使得JavaScript语言在过去几年里取得了显著的提升。

至此，我们差不多已经讲完了，但在我们的工作流中依然存在一些未完成的边界。如果我们关心性能，我们需要缩小([minifying](https://en.wikipedia.org/wiki/Minification_%28programming%29))打包后的文件，因为我们有了构建这一步，缩小代码也变得非常简单。每次修改JavaScript后，我们都需要重新运行webpack命令，接下来我们将讨如何通过工具来解决这一问题。

### 使用task runner(npm脚本)
既然我们已经使用了构建步骤来处理JavaScript模块，使用一个task runner来自动化各构建任务也变得非常合理。对于前端开发，任务包括缩小代码(minifying code)、优化图片，运行测试等。

在2013年，Grunt是最流行的前端task runner，随后Glup流行了一小段时间，它们都是借助插件来包装其他的命令行工具。而现在最流行的选择似乎是使用npm包管理器内置的脚本功能，该功能不需要依赖插件，而是直接与其他命令行工具一起使用。

让我们编写一些npm脚本来帮助我们更方便地使用webpack，这需要改动 ```package.json```文件：

```
{
  "name": "modern-javascript-example",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "webpack --progress -p",
    "watch": "webpack --progress --watch"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "moment": "^2.19.1"
  },
  "devDependencies": {
    "babel-core": "^6.26.0",
    "babel-loader": "^7.1.2",
    "babel-preset-env": "^1.6.1",
    "webpack": "^3.7.1"
  }
}
```

这里我们添加了两个新的脚本：```build```和```watch```。我们需要通过下述命令来运行```build```脚本：

```
$ npm run build
```
这会运行webpack(使用我们之前写在```webpack.config.js```中的配置项)，并带了```--progress```和```-p```这两个可选项。其中```--progress```用于展示百分比进度，```-p```会在生产环境下压缩代码。下述命令会运行```watch```脚本：

```
$ npm run watch
```
这使用了```--watch```选项，它会在每次JavaScript文件更改后自动运行webpack，这对开发者来说非常有用。

请注意，在使用 ```package.json```中的脚本运行webpack时，因为 ```node.js```知道所有npm模块的路径，我不需要指定webpack的全路径：```./node_modules/.bin/webpack```。这非常方便！此外，我们还可以通过安装 ```webpack-dev-server```，来让事情变得更加简单。 ```webpack-dev-server```是一个单独的工具，它提供了一个带有实时重新加载(live reloading)功能的简单web服务器。我们可以通过下述命令将它安装为开发依赖：

```
$ npm install webpack-dev-server --save-dev 
```
然后在 ```package.json```文件添加如下npm脚本：

```
{
  "name": "modern-javascript-example",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "webpack --progress -p",
    "watch": "webpack --progress --watch",
    "server": "webpack-dev-server --open"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "moment": "^2.19.1"
  },
  "devDependencies": {
    "babel-core": "^6.26.0",
    "babel-loader": "^7.1.2",
    "babel-preset-env": "^1.6.1",
    "webpack": "^3.7.1"
  }
}
```

现在，你可以通过以下命令启动开发服务器：

```
$ npm run server
```

它会自动在浏览器中打开地址为 ```localhost:8080```(默认情况下)的网站，并显示 ```index.html```中的内容。你每次改变 ```index.js```中的JavaScript代码， ```webpack-dev-server```都会重新生成打包(bundle)后的JavaScript代码，并自动刷新浏览器。这可以显著地节省我们的时间，因为它能够使我们专注于代码，而不需要不断地在代码与浏览器之间切换以查看新的更改。

你当然也可以编写npm脚本来执行其他的任务，例如将Sass转换为CSS、压缩图片，运行测试 -- 任何具有命令行的工具我们都可以通过npm脚本来执行。npm本身也有一些很棒的高级选项和技巧，[Kate Hudson](https://twitter.com/k88hudson)的这个[演讲](https://www.youtube.com/watch?reload=9&v=0RYETb9YVrk)就是一个很好的开始。

### 结论

简而言之，这就是现代JavaScript。我们从简单的HTML和JS转向使用**包管理器(package manager)**来自动下载第三方依赖包，使用**模块打包器(module bundler)**来创建单个脚本文件，使用**转译器(transpiler)**来支持未来的语言特性，和使用**task runner**来自动化各构建过程。这里面肯定有许多迷人的东西，特别是对一个初学者来说。Web开发因为它很容易开始与运行，曾是编程新手们的一个很好的切入点；但现在它可能会非常难，特别是在各种工具都在快速变化的情况下。

尽管如此，这一切并没有像看起来的那么糟。事情正在稳定下来，特别是在引入node生态作为一种可用的前端开发方式后。我们可以使用npm作为包管理器，使用node的```require```或```import```语句来处理模块，使用npm脚本来运行各种任务，这个开发流程很好也很一致，与一两年前相比，这已经是一个大大简化的工作流程了。

对初学者或经验丰富的开发人员来说，另外一个好消息是现在的框架通常都会提供一些工具，以帮助开发者们更简单地开始或入门。如Ember的 ```ember-cli```，它对Angular的 ```angular-cli```有非常大的影响、React的 ```create-react-app```、Vue的```vue-cli```等等。所有这些工具都会帮你设置好工程需要的一切－－你所需要做的就是开始编写代码。这些工具本身并不神奇，它们只是按照一种一致且起作用的方式帮你设置好所有的东西－－但你依然可能会遇到需要对webpack、babel等工具进行额外配置的情况，因此理解这些工具都是干什么的依然十分重要。

现代JavaScript肯定会让人感到沮丧，因为它依然在快速地发展与进化。即使它有时可能会重复造一些轮子，但JavaScript的快速发展也推动了诸如hot reloading、real-time linting和time-travel debugging等创新技术的产生。成为一名开发人员是一个激动人心的时刻，我希望这些信息能够成为一份路线图，帮助您踏上旅途！

![](https://cdn-images-1.medium.com/max/1600/1*H6NN_RxZNeVyLYpCirsslg.png)

*特别感谢[@ryanqnorth](https://twitter.com/ryanqnorth)的[恐龙漫画](http://www.qwantz.com/)，它在2003年(恐龙统治网络时)以来提供了一些最好的荒诞幽默。*
</font>
















