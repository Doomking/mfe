![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edc7ff2dce324f89b184c216affce516~tplv-k3u1fbpfcp-no-mark:480:480:0:0.awebp?)

# 1.引子


开发npm包的时候，难免要在本地进行调试，这个时候自然免不了使用npm link 将包link到项目中，在一般情况下，都是可以正常运行。

可是当npm包里面的依赖使用的是peerDependencies时候，会出现提示找不到依赖的包；而项目又没有暴露webpack等相关配置。

这时调试工作就无法进行了。

为了解决这个问题，我们就得先看看npm link和peerDependencies到底是如何工作的？

# 2.npm link

npm link使用软链接的方式，来实现将本地的npm包链接到项目中，使用过程包括2个命令：

-   npm link  


在本地npm包的目录下，执行npm link，npm会将当前的npm包软链接到npm 全局包安装目录。  


-   npm link package-name

在项目中，执行 npm link xxx，然后xxx包就链接到当前项目了，就可以正常使用了。

npm使用起来还是很简单方便的，卸载也很方便，npm unlink即可。
    
# 3.peerDependencies
    
   dependencies 和 devDependencies是非常常见的，一般的项目都会存在，但是peerDependencies相对就比较少见，没有开发过npm包的同学就会感到比较陌生；
    
   peerDependencies 即是对等依赖的意思，其中的包，在 npm install 时并不会被安装；同时打包项目时，其中的包也不会被打包进去。
    
   peerDependencies 中的包是没有显式依赖的，它默认库的使用方项目中已经安装过相关依赖，但是它不会自动检测并安装。通常这能保证一个项目中，对一个包的多次引用时，引用的是同一个对象或实例，能确保代码正确运行。
    
# 4.问题
    
   当peerDependencies遇到npm link的时候，就会发生一些事情比如项目ProjectA依赖包PackageB和DepC这2个包；其中DepC包是个第三方，而ProjectA和PackageB是自己本地维护的代码；三者形成了如下的关系；
    ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc353953863a48f5a5b69ceb2922ed08~tplv-k3u1fbpfcp-zoom-1.image)
    
   同时由于DepC包会导出一个实例，ProjectA和PackageB需共享DepC的同一个实例，否则系统运行不正常；
    
   因此PackageB的包里对DepC的依赖，必须要使用peerDependencies，否则ProjectA和PackageB各自引用DepC包就会出现不使用同一个实例，导致运行不达预期；
    
   然后在PackageB的目录里面执行 npm link，成功后，在ProjectA的目录中执行 npm link PackageB，同时也安装了DepC包，但是在打包构建时，却出现找不到DepC包的错误，明明已安装，怎么还会找到到呢？
    
   仔细查看错误信息，发现报错的目录指向PackageB目录，由于PackageB目录与ProjectA目录是兄弟目录，而DepC包安装到了ProjectA的node_modules目录中，在PackageB目录中的文件构建时，无法在当前目录及父级以及全局中找到DepC包，从而提示无法找到DepC包。
    
   问题的原因就在于npm link使用的是软链接的方式，全局的PackageB包只是实际PackageB目录的快捷方式，因此导致构建时DepC包无法找到。
    
# 5.yalc
    
   由于npm link及yarn link等都是采用软链接的方式，想解决这个问题最好是能把PackageB直接安装到ProjectA的目录中，但是发版又太繁琐了；
    
   解决方案就是yalc（https://github.com/wclr/yalc）：对包开发者而言，一种比yarn/npm link更好的开发流程
    
   它的主要对标者就是yarn/npm link，它主要解决了一些yarn/npm link本身存在的缺陷，满足了包开发者的实际需求。
    
   yalc主要有2个命令：

-   yalc publish
    
    在Package B的目录下执行yalc publish，同时会逐一执行npm 生命周期脚本，如：prepublish, prepare, prepublishOnly, prepack, preyalcpublish等  



-   yalc add  
    在ProjectA的目录下执行yalc add PackageB，然后就可以正常运行了  


yalc解决此问题的方式是，在yalc add命令执行时，将PackageB包的内容复制到ProjectA目录下的.yalc目录下，即ProjectA/.yalc/PackageB这样的目录，同时在package.json文件中注入对PackageB包的依赖：

```
{  "PackageB": "file:.yalc/PackageB"}
```

这样PackageB包就安装到了ProjectA目录下，解决了DepC包无法找到的问题。