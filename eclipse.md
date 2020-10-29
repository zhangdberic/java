# eclipse

## eclispe升级

1.添加新的更新源

Windows - Preferences - Install/Update - Available Software

这里使用的是国内东软信息学院的镜像地址(http://mirrors.neusoft.edu.cn/eclipse/releases/)

Name：2020-06
Location：http://mirrors.neusoft.edu.cn/eclipse/releases/2020-06/
**并且去掉所有其他的更新源**

2.再执行
Help - Check for Updates

## maven插件降级

eclipse 202003版本以后的maven插件使用的1.5的版本，在执行mvn deploy发布程序到maven仓库的时候，会报错401没有授权，尽管我们能保证用户、密码和权限的正确性，但还是提示401无授权。这个是新版eclipse附带maven插件的问题。这里的解决方法是降级maven插件，安装使用eclipse 2019-12版本使用的maven1.4插件。

具体操作如下：

1.删除自带的maven 1.5插件。

步骤：help->eclipse marketplace->find(搜索关键字maven)->选择"Maven Integration for Eclipse(luna) 1.5.0"->点击unstall按钮。插件卸载后重新启动eclipse。

2.安装maven1.4插件。

步骤：

help->Install New Software->Manager...

选择Enable按键，先禁用2020-09。

点击add按钮，增加2019-12，http://download.eclipse.org/releases/2019-12

返回，选择，Work with：2019-12 - http://download.eclipse.org/releases/2019-12，并搜索m2e，选中：General Purpose Tools，下面应该同步选择三个m2e插件，    

m2e - Extensions Development Support (Optional)	1.14.0.20191209-1925

m2e - Maven Integration for Eclipse (includes Incubating components)	1.14.0.20191209-1925

m2e - slf4j over logback logging (Optional)	1.14.0.20191209-1925

点击next按钮->install按钮，开始安装。

重启eclipse，并且恢复2020-09，并禁用2019-12













