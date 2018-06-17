<font face="Times New Roman">
<h1>Modern JavaScript Explained For Dinosaurs</h1>

![](https://cdn-images-1.medium.com/max/1600/1*H8PH-HaV43gZyBJz0mJHxA.png)
中途转过来学习当代JavaScript是一件很难的事情，它的生态系统在快速地发展与改变，以至于你很难理解它众多的工具都是被用来解决什么样的问题。我从1998年开始变成，但是直到2014年，我才从真正意义上开始学习JavaScript。我记得当时我看到[Browserify](http://browserify.org/)，并盯着它的标语：

> “Browserify lets you require('modules') into the browser by bundling up all of your dependencies.”

我几乎不理解这句话中的任何一个词，却还要挣扎着去思考这句对作为开发者的我会有什么样的帮助。

本文的目的是提供一个JavaScript工具如何发展到今天(2017年)的历史背景。我们讲从零开始，构建一个不借助任何工具，只需要简单的HTML和JavaScript的示例网站。接着我们将依次引入不同的工具，并解释它解决了什么样的问题。有了这种历史背景，您将能更好地学习和适应未来不断变化的JavaScript格局。让我们开始吧！

### JavaScript的"老派"使用方式
让我们从一个只使用HTML和JavaScript的“老派”网站开始，它需要我们手动地下载和链接文件。这里是一个简单的 ```index.html``` 文件，并链接了一个JavaScript文件：

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

着就是您制作一个网站所需的一切。现在，假设我们想要添加一个第三方的库，如[moment.js](http://momentjs.com/)(一个可以将日期格式化为人类可读方式的库)。如下所示，你可以在JavaScript使用 ```moment``` 提供的功能：

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

注意，```moment.min.js```在```index.js```之前加载，这意味着我们可以在```index.js```中使用moment的功能，如下所示：

```javascript
// index.js
console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());
```

这就是以前我们使用JavaScript库制作网站的方式，它的好处是很容易理解；而坏的一面是，当JavaScript库更新时，我们需要重新寻找并下载新版本的JavaScript库，这很烦人。

### 使用JavaScript包管理器(npm)

大概从2010年开始，出现了几个相互之间具有竞争关系的包管理器，它们帮助开发者自动化地完成从中央存储库下载和升级库的过程。在2013年左右，[Bower](https://bower.io/)毫无疑问是最受欢迎的包管理器，最终在2015年左右，它被[npm](https://www.npmjs.com/)取代。(值得注意的是，从2016年年末开始，[yarn](https://yarnpkg.com/en/)作为npm的替代品，已经获得了很大的吸引力，但yarn背后依然是npm。)

请注意，npm最初是专门为node.js制作的包管理器，node.js是一个旨在运行在服务器而非浏览器上的JavaScript运行时。所以使用npm作为一个前端包管理器来管理运行在浏览器上的库，是一个非常奇怪的选择。

*__注意：__使用包管理器通常会涉及命令行的使用，而在过去，这是一个前端开发者从不需要的技能。如果你从未使用过命令行，这里有一个很好的[入门教程](https://www.learnenough.com/command-line-tutorial)。无论好与坏，知道如何使用命令行是当代JavaScript开发的重要组成部分(同时它也为你在其他领域下的开发打开了一扇大门)。*

下面让我们来看看如何通过npm，而非手动下载的方式来安装moment.js。如果你已经安装了node.js，你也已经安装了npm，这意味着你可以在命令行中导航到```index.html```所在的文件夹并输入如下命令：

```
$ npm init
```

这将提示你几个问题(一般情况下选默认值即可)并创建一个名为```package.json```的新文件。```package.json```是一个配置文件，npm用它来存储所有项目信息。默认情况下，```package.json```的内容应该是这样：

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

这句命令做了两件事－－首先，它将moment.js包中的所有代码下载到一个名```node_modules```的文件夹中。其次，它自动修改 ```package.json```，并将moment.js作为项目依赖记录下来。

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

这在与他人共享项目时非常有用－－只需要共享```package.json```文件，而非整个 ```node_modules```文件夹(它可能会变得非常大)，其他开发者可以通过 ```npm install```命令来自动安装项目需要的软件包。

所以，现在我们再也不需要从网站手动下载moment.js了，我们可以通过使用npm来自动下载并更新它。查看 ```node_modules```文件夹，我们可以在 ```node_modules/moment/min```文件夹下找到 ```moment.min.js```文件，这意味着我们可以在 ```index.html```中链接通过npm下载的 ```moment.min.js```：

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

因此，好的一面是，通过命令行，我们可以使用npm来下载和更新我们的依赖包；坏的一面是，我们需要深入到 ```node_modules```文件夹内部来寻找每个依赖包的位置，并需要手动将它们添加到HTML中。这非常麻烦，接下来我们将看看如何自动化地执行这个过程。

![](https://cdn-images-1.medium.com/max/1600/1*GeEETvRqyG4o7SZdbU2Guw.png)

### 使用JavaScript模块打包工具(webpack) 

大部分编程语言都提供了将代码从一个文件import到另一个文件的方法。JavaScript最初被设计为只能在浏览器中运行，出于安全的原因，它不能访问客户计算机的文件系统，因此JavaScript没有这一特性。在很长的一段时间内，在多个文件中组织JavaScript代码需要使用全局共向变量来加载每个文件。

在上述moment.js例子中，我们就是这么干的－－整个 ```moment.min.js```文件被加载到HTML中，该HTML定义了一个叫 ```moment```的全局变量，然后在 ```moment.min.js```之后加载的任何文件都可以使用这个变量(无论这个文件是否需要使用到这个变量)。

在2009年，一个名为CommonJS的项目启动，它的目的定义一个在浏览器之外的JavaScript生态系统。CommonJS的一个重要组成部分就是它的模块规范，它最终允许JavaScript像大多数编程语言一样可以跨跃文件导入和导出代码，而无需借助全局变量。CommonJS模块规范最著名的实现是node.js。

![](https://cdn-images-1.medium.com/max/1600/1*xeF1flp1zDLLJ4j7rDQ6-Q.png)

正如上文提到的，node.js是一个运行在服务器上的JavaScript运行时。在使用node.js的module来改造前面的例子时，你可以直接在JavaScript文件中加载 ```moment.min.js```，而不需要使用HTML的script标签，代码如下：

```javascript
// index.js
var moment = require('moment');
console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());
```

这就是node.js加载模的方式，它效果很好，因为node.js是一门可以访问计算机文件系统的服务器端语言。同时node.js也知道每个npm模块路径的位置，所以加载模块时，我们可以简单地写成 ```require('moment')```，而不需要写 ```require('./node_modules/moment/min/moment.min.js')```。

这一切在node.js上都能很好地运行，当时当你尝试在浏览器中之行上述代码时，浏览器会报出一个 ```require```未被定义的错误。加载文件必须以同步(会降低执行速度)或异步(可能有时间问题)的方式动态完成，而浏览器不能访问计算机的文件系统，这意味着以这种方式加载模块非常的棘手。

这时我们就需要使用到模块打包工具。JavaScript模块打包工具，通过添加一步叫做build(可以访问计算机文件系统)的操作，创建与浏览器兼容的最终输出(不需要访问文件系统)，来解决上述问题。在这种场景下，我们需要用模块打包工具来找到所有的 ```require```(这是无效的浏览器JavaScript语法)语句，并将其替换为每个所需文件的实际内容。最终的输出结果是一个打包好的JavaScript文件(没有require语法)。

[Browserify](http://browserify.org/)在2011年发布后，即成为最受欢迎的Module Bundler，同时它率先在前端使用node.js样式的require语句(这也是使npm成为前端包管理器之一的原因)。在2015年左右，受React前端框架普及的推动，[webpack](https://webpack.github.io/)最终成为更广泛使用的模块打包程序，React框架充分利用了webpack的各种功能。

让我们来看看如何使用webpack来使 ```require('moment')```在浏览器中正常运行。首先，我们需要将webpack安装到项目中，webpack本身就是一个npm包，所以我们可以从命令行安装它：

```
$ npm install webpack --save-dev
```

请注意 ```--save-dev```参数，webpack将会保存为开发依赖项，这意味着它是在开发环境而非生产环境需要的软件包，你可以从 ```package.json```中看出这一区别：

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

这句命令会运行安装在 ```node_modules```文件夹下的webpack工具，它将从 ```index.js```文件开始，查找所有的 ```require```语句，并将它们替换为对应的代码，最后创建了名为 ```bundle.js```的单个输出文件。我们将不再在浏览器中使用包涵无效 ```require```语句的```index.js```文件了，相反，我们使用webpack输出的 ```bundle.js```文件，HTML改动如下：

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

如果你刷新浏览器，你会看到一切都像以前一样正常工作。

注意，每次修改 ```index.js```文件后，我们都需要运行上述webpack命令。这很枯燥，当我们使用webpack高级功能(例如[生成源码映射](https://webpack.js.org/guides/development/#using-source-maps)以帮助我们从转译后的代码调试原始代码)时，它会变得更加枯燥乏味。webpack可以从工程的根目录下一个名为 ```webpack.config.js```的文件中读取配置选项，在我们的例子中，```webpack.config.js```应该长这样：

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

我们不需要再指定```index.js```和 ```bundle.js```这两个选项了，因为webpack可以从 ```webpack.config.js```读取这些选项。相比之前，这样更好，但每次代码改变后都需要输入这行命令依然很枯燥。我们将会使这个过程变得更顺滑一点。

总体而言，这可能看起来并不多，但这个工作流程有一些巨大的优势。我们再也不用通过全局变量加载外部脚本了，任何新的JavaScript库都将通过JavaScript语言中的```require```语句添加，而不是通过在HTML中添加新的\<script\>标签。相对多个文件，单个bundle文件通常对性能更好。同时，既然现在我们在开发工作流程中添加了一个构建步骤，我们还可以添加一些其他强大功能！

![](https://cdn-images-1.medium.com/max/1600/1*ee_ivxNTKgIJTjmEMC4-dg.png)

### 转译代码以支持新的语言特性(babel)

转译代码意味着将一种语言的代码转换为另一种类似语言的代码。这是前端开发的一个重要组成部分－－因为浏览器添加新功能的速度非常缓慢，而带有实验性功能的新语言可以被转译成浏览器兼容的语言。

对于CSS，有[Sass](http://sass-lang.com/)，[Less](http://lesscss.org/)和[Stylus](http://stylus-lang.com/)等。对于JavaScript，曾经一段时间内最受欢迎的转换器是CoffeeScript(2010年左右发布)，而现在，大部分人都使用[babel](https://babeljs.io/)或[TypeScript](http://www.typescriptlang.org/)。CoffeeScript是一门通过大量修改JavaScript语言来提升JavaScript能力的一门语言。Babel本身不是一门新的语言，但是它是一个将浏览器尚未全部兼容的下一代JavaScript(ES6及以上)转换为更兼容的JavaScript(ES5)的转换器。Typescript是一种与下一代JavaScript基本相同的语言，但也增加了可选的静态类型。许多人选择使用babel，因为它最接近vanilla JavaScript。

让我们来看一个如何在现有的webpack构建步骤中使用babel的例子。首先我们通过下面的命令将babel安装到我们的工程中：

```
$ npm install babel-core babel-preset-env babel-loader --save-dev
```

注意，我们在开发依赖中安装了三个独立的包-- ```babel-core```是babel的主要组成部分，```babel-preset-env```预订了需要转换的JavaScript新特性，```babel-loader```是一个能使babel与webpack一起工作的软件包。我们通过修改 ```webpack.config.js```来使用```babel-loader```：

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

这个语法可能会让人感到困惑，不过幸运的是，我们不需要经常去改动它。基本原理是，我们告诉webpack去查找所有.js文件(包括 ```node_modules```中的.js文件)，然后通过```babel-loader```与```babel-preset-env```来进行转换。你可以在[这里](https://webpack.js.org/configuration/)看到配置webpack的详细语法。

现在一切都已搭建好，我们可以在JavaScript中使用ES2015的特性了。以下是使用[ES2015模版字符串](https://babeljs.io/docs/en/learn/#ecmascript-2015-features-template-strings)的示例：

```javascript
// index.js
var moment = require('moment');

console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());

var name = "Bob", time = "today";
console.log(`Hello ${name}, how are you ${time}?`);
```

我们还可以使用[ES2015 import](https://babeljs.io/docs/en/learn/#ecmascript-2015-features-modules)语句来替代require加载模块的功能。今天，你可以在很多代码库发现这种写法。

```javascript
// index.js
import moment from 'moment';

console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());

var name = "Bob", time = "today";
console.log(`Hello ${name}, how are you ${time}?`);
```

在这个例子中，import语法与require语法没有多大区别，但对于更高级的场景，import会更灵活。因为修改了 ```index.js```文件，我们需要再次运行这句命令：

```
$ ./node_modules/.bin/webpack
```
刷新浏览器后你会看到期望的效果。在写这篇文章的时候，许多当代浏览器都已经支持了ES2015的特性，所以很难分辨出babel是否起了作用。你可以在IE9等旧版浏览器中对其进行测试，你也可以在 ```bundle.js```查找转换后的这行代码：

```javascript
// bundle.js
// ...
console.log('Hello ' + name + ', how are you ' + time + '?');
// ...
```

这里你可以看到，babel将ES2015的模版字符串转换成了常规的JavaScript字符串拼接，以此来保证浏览器兼容性。尽管这个例子可能不太令人兴奋，但是能转译代码是一个非常强大的功能。我们现在就可以使用JavaScript即将要支持的新特性，如[async/await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)等，来编写更好的代码。尽管转译有时候会繁琐而痛苦，但它让我们能够在今天去测试将来的语言特性，这也使得JavaScript语言在过去几年里得到了显著的提升。

我们差不多完成了，但在我们的工作流中任然存在一些未完成的边界。如果我们关心性能，我们需要缩小([minifying](https://en.wikipedia.org/wiki/Minification_%28programming%29))打包后的文件，因为我们有了构建这一步，缩小代码也变得非常简单。每次修改JavaScript后，我们都需要重新运行webpack命令，接下来我们将讨论解决这一问题的便捷工具。

### 使用task runner(npm脚本)
既然我们已经使用了构建步骤来处理JavaScript模块，使用一个task runner来自动化各构建任务也变得非常合理。对于前端开发，任务包括缩小代码(minifying code)、优化图片，运行测试等。

在2013年，Grunt是最流行的前端前端task runner，随后Glup流行了一小段时间，-它们都是通过插件来包装其他命令行工具。而现在最流行的选择似乎是使用npm包管理器内置的脚本功能，该功能不需要依赖插件，而是直接与其他命令行工具一起使用。

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
这会使用```--watch```选项，它会在每次JavaScript文件更改时自动运行webpack，这对开发来说非常有用。

</font>




















