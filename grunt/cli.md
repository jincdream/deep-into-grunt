# Grunt源码解析之'/grunt/cli.js'

源码路径：https://github.com/gruntjs/grunt/blob/master/lib/grunt/cli.js

该模块主要涉及Grunt中的[cli][]部分即命令行用户接口。截止到目前为止，Grunt的命令执行还是由命令行进行的，当然已知的是TooBug等高级用户已经在研究图形化界面的Grunt配置及操作，但即使是那样，[cli][]部分也是及其重要的，因为所有的图形化界面操作最后还是会调用命令行接口来进行最终操作。

## grunt内部调用模块

这里调用了[grunt.js][]模块，这也是Grunt最核心的模块（最终暴露给外界的模块）。

```javascript
var grunt = require('../grunt');
```
## node.js内部调用模块

在Node模块中，调用了非常常见的[path][]模块，用来处理和转换文件路径相关操作。

```javascript
var path = require('path');
```

## 外部调用模块：

这里调用了[nopt][]模块，该模块是一个__option parser__，主要是用来配置，管理以及解析命令行参数，这个模块在这里起到了举足轻重的作用。

也许看完了这个章节，你也会选择使用[nopt][]在你的产品或模块中担任参数选项解析的工作。

```javascript
var nopt = require('nopt');
```
## 模块内部代码解析

### cli变量

定义`cli`变量作为模块返回的对象，该变量实际上指向了一个接受`options`和`done`参数的方法。

___强烈建议先跳过这一节，直接进行到下一节，在后面的内容都完成后，再回到这一节。这也是我非常不喜欢现在Node.js中代码的很重要的原因，在读取很多代码时，代码并不是从上往下逐步推进的，很多在前十行就用到的变量和函数你会发现在最后十行才出现，这个可能也是`moduel.exports`的副作用之一___

```javascript
// This is only executed when run via command line.
var cli = module.exports = function(options, done) {
```

当`options`参数不为null时，通过[Object.keys][]得到`options`中的所有项目，并通过[Array.prototype.forEach][]方法对`options`中每个项目进行处理。

```javascript
  // CLI-parsed options override any passed-in "default" options.
  if (options) {
    // For each default option...
    Object.keys(options).forEach(function(key) {
```

如果遍历到的选项不是`cli.options`中的一员，将这个选项的键值对加入`cli.options`对象中。否则的话，如果遍历到的选项是`cli.options`中的成员，而且该选项在`cli.optlist`对象中的`type`为`Array`，则通过调用[Array.prototype.push][]方法将当前选项的值加入`cli.options`中的对应数组中。

```javascript
      if (!(key in cli.options)) {
        // If this option doesn't exist in the parsed cli.options, add it in.
        cli.options[key] = options[key];
      } else if (cli.optlist[key].type === Array) {
        // If this option's type is Array, append it to any existing array
        // (or create a new array).
        [].push.apply(cli.options[key], options[key]);
      }
    });
  }
```
__注意：这里的代码`[].push.apply(cli.options[key], options[key])` 并不等价于下面的格式:__

```javascript
cli.options[key].push(options[key]);
```
__而是等价于下面的格式：__

```javascript
Array.prototype.push.apply(cli.options[key], options[key])
```
__这里其实包含一个很大的技巧，就是通过[Array.prototype.push][]来达到数组`concat`的作用。__ 不明白发生了神马？说明还需要对[Function.prototype.apply][]加深理解啊。

众所周知，[Function.prototype.apply][]接受两个参数`fun.apply(thisArg, [argsArray])`，用下面这个简单的例子来说明：

```javascript
var a = [1,2,3];
var b = [4,5,6];
Array.prototype.push.apply(a,b);
//结果a值为[1,2,3,4,5,6]
```
当执行`Array.prototype.push.apply(a, b)`时，a为`thisArg`，b为参数列表（__注意：是参数列表__），所以代码实际执行如下：

```javascript
var a = [1,2,3];
a.push(4, 5, 6)
//结果a值为[1,2,3,4,5,6]
```

同样用试验来说明：

```javascript
//定义一个parsed变量，该变量模拟了最终的cli.options对象
>var parsed = { force: true, target: 'dev' }

//当对target项目执行下面的代码时
> [].push.apply(parsed["target"], ["cssmin","htmlmin"])
3
//在运行结束后，我们发现什么也没有变化
> parsed
{ force: true, target: 'dev' }

//接下来，我们为parsed变量添加一个tasks属性，属性值为空数组
> parsed.tasks = []
[]
> parsed
{ force: true, target: 'dev', tasks: [] }

//然后运行相同的代码如下
> [].push.apply(parsed["tasks"], ["cssmin","htmlmin"])
2
//查看结果，我们发现数组内容被append到tasks属性中
> parsed
{ force: true,
  target: 'dev',
  tasks: [ 'cssmin', 'htmlmin' ] }
>
```
同样的过程，我们来试验一下`cli.options[key].push(options[key])`:

```javascript
//定义一个parsed变量，该变量模拟了最终的cli.options对象
>var parsed = { force: true, target: 'dev' }

//当对target项目执行下面的代码时，出现异常
> parsed["target"].push(["cssmin","htmlmin"])
TypeError: Object dev has no method 'push'
    at repl:1:19
    at REPLServer.self.eval (repl.js:110:21)
    at Interface.<anonymous> (repl.js:239:12)
    at Interface.EventEmitter.emit (events.js:95:17)
    at Interface._onLine (readline.js:202:10)
    at Interface._line (readline.js:531:8)
    at Interface._ttyWrite (readline.js:760:14)
    at ReadStream.onkeypress (readline.js:99:10)
    at ReadStream.EventEmitter.emit (events.js:98:17)
    at emitKey (readline.js:1095:12)

//接下来，我们为parsed变量添加一个tasks属性，属性值为空数组
> parsed.tasks = []
[]
> parsed
{ force: true, target: 'dev', tasks: [] }

//然后在tasks项目运行代码如下
> parsed["tasks"].push["cssmin","htmlmin"])
1
//查看结果，我们发现数组内容被append到tasks属性中，但确是作为一个元素加入的
> parsed
{ force: true,
  target: 'dev',
  tasks: [ [ 'cssmin', 'htmlmin' ] ] }
>

```
重复上面的步骤测试`Array.prototype.push.apply`:

```javascript
//定义一个parsed变量，该变量模拟了最终的cli.options对象
>var parsed = { force: true, target: 'dev' }

//当对target项目执行下面的代码时
> Array.prototype.push.apply(parsed["target"], ["cssmin","htmlmin"])
3
//在运行结束后，我们发现什么也没有变化
> parsed
{ force: true, target: 'dev' }

//接下来，我们为parsed变量添加一个tasks属性，属性值为空数组
> parsed.tasks = []
[]
> parsed
{ force: true, target: 'dev', tasks: [] }

//然后运行相同的代码如下，结果与"[].push.apply"一致
> Array.prototype.push.apply(parsed.tasks, ["cssmin","htmlmin"])
2
> parsed
{ force: true,
  target: 'dev',
  tasks: [ 'cssmin', 'htmlmin' ] }
```
好了，关于这个问题就先到这里。如果还有兴趣的话，可以看看在jsperf上的[Array.prototype.push.apply vs concat](http://jsperf.com/array-prototype-push-apply-vs-concat/20)。

继续看源码，将`cli.tasks`和`cli.options`和`done`作为参数调用`grunt.tasks`方法，具体的关于`cli.tasks`以及`cli.options`都是什么，是怎么来的，还需要继续往下看。

```javascript
  // Run tasks.
  grunt.tasks(cli.tasks, cli.options, done);
};
```

后话：当你去查看在`C:\Users\username\AppData\Roaming\npm\node_modules\grunt-cli\bin\grunt`文件时（这个文件是安装好grunt-cli后，运行`grunt`命令所直接调用的文件。）。你会发现在这个文件的最后一行如下示：

```javascript
// Everything looks good. Require local grunt and run it.
require(gruntpath).cli();
```
知道`cli`为什么这么关键了吧，它是`grunt`命令行的入口。

### cli.optlist

在`cli`中指定`optlist`属性，用来存放所有的参数长选项。

通过观察可以发现，每个长选项最多有如下属性:

* `short`: 指定参数短选项。

* `info`: 表明该选项的用途。

* `type`: 该选项的值类型。

* `negate` : 该选项的默认值？

每个选项必有的属性为`info`以及`type`。

```javascript
// Default options.
var optlist = cli.optlist = {
  help: {
    short: 'h',
    info: 'Display this help text.',
    type: Boolean
  },
  base: {
    info: 'Specify an alternate base path. By default, all file paths are relative to the Gruntfile. (grunt.file.setBase) *',
    type: path
  },
  color: {
    info: 'Disable colored output.',
    type: Boolean,
    negate: true
  },
  gruntfile: {
    info: 'Specify an alternate Gruntfile. By default, grunt looks in the current or parent directories for the nearest Gruntfile.js or Gruntfile.coffee file.',
    type: path
  },
  debug: {
    short: 'd',
    info: 'Enable debugging mode for tasks that support it.',
    type: [Number, Boolean]
  },
  stack: {
    info: 'Print a stack trace when exiting with a warning or fatal error.',
    type: Boolean
  },
  force: {
    short: 'f',
    info: 'A way to force your way past warnings. Want a suggestion? Don\'t use this option, fix your code.',
    type: Boolean
  },
  tasks: {
    info: 'Additional directory paths to scan for task and "extra" files. (grunt.loadTasks) *',
    type: Array
  },
  npm: {
    info: 'Npm-installed grunt plugins to scan for task and "extra" files. (grunt.loadNpmTasks) *',
    type: Array
  },
  write: {
    info: 'Disable writing files (dry run).',
    type: Boolean,
    negate: true
  },
  verbose: {
    short: 'v',
    info: 'Verbose mode. A lot more information output.',
    type: Boolean
  },
  version: {
    short: 'V',
    info: 'Print the grunt version. Combine with --verbose for more info.',
    type: Boolean
  },
  // Even though shell auto-completion is now handled by grunt-cli, leave this
  // option here for display in the --help screen.
  completion: {
    info: 'Output shell auto-completion rules. See the grunt-cli documentation for more information.',
    type: String
  },
};

```
接下来定义了`aliases`和`known`变量，初始值均为空对象"{}"。

```javascript
// Parse `optlist` into a form that nopt can handle.
var aliases = {};
var known = {};
```
通过[Object.keys][]获得`optlist`变量中的所有长选项名，然后通过[Array.prototype.forEach]对所有选项名进行遍历并对`aliases`和`known`进行赋值。

```javascript
Object.keys(optlist).forEach(function(key) {
```
如果对应的`optlist`对象中的选项对象定义了`short`属性，则把`short`属性值作为键，"--"与属性名连接作为值，添加到`aliases`对象中。

```javascript
  var short = optlist[key].short;
  if (short) {
    aliases[short] = '--' + key;
  }
```
同时将属性名作为键，属性名对应的属性对象的`type`值作为值，添加到`known`对象中。

```javascript
  known[key] = optlist[key].type;
});
```
事实上通过以上代码后，`aliases`对象和`known`对象中内容如下：

```javascript
> aliases
{ h: '--help',
  d: '--debug',
  f: '--force',
  v: '--verbose',
  V: '--version' }

> known
 {help: Boolean,
  base: path,
  color: Boolean,
  gruntfile: path,
  debug: [Number, Boolean],
  stack: Boolean,
  force: Boolean,
  tasks: Array,
  npm: Array,
  write: Boolean,
  verbose: Boolean,
  version: Boolean,
  completion: String}

```
很明显，这里的`aliases`和`known`都是为了使用[nopt][]所准备的，`known`对象包含所有已知的长选项名以及对应的参数值类型，`aliases`对象存放着所有的短选项名与对应的长选项。[process.argv][]是一个包含有命令行参数的数组，数组第一个项目为`node`，第二个项目对于`grunt`来说应该是`grunt`的执行文件，在我的机器上如下（win32平台）

```
C:\Users\username\AppData\Roaming\npm\node_modules\grunt-cli\bin\grunt
```
参数列表中的`2`表示从`process.argv`数组下标2开始处理参数，即从`node`和`grunt`执行文件之后的参数。

```javascript
var parsed = nopt(known, aliases, process.argv, 2);
```

由`nopt`返回的对象会附带一个特殊的属性成员`argv`，这个对象包含着三个字段：

* `remain`: 包含着那些参数解析结束后，剩下的参数。比如：在`grunt uglify --target=dev`中的`uglify`。

* `origin`: 包含着所有原始的参数。

* `cooked`: 在通过将短选项进行扩展后的参数。

在通过[nopt][]得到了解析后的参数对象`parsed`后，会将`parsed.argv.remain`赋值给`cli.tasks`，正如上面分析的那样，如果还是上面那个例子的话，这里的`cli.tasks`将会是`uglify`
。

接下来，将`parsed`的对象赋值给`cli.options`，随后对`parsed.argv`进行删除。

很多同学不禁要问，这个`parsed`变量究竟包含哪些内容呢？我做了下面的试验，结果内容应该会让你秒懂的。

```javascript
> var parsed = nopt(known, aliases, ["node", "C:\Users\username\AppData\Roaming\npm\node_modules\grunt-cli\bin\grunt", "--force", "--target=dev", "build"], 2);

> parsed
{ force: true,
  target: 'dev',
  argv:
   { remain: [ 'build' ],
     cooked:
      [ '--force',
        '--target',
        'dev',
        'build' ],
     original:
      [ '--force',
        '--target=dev',
        'build' ],
     toString: [Function] } }
```

__这里将`parsed.argv`进行[delete][]并不会直接释放内存，它只是将变量间的引用进行打断破坏有利于内存回收。对于内存管理方面的内容可以查看[memory management](https://developer.mozilla.org/en-US/docs/JavaScript/Memory_Management)。__

```javascript
cli.tasks = parsed.argv.remain;
cli.options = parsed;
delete parsed.argv;
```
接下来，把在`optlist`中的参数值类型为`Array`，但是没有出现在此次调用中的选项加入`cli.options`对象，并将值设为空数组`[]`。

如果我们回过头去看`optlist`变量，我们会发现只有tasks和npm选项的`type`是`Array`。所以下面的代码主要是为了处理这两个选项，我们可以说对于所有情况`cli.options`中都会有`tasks`还有`npm`的属性字段，不同的是，如果我们在调用`grunt`的参数列表中用到这两个选项的话，它们的值会是通过[nopt][]处理后的值，如果不在调用时的选项列表中的话，它们的值为空数组。

```javascript
// Initialize any Array options that weren't initialized.
Object.keys(optlist).forEach(function(key) {
  if (optlist[key].type === Array && !(key in cli.options)) {
    cli.options[key] = [];
  }
});
```

[cli]: http://en.wikipedia.org/wiki/Command-line_interface "Command-line interface"
[grunt.js]: https://github.com/gruntjs/grunt/blob/master/lib/grunt.js "grunt.js"
[nopt]: https://github.com/npm/nopt "nopt"
[Object.keys]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/keys 
[Array.prototype.forEach]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach "Array.prototype.forEach"
[Array.prototype.push]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/push "Array.prototype.push"
[process.argv]: http://nodejs.org/api/process.html#process_process_argv "process.argv"
[delete]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/delete "delete"
[Function.protorype.apply]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply "Function.prototype.apply"
[path]: http://nodejs.org/api/path.html "path"
