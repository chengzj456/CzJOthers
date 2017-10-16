# iOS 自动化构建说明文档

## 背景

本公司 app开发是由Cordova+原生混合开发，由于打包时需要打h5包和原生包，太过于繁琐，因此开发出这套自动化构建脚本。目前该脚本只能对单个租户自动化构建，一次构建多租户为下一个研究的方向。

## 环境搭建

### 基础要求

* Cordova 5.4.1
* Ionic 2.2.1
* 请务必遵循Cordova和Ionic的版本，版本不对可能导致白屏

### 配置开发环境

* Xcode 8 

### 构建流程

![iOS 自动打包流程](https://github.com/chengzj456/CzJimages/blob/master/iOS%20自动打包流程.png?raw=true)

*  设置租户相关的 APP KEY

  ```shell
  exe-app-ios/tools/conf 目录下修改租户 id 命名的 sh和xml两个文件
  sh文件: 配置app版本和h5版本
  xml文件: 配置 微信、友盟、极光、高德
  ```

- 从git上拉取最新sp 复制到exe-app目录下

- 若为新租户

  - exe-app-ios/tools/conf 下创建 Id.sh Id.xml 两个文件

    ```shell
    export APP_VERSION=app版本
    export H5_VERSION=h5版本
    export GULP_CMD="gulp native -m -t="租户ID"
    export DEV_PROFILE=dev_租户ID
    export DIST_PROFILE=dist_租户ID
    ```


  - xml

    ```xml
    <?xml version="1.0" encoding="UTF-8" ?>
    <configuration>
    <target name="租户ID">
        <property name="appId" value="com.xmexe.租户ID"/>
        <property name="jpush.appId" value="极光key" />
        <property name="wechat.appId" value="微信key"/>
        <property name="info.plist" value="租户ID-info.plist"/>
        <property name="amap" value="高德key" />
    </target>
    </configuration>
    ```

  - 生成icon和splash：将 UI 给的 `icon` 图标和 `splash` 图放到该目录下 `exe-app/resources` 目录下
    `icon: 1024*1024` 、`splash: 2208*2208` 。

    从git上拉取最新materials 复制到exe-app目录下：

    ```shell
    $ cd exe-app-ios/exe-app/resources 
    $ ionic resources 
    ```

    > 生成完后将 resources 这个文件拷贝到 `exe-app-ios/exe-app/materials/` 租户 ID/master 替换resources

- 执行自动打包脚本 传入打包租户ID

  ```Shell
  $ ./zxl-build.sh $1 
  ```


* 选择打包类型，自动构建Ipa，生成的Ipa文件自动保存在桌面

  ```shell
  AppStore 为苹果商店包
  Enterprise 为企业账号包
  ```


* 自动上传 (目前只支持 Fir)

  ```Shell
  目前脚本内的fir的token为公司账号，若需变更fir账号
  打开/exe-app-ios/exe-app/platforms/ios/ExeAutoPackageScript/ExeAutoPackageScript.sh 文件
  修改代码:
  fir publish $export_ipa_path/$ipa_name.ipa -T @"你的token"
  ```


### 功能实现

1.命令行输入 ./build.sh exedev exedev 为租户ID

   这行命令调用的是build.sh 这个文件 

```sh
//$1 为传入的ID参数
$ ./zxl-build.sh $1  
接着调用 zxl-build.sh文件，传入参数$1
```
2.以下为zxl-build.sh代码

```sh
# 获取文件根目录
PWD=$(dirname $0)
if [ "$PWD"  ==  "." ]; then
    export PWD=`pwd`
fi
export ZXL_HOME=$(dirname $PWD)
export ZXL_TOOLS=$ZXL_HOME/tools
echo "ZXL_HOME: ${ZXL_HOME}"

# 判断传入的参数个数是否小于1
if [ $# -lt 1 ]; then
 echo "error"
 echo "usage: cmd <target> [version] [html5_version]"
 exit
fi

# 判断ID.sh是否存在
export target=$1
ENV_FILE=$ZXL_TOOLS/conf/$1.sh
if ! [ -f $ENV_FILE ]; then
    echo "$ENV_FILE not exist"
    exit
fi

# 读取ID.sh参数
source $ENV_FILE
# 参数如果有版本，取参数上设定的
if [ $# -gt 1 ]; then
    #app版本
    export APP_VERSION=$2
fi
if [ $# -gt 2 ]; then
    #h5版本
    export H5_VERSION=$3
fi

#echo $APP_VERSION

# 执行 zxl-config.sh
echo "=== start build target: $1 ==="
cd $ZXL_TOOLS
if [ "${ZXL_NO_CONFIG}" != "1" ]; then
    ${ZXL_TOOLS}/zxl-config.sh
fi

echo "=== $(date) build ${target} success ==="
```
3.以下为zxl-config.sh代码

```sh
echo "参数设置："
echo "目录:${ZXL_HOME}"
echo "verson:${APP_VERSION}"
echo "target:${target}"
echo "html_version:${H5_VERSION}"

cd $ZXL_HOME/exe-app

#删除中间产生的代码
rm -rf pack
rm -rf www

#step:pack html5
if [ "${ZXL_NO_PACK}" != "1" ];
then
    echo "*** start gulp pack ***"
    $GULP_CMD
else
    echo "ZXL_NO_PACK:${ZXL_NO_PACK}"
fi

#step:ionic_build
if [ "${ZXL_NO_IONIC_BUILD}" != "1" ];
then
    #rm -fR platforms/ios/build/*
    echo "*** start ionic build ios ***"
    ionic build ios
else
    echo "ZXL_NO_IONIC_BUILD:${ZXL_NO_IONIC_BUILD}"
fi

#step:更改版本信息
echo '*** 更改版本信息 ***'
java -jar ${ZXL_TOOLS}/zxlbuild.jar \
    -f ${ZXL_TOOLS}/conf/${target}.xml \
    -v ${APP_VERSION} \
    -t ${target} \
    -p ${ZXL_HOME}/exe-app/platforms/ios

if [ -n $H5_VERSION ]
then
	echo $H5_VERSION > ${ZXL_HOME}/exe-app/platforms/ios/www/js/versionConfig.txt
fi

#icons cp 更改app图标
cp -r resources/ios/icon/ platforms/ios/exe/Resources/icons

echo '=== 配置版本信息 success ==='

echo '*** 配置第三方key ***'
cd $ZXL_HOME/tools
sh changekey.sh ${target}

echo '*** 开始自动打包 ***'
cd $ZXL_HOME/exe-app/platforms/ios/ExeAutoPackageScript
sh ExeAutoPackageScript.sh ${target}
```
4.xcode自动打包流程

```sh
# 计时
echo "=== 开始打包:$1 ==="
SECONDS=0
# 是否编译工作空间 (例:若是用Cocopods管理的.xcworkspace项目,赋值true;用Xcode默认创建的.xcodeproj,赋值false)
is_workspace="false"
# 指定项目的scheme名称
# (注意: 因为shell定义变量时,=号两边不能留空格,若scheme_name与info_plist_name有空格,脚本运行会失败,暂时还没有解决方法,知道的还请指教!)
scheme_name=$1
# 工程中Target对应的配置plist文件名称, Xcode默认的配置文件为Info.plist
info_plist_name="$1-info"
# 指定要打包编译的方式 : Release,Debug...
build_configuration="Release"


# ===============================自动打包部分(无特殊情况不用修改)============================= #

# 导出ipa所需要的plist文件路径 (默认为AdHocExportOptionsPlist.plist)
ExportOptionsPlistPath="./ExeAutoPackageScript/AdHocExportOptionsPlist.plist"
# 返回上一级目录,进入项目工程目录
cd ..
# 获取项目名称
project_name=`find . -name *.xcodeproj | awk -F "[/.]" '{print $(NF-1)}'`
# 获取版本号,内部版本号,bundleID
info_plist_path="$project_name/$info_plist_name.plist"
bundle_version=`/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" $info_plist_path`
bundle_build_version=`/usr/libexec/PlistBuddy -c "Print CFBundleIdentifier" $info_plist_path`
bundle_identifier=`/usr/libexec/PlistBuddy -c "Print CFBundleVersion" $info_plist_path`

# 删除旧.xcarchive文件
rm -rf ~/Desktop/$scheme_name-IPA/$scheme_name.xcarchive
# 指定输出ipa路径
export_path=~/Desktop/$scheme_name-IPA
# 指定输出归档文件地址
export_archive_path="$export_path/$scheme_name.xcarchive"
# 指定输出ipa地址
export_ipa_path="$export_path"
# 指定输出ipa名称 : scheme_name + bundle_version
ipa_name="$scheme_name-v$bundle_version"

# AdHoc,AppStore,Enterprise三种打包方式的区别: http://blog.csdn.net/lwjok2007/article/details/46379945
echo "\033[36;1m请选择打包方式(输入序号,按回车即可) \033[0m"
echo "\033[33;1m1. AdHoc       \033[0m"
echo "\033[33;1m2. AppStore    \033[0m"
echo "\033[33;1m3. Enterprise  \033[0m"
echo "\033[33;1m4. Development \033[0m"
# 读取用户输入并存到变量里
read parameter
sleep 0.5
method="$parameter"

# 判读用户是否有输入
if [ -n "$method" ]
then
    if [ "$method" = "1" ] ; then
    ExportOptionsPlistPath="./ExeAutoPackageScript/AdHocExportOptionsPlist.plist"
    elif [ "$method" = "2" ] ; then
    ExportOptionsPlistPath="./ExeAutoPackageScript/AppStoreExportOptionsPlist.plist"
    elif [ "$method" = "3" ] ; then
    ExportOptionsPlistPath="./ExeAutoPackageScript/EnterpriseExportOptionsPlist.plist"
    elif [ "$method" = "4" ] ; then
    ExportOptionsPlistPath="./ExeAutoPackageScript/DevelopmentExportOptionsPlist.plist"
    else
    echo "输入的参数无效!!!"
    exit 1
    fi
fi

echo "\033[32m*************************  开始构建项目  *************************  \033[0m"
# 指定输出文件目录不存在则创建
if [ -d "$export_path" ] ; then
echo $export_path
else
mkdir -pv $export_path
fi

# 判断编译的项目类型是workspace还是project
if $is_workspace ; then
# 编译前清理工程
xcodebuild clean -workspace ${project_name}.xcworkspace \
                 -scheme ${scheme_name} \
                 -configuration ${build_configuration}

xcodebuild archive -workspace ${project_name}.xcworkspace \
                   -scheme ${scheme_name} \
                   -configuration ${build_configuration} \
                   -archivePath ${export_archive_path}
else
# 编译前清理工程
xcodebuild clean -project ${project_name}.xcodeproj \
                 -scheme ${scheme_name} \
                 -configuration ${build_configuration}

xcodebuild archive -project ${project_name}.xcodeproj \
                   -scheme ${scheme_name} \
                   -configuration ${build_configuration} \
                   -archivePath ${export_archive_path}
fi

#  检查是否构建成功
#  xcarchive 实际是一个文件夹不是一个文件所以使用 -d 判断
if [ -d "$export_archive_path" ] ; then
echo "\033[32;1m项目构建成功 🚀 🚀 🚀  \033[0m"
else
echo "\033[31;1m项目构建失败 😢 😢 😢  \033[0m"
exit 1
fi

echo "\033[32m*************************  开始导出ipa文件  *************************  \033[0m"
xcodebuild  -exportArchive \
            -archivePath ${export_archive_path} \
            -exportPath ${export_ipa_path} \
            -exportOptionsPlist ${ExportOptionsPlistPath}
# 修改ipa文件名称
mv $export_ipa_path/$scheme_name.ipa $export_ipa_path/$ipa_name.ipa

# 检查文件是否存在
if [ -f "$export_ipa_path/$ipa_name.ipa" ] ; then
echo "\033[32;1m导出 ${ipa_name}.ipa 包成功 🎉  🎉  🎉   \033[0m"
open $export_path
echo "\033[32m*************************  开始上传fir  *************************  \033[0m"
fir publish $export_ipa_path/$ipa_name.ipa -T 51fadfb8910de093d1ab21d4dbbff77c
else
echo "\033[31;1m导出 ${ipa_name}.ipa 包失败 😢 😢 😢     \033[0m"
exit 1
fi
# 输出打包总用时
echo "\033[36;1m打包总用时: ${SECONDS}s \033[0m"
```