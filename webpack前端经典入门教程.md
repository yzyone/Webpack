## webpakc 构建工具

一、安装

    1）npm install --save-dev webpack

    2）npm install --save-dev webpack-cli

    注意：

    1）建议安装到项目里

    2）4.x及以上的版本，需要再安装webpack-cli

二、webpakc的配置文件

    1、entry    入口文件

        1）只打包一个文件（一个入口），写字符串

        2）把多个文件打包成一个文件，写个数组

        3）把多个文件分别打包成多个文件，写成对象

            （webpack把打包后的文件叫Chunck)

    2、output    出口文件

        https://www.webpackjs.com/configuration/output/)

        1）filename    输出文件的名称

                 A、输出一个文件，写个字符串

                 B、输出多个文件，文件名前面加个标识符（id/name/hash)

        2）path    输出文件的路径

                A、路径必须位绝对路径

                B、__dirname是nodejs里的一个模块，表示当前文件的绝对路径

                C、path为系统模块    直接引入后调用

                path.resolve(__dirname, ‘输出文件的路径’)

        3）module    模块

        4）plugins    插件

                A、每个插件都是一个模块，需要先安装，再引入

                B、在plugins里new一个，传入配置参数即可

        5）devServer    服务器

                A、安装

                        npm install webpack-dev-server -D

                B、配置

                        contentBase    服务器要访问的目录

                        host                    服务器的IP地址

                        port                    服务器的端口

                        open                   自动打开页面

                                a、引入webpakc模块const webpack=require('webpack');

                                b、加入插件new webpack.HotModuleReplacementPlugin();

                C、运行命令：webpack-dev-server

                        注意：这里可能会出现安装完成后执行webpack-dev-server提示不是内部命令

                        解决办法：

                                a、在全局里也安装一下

                                b、在scripts里写上"dev"："webpack-dev-server"

                 D、https://webpack.docschina.org/configuration/dev-server/

        6）mode    模式，设置代码的运行环境（开发模式、生产模式）（4.x新增的）

                A、development    开发环境

                        a）使用eval构建module，提升增量构建速度

                        b）不需要定义 

                                new webpack.DefinePlugin(

                                    {"process.env.NODE_ENV": JSON.stringify("development")}

                                )    //默认    development

                        c）默认开启    NamedModulesPlugin -> optimization.namedModules

                                使用模块热替换(HMR)时会显示模块的相对路径

                        d）有浏览器调试工具，开发阶段的详细错误日志和提示，快速和优化的增量构建机制

               B、production    生产环境

                        a）提供uglifyjs-webpack-plugin    代码压缩

                        b）不要定义

                                new webpack.DefinePlugin(

                                    {"process.env.NODE_ENV": JSON.stringify("production")}

                                )    //默认    production

                        c）默认开启 NoEmitOnErrorsPlugin -> optimization.noEmitOnErrors,

                                编译出错时跳出输出，以确保输出资源不包含错误

                        d）默认开启    ModeleConcatenationPlugin -> optimization.concatenateModules, wepack3添加的作用域提升（Scope Hoisting）






