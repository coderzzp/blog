###  写在前面
npm是每个现代的前端工程师都应该掌握的包管理工具，但是我们大多数时间都只是在git clone ->npm install ->npm start 三连，我自己也是在遇到一些坑之后才去解到npm背后的规则


### 当我们在npm install的时候，我们在install什么
废话，当然是包了，包简而言之是我们工程项目中所依赖的由广大开发者所提供的一个函数，或者一个类等等。  
一个npm 包最少需要一个package.json文件，这个文件会用来描述这个包的名称（在npm仓库中唯一），用途，版本，依赖包等等，官方当然不会让你手写这个文件啦，npm内置了npm init 这个方法
```shell
mkdir my-package && cd my-package
npm init
```
![image](https://user-images.githubusercontent.com/24691802/48976762-bde72380-f0c7-11e8-9054-5a9d368d7957.png)
如果暂时不care这些东西，可以使用npm init --yes 快速生成全部填写默认值的包
```shell
mkdir my-package && cd my-package
npm init --yes
```
这是我生成的package.json文件，注意到main这个字段了吗，这个字段描述的是
这个包的入口文件，假如你想对外输出一个函数，可以新建一个main.js并通过module.exports 导出
![image](https://user-images.githubusercontent.com/24691802/48976847-57fb9b80-f0c9-11e8-9df1-0e23c72b7b4d.png)

```javascript
//main.js
module.exports=function(){
   console.log('真香!')
}
```
发布之前我们还需要了解npm的版本管理机制：semver规范，semver 约定一个包的版本号必须包含3个数字，格式必须为 MAJOR.MINOR.PATCH, 意为 主版本号.小版本号.修订版本号
- MAJOR 对应大的版本号迭代，做了不兼容旧版的修改时要更新 MAJOR 版本号
- MINOR 对应小版本迭代，发生兼容旧版API的修改或功能更新时，更新MINOR版本号
- PATCH 对应修订版本号，一般针对修复 BUG 的版本号

ok，在了解了这些之后，我们可以开始发布了，注意npm 要求在 publish 之前，必须更新版本号 ，你可以选择手动更新版本号，也可以使用npm自带的命令`npm version major|minor|patch`来更新版本。
一切准备就绪之后，使用`npm publish`来发布我们的包

### npm install
npm install 有两种使用方式，一种是不带参数`npm install`，它会下载当前工程的所有依赖包到本地node_modules目录下，另一种是`npm install xxx`，表示我要给这个工程新增一个包依赖。npm install之后我们会在node_modules这个文件下找到所有我们下载的依赖包，然后在项目中requrie这个包即可使用。
### npm历史
为简单起见，我们假设应用目录为 app, 用两个流行的包 webpack, nconf 作为依赖包做示例说明。并且为了正常安装，使用了“上古” npm 2 时期的版本 webpack@1.15.0, nconf@0.8.5.
### npm 2
npm 2 在安装依赖包时，采用简单的递归安装方法。执行 npm install 后，npm 2 依次递归安装 webpack 和 nconf 两个包到 node_modules 中。执行完毕后，我们会看到 ./node_modules 顶层只含有这两个子目录。
![image](https://user-images.githubusercontent.com/24691802/48977878-188b7a00-f0de-11e8-82bc-a7cffca7c8c7.png)
这种做法的好处是结构非常清楚，每层包的依赖都是很明确的是，但缺点是路径有可能会很深，甚至会导致windows 文件系统中，文件路径不能超过 260 个字符长的错误，其次这些包很多包的版本重复，会有两个包依赖同一个版本的包这种情况，但还是需要重复安装，导致包的体积臃肿  
在我们的示例中就有这个问题，webpack 和 nconf 都依赖 async 这个包，所以在文件系统中，webpack 和 nconf 的 node_modules 子目录中都安装了相同的 async 包，并且是相同的版本`
![image](https://user-images.githubusercontent.com/24691802/48978670-f7308b00-f0e9-11e8-8c11-fde9eb35cb4e.png)

### npm 3 -扁平结构
![image](https://user-images.githubusercontent.com/24691802/48978760-a457d300-f0eb-11e8-95db-908e3dc66c27.png)
虽然这样一来 webpack/node_modules 和 nconf/node_modules 中都不再有 async 文件夹，但得益于 node 的模块加载机制，他们都可以在上一级 node_modules 目录中找到 async 库。所以 webpack 和 nconf 的库代码中 require('async') 语句的执行都不会有任何问题。  

这只是最简单的例子，实际的工程项目中，依赖树不可避免地会有很多层级，很多依赖包，其中会有很多同名但版本不同的包存在于不同的依赖层级，对这些复杂的情况, npm 3 都会在安装时遍历整个依赖树，计算出最合理的文件夹安装方式，使得所有被重复依赖的包都可以去重安装。  
npm 文档提供了更直观的例子解释这种情况：假如 package{dep} 写法代表包和包的依赖，那么 A{B,C}, B{C}, C{D} 的依赖结构在安装之后的 node_modules 是这样的结构：
```
A
+-- B
+-- C
+-- D
```
这里之所以 D 也安装到了与 B C 同一级目录，是因为 npm 会默认会在无冲突的前提下，尽可能将包安装到较高的层级。  
如果是 A{B,C}, B{C,D@1}, C{D@2} 的依赖关系，得到的安装后结构是：
```
A
+-- B
+-- C
   `-- D@2
+-- D@1
```
这里是因为，对于 npm 来说同名但不同版本的包是两个独立的包，而同层不能有两个同名子目录，所以其中的 D@2 放到了 C 的子目录而另一个 D@1 被放到了再上一层目录。  
很明显在 npm 3 之后 npm 的依赖树结构不再与文件夹层级一一对应了。想要查看 app 的直接依赖项，要通过 npm ls 命令指定 --depth 参数来查看：
```shell
npm ls --depth
```
### npm 5 
在讲述npm 5之前我们是使用package.json来下载依赖的，我们来思考这样一个🌰：
package A
```
{
  "name": "A",
  "version": "0.1.0",
  "dependencies": {
    "B": "<0.1.0"
  }
}
```
package B:
```
{
  "name": "B",
  "version": "0.0.1",
  "dependencies": {
    "C": "<0.1.0"
  }
}
```
and package C:
```
{
  "name": "C",
  "version": "0.0.1"
}
```
这个时候我们在A项目中install，下载的目录结构如下
【图片】
这个时候如果B@0.0.2发布了，我们对B项目的升级完全是无感知的，这时候假如B@0.0.2有一个bug，我们要将B包锁死在@0.0.1，我们想当然的可以在A项目中这么写：

package A
```
{
  "name": "A",
  "version": "0.1.0",
  "dependencies": {
    "B": "@0.0.1"
  }
}
```
这时候B是锁死在了0.0.1版本没错，但是如果这个bug是因为C@0.0.2的升级引起的，你锁死B的版本是没有用的，因为B对于C的包是不锁死的，发现了没有，如果仅仅使用package.json，你是无法控制所有的包版本的，这个时候：package.lock.json应运而生，他可以锁死你在项目中引用的所有包  
```javascript
{
    "name":  "app",
    "version":  "0.1.0",
    "lockfileVersion":  1,
    "requires":  true,
    "dependencies": {
        // ... 其他依赖包
        "webpack": {
            "version": "1.8.11",
            "resolved": "https://registry.npmjs.org/webpack/-/webpack-1.8.11.tgz",
            "integrity": "sha1-Yu0hnstBy/qcKuanu6laSYtgkcI=",
            "requires": {
                "async": "0.9.2",
                "clone": "0.1.19",
                "enhanced-resolve": "0.8.6",
                "esprima": "1.2.5",
                "interpret": "0.5.2",
                "memory-fs": "0.2.0",
                "mkdirp": "0.5.1",
                "node-libs-browser": "0.4.3",
                "optimist": "0.6.1",
                "supports-color": "1.3.1",
                "tapable": "0.1.10",
                "uglify-js": "2.4.24",
                "watchpack": "0.2.9",
                "webpack-core": "0.6.9"
            }
        },
        "webpack-core": {
            "version": "0.6.9",
            "resolved": "https://registry.npmjs.org/webpack-core/-/webpack-core-0.6.9.tgz",
            "integrity": "sha1-/FcViMhVjad76e+23r3Fo7FyvcI=",
            "requires": {
                "source-list-map": "0.1.8",
                "source-map": "0.4.4"
            },
            "dependencies": {
                "source-map": {
                    "version": "0.4.4",
                    "resolved": "https://registry.npmjs.org/source-map/-/source-map-0.4.4.tgz",
                    "integrity": "sha1-66T12pwNyZneaAMti092FzZSA2s=",
                    "requires": {
                        "amdefine": "1.0.1"
                    }
                }
            }
        },
        //... 其他依赖包
    }
}
```
因为这个文件记录了 node_modules 里所有包的结构、层级和版本号甚至安装源，它也就事实上提供了 “保存” node_modules 状态的能力。只要有这样一个 lock 文件，不管在那一台机器上执行 npm install 都会得到完全相同的 node_modules 结果。  
这就是 package-lock 文件致力于优化的场景：在从前仅仅用 package.json 记录依赖，由于 semver range 的机制；一个月前由 A 生成的 package.json 文件，B 在一个月后根据它执行 npm install 所得到的 node_modules 结果很可能许多包都存在不同的差异，虽然 semver 机制的限制使得同一份 package.json 不会得到大版本不同的依赖包，但同一份代码在不同环境安装出不同的依赖包，依然是可能导致意外的潜在因素。  

相同作用的文件在 npm 5 之前就有，称为 npm shrinkwrap 文件，二者作用完全相同，不同的是后者需要手动生成，而 npm 5 默认会在执行 npm install 后就生成 package-lock 文件，并且建议你提交到 git/svn 代码库中。
### 依赖版本升级
问题来了，在安装完一个依赖包之后有新版本发布了，如何使用 npm 进行版本升级呢？——答案是简单的 npm install 或 npm update，但在不同的 npm 版本，不同的 package.json, package-lock.json 文件，安装/升级的表现也不同。  
我们不妨还以 webpack 举例，做如下的前提假设:
- 我们的工程项目 app 依赖 webpack
- 项目最初初始化时，安装了当时最新的包 webpack@1.8.0，并且 package.json 中的依赖配置为: "webpack": "^1.8.0"
- 当前 webpack 最新版本为 4.27.1, webpack 1.x 最新子版本为 1.15.0  

具体表现如下：  
`下表为表述简单，省略了包名 webpack, install 简写 i, update 简写为 up`

|# | package.json (BEFORE) | node_modules (BEFORE) | package-lock (BEFORE) | command | package.json (AFTER) | package.json (AFTER)
|-- | -- | -- | -- | -- | -- | --
|a) | ^1.8.0 | @1.8.0 | @1.8.0 | i | ^1.8.0 | @1.8.0
|b) | ^1.8.0 | 空 | @1.8.0 | i | ^1.8.0 | @1.8.0
|c) | ^1.8.0 | @1.8.0 | @1.8.0 | up | ^1.15.0 | @1.15.0
|d) | ^1.8.0 | 空 | @1.8.0 | Up | ^1.8.0 | @1.15.0
|e) | ^1.15.0 | @1.8.0 (旧) | @1.15.0 | i | ^1.15.0 | @1.15.0
|f) | ^1.15.0 | @1.8.0 (旧) | @1.15.0 | up | ^1.15.0 | @1.15.0

### 最佳实践
- 使用 npm: >=5.1 版本, 保持 package-lock.json 文件默认开启配置
- 初始化：第一作者初始化项目时使用 npm install <package> 安装依赖包, 默认保存 ^X.Y.Z 依赖 range 到 package.json中; 提交 package.json, package-lock.json, 不要提交 node_modules 目录
- 初始化：项目成员首次 checkout/clone 项目代码后，执行一次 npm install 安装依赖包
- 不要手动修改 package-lock.json，当然也不要手动删除package-lock.json，除非你要升级所有包版本
- 升级依赖包:
  - 升级小版本: 本地执行 npm update 升级到新的小版本
  - 升级大版本: 本地执行 npm install <package-name>@<version> 升级到新的大版本
也可手动修改 package.json 中版本号为要升级的版本(大于现有版本号)并指定所需的 semver, 然后执行 npm install
  - 本地验证升级后新版本无问题后，提交新的 package.json, package-lock.json 文件
- 降级依赖包:
  - 正确: npm install <package-name>@<old-version> 验证无问题后，提交 package.json 和 package-lock.json 文件
  - 错误: 手动修改 package.json 中的版本号为更低版本的 semver, 这样修改并不会生效，因为再次执行 npm install 依然会安装 package-lock.json 中的锁定版本
- 删除依赖包:
  - Plan A: npm uninstall <package> 并提交 package.json 和 package-lock.json
  - Plan B: 把要卸载的包从 package.json 中 dependencies 字段删除, 然后执行 npm install 并提交 package.json 和 package-lock.json
  - 任何时候有人提交了 package.json, package-lock.json 更新后，团队其他成员应在 svn update/git pull 拉取更新后执行 npm install 脚本安装更新后的依赖包


