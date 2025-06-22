Oracle官网下载相关版本JDK版本安装包，如JDK8、JDK17：

安装后配置环境变量：

```bash
# 打开配置文件
vim ~/.bash_profile

# 配置JDK8、JDK17路径
export JAVA_8_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_201.jdk/Contents/Home
export JAVA_17_HOME=/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home

# 配置别名
alias jdk8="export JAVA_HOME=$JAVA_8_HOME"
alias jdk17="export JAVA_HOME=$JAVA_17_HOME"

# 默认JDK8
export JAVA_HOME=$JAVA_8_HOME
export PATH="$JAVA_HOME:$PATH"

# 保存配置内容
# 配置生效
source ~/.bash_profile
```

控制台切换版本&查看版本：

```bash
$ java -version
java version "1.8.0_201"
Java(TM) SE Runtime Environment (build 1.8.0_201-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.201-b09, mixed mode)

# 切换jdk17
$ jdk17
$ java -version

java version "17.0.14" 2025-01-21 LTS
Java(TM) SE Runtime Environment (build 17.0.14+8-LTS-191)
Java HotSpot(TM) 64-Bit Server VM (build 17.0.14+8-LTS-191, mixed mode, sharing)
```