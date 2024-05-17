---
title: 用 Shell 脚本实现自动化从 Unity 导出 iOS 工程、打包、上传
date: 2024-04-19 15:47:55 +0800
categories: [iOS]
---

## Shell 脚本

* Shell 脚本内容根据自己项目按需修改
* 如果将 Shell 脚本保存为 `XXX.sh` 文件，在 macOS 上的终端执行脚本即可完成所有操作：`sh XXX.sh`
* 如果将 Shell 脚本保存为 `XXX.command` 文件，只需双击文件，即可执行脚本，完成所有操作，首次需要先赋予执行权限，之后双击即可执行。
* 如果是不涉及 Unity 相关的纯 iOS 项目，把从 Unity 导出 iOS 工程相关的代码删去即可。

<br>

``` shell
#!/bin/sh

# 注意事项
# 1、如果提示permission denied: ./XXX.sh ， 则先附加权限，命令如下：chmod 777 XXX.sh
# 2、请根据自己项目的情况选择使用 workspace 还是 project 形式，目前默认为 workspace 形式

### 需要根据自己项目的情况进行修改

# 开始计时
start_time=$(date +%s)

# 版本更新说明
UpdateDescription="XXX"

# 本机 Unity 指定版本的路径，按实际修改
UNITY_DIR="/Applications/Unity/Hub/Editor/XXX/Unity.app/Contents/MacOS/Unity"

# Unity 工程的路径，可修改
UNITY_PROJECT_DIR="$(dirname "$(dirname "$0")")/UnityTest"

# Unity 导出 iOS 工程的保存路径，可修改
IOS_EXPORT_DIR=$(dirname "$0")/Unity/EX_iOS

# 这行命令末尾的两个值："outputPath" 是参数，"$IOS_EXPORT_DIR" 是参数对应的值。ExportProject.cs 脚本需要放在 Unity 工程的 Assets/Editor 目录下
echo "============ Unity Export iOS Project Begin ============"
$UNITY_DIR -quit -batchmode -projectPath $UNITY_PROJECT_DIR -executeMethod ExportProject.ExportIOS -outputPath $IOS_EXPORT_DIR

# 返回结果，如果结果不是0表示编译出现错误
exit_code=$?
if [ $exit_code -eq 0 ]; then
  echo "============ Unity Export iOS Project Success =========="
else
  echo "============ Unity Export iOS Project Fail ============="
  exit
fi

end_time_1=$(date +%s)

# 文件替换，看项目实际需要
SOURCE_FILE=$(dirname "$0")/XXX.mm
TARGET_FILE=$IOS_EXPORT_DIR/Classes/UI/XXX.mm
cp $SOURCE_FILE $TARGET_FILE


# Project名称
PROJECT_NAME=XXX

## Scheme名
SCHEME_NAME=XXX

## 编译类型 Debug/Release二选一
BUILD_TYPE=Release

## 项目根路径，xcodeproj/xcworkspace所在路径
PROJECT_ROOT_PATH=$(dirname "$0") # 使用 dirname 命令获取当前脚本所在的目录

## 打包生成路径
PRODUCT_PATH=${PROJECT_ROOT_PATH}/Build

## ExportOptions.plist文件的存放路径，该文件描述了导出ipa文件所需要的配置
## 如果不知道如何配置该plist，可直接使用xcode打包ipa结果文件夹的ExportOptions.plist文件
EXPORTOPTIONSPLIST_PATH=${PROJECT_ROOT_PATH}/Others/ExportOptions.plist


## workspace路径
WORKSPACE_PATH=${PROJECT_ROOT_PATH}/XXX.xcworkspace

## project路径
PROJECT_PATH=${PROJECT_ROOT_PATH}/${PROJECT_NAME}/${PROJECT_NAME}.xcodeproj

### 编译打包过程 ###

echo "============ Build Clean Begin ========================="

## 清理缓存

## project形式
# xcodebuild clean -project ${PROJECT_PATH} -scheme ${SCHEME_NAME} -configuration ${BUILD_TYPE} || exit

## workspace形式
xcodebuild clean -workspace ${WORKSPACE_PATH} -scheme ${SCHEME_NAME} -configuration ${BUILD_TYPE} || exit

echo "============ Build Clean End ==========================="

#获取Version
VERSION_NUMBER=`sed -n '/MARKETING_VERSION = /{s/MARKETING_VERSION = //;s/;//;s/^[[:space:]]*//;p;q;}' ${PROJECT_PATH}/project.pbxproj`
# 获取build
BUILD_NUMBER=`sed -n '/CURRENT_PROJECT_VERSION = /{s/CURRENT_PROJECT_VERSION = //;s/;//;s/^[[:space:]]*//;p;q;}' ${PROJECT_PATH}/project.pbxproj`

## 编译开始时间,注意不可以使用标点符号和空格
BUILD_START_DATE="$(date +'%Y-%m-%d_%H-%M')"

## IPA所在目录路径
IPA_DIR_NAME=${VERSION_NUMBER}_${BUILD_NUMBER}_${BUILD_START_DATE}

##xcarchive文件的存放路径
ARCHIVE_PATH=${PRODUCT_PATH}/IPA/${IPA_DIR_NAME}/${SCHEME_NAME}.xcarchive
## ipa文件的存放路径
IPA_PATH=${PRODUCT_PATH}/IPA/${IPA_DIR_NAME}

# 解锁钥匙串 -p后跟为电脑密码
security unlock-keychain -p XXX

echo "============ Build Archive Begin ======================="
## 导出archive包

## project形式
# xcodebuild archive -project ${PROJECT_PATH} -scheme ${SCHEME_NAME} -archivePath ${ARCHIVE_PATH}

## workspace形式
xcodebuild archive -workspace ${WORKSPACE_PATH} -scheme ${SCHEME_NAME} -archivePath ${ARCHIVE_PATH} -quiet || exit

echo "============ Build Archive Success ====================="


echo "============ Export IPA Begin =========================="
## 导出IPA包
xcodebuild -exportArchive -archivePath $ARCHIVE_PATH -exportPath ${IPA_PATH} -exportOptionsPlist ${EXPORTOPTIONSPLIST_PATH} -quiet || exit

if [ -e ${IPA_PATH}/${SCHEME_NAME}.ipa ];
    then
    echo "============ Export IPA Success ========================"
    open ${IPA_PATH}
    else
    echo "============ Export IPA Fail ==========================="
fi


# 结束打包计时
end_time_2=$(date +%s)


# 删除Archive文件，可根据各自情况选择是否保留
# rm -r ${ARCHIVE_PATH}


### 上传过程 ###

## 上传app store

## 验证
## xcrun altool --validate-app -f ${IPA_PATH}/${SCHEME_NAME}.ipa -t ios --apiKey xxx --apiIssuer xxx --verbose

## 上传
## xcrun altool --upload-app -f ${IPA_PATH}/${SCHEME_NAME}.ipa -t ios --apiKey xxx --apiIssuer xxx --verbose

## 上传到蒲公英
echo "============ Upload PGYER Begin ========================"

## 具体参数可见 http://www.pgyer.com/doc/view/api#uploadApp
PGYER_UPLOAD_RESULT=`curl \
-F "file=@${IPA_PATH}/${SCHEME_NAME}.ipa" \
-F "buildInstallType=XXX" \
-F "buildPassword=XXX" \
-F "buildUpdateDescription=${UpdateDescription}" \
-F "_api_key=XXX" \
https://www.pgyer.com/apiv2/app/upload`

echo "============ Upload PGYER Success ======================"
## 返回结果码，其中0为成功上传，因为返回结果中带回来的有中文显示乱码，无法利用jq解析


# 如需上传到fim，可查阅 https://www.betaqr.com/docs/publish 文档，需要分两步，先获取凭证，再用凭证上传包
# macOS 上需要先安装 jq 工具解析 json，安装命令：brew install jq

# echo "============ Upload fir.im Begin ======================="

# FIR_GET_TICKET_RESULT=`curl -X "POST" "http://api.appmeta.cn/apps" \
# -H "Content-Type: application/json" \
# -d "{\"type\":\"ios\", \"bundle_id\":\"com.xxxxxxxx\", \"api_token\":\"xxxxxxxxxxxxxxxxxxxxxxxx\"}"`

# echo

# fir_key=$(echo $FIR_GET_TICKET_RESULT | jq -r '.cert.binary.key')
# fir_token=$(echo $FIR_GET_TICKET_RESULT | jq -r '.cert.binary.token')
# fir_upload_url=$(echo $FIR_GET_TICKET_RESULT | jq -r '.cert.binary.upload_url')

# APP_NAME=$(sed -n '/INFOPLIST_KEY_CFBundleDisplayName = /{s/INFOPLIST_KEY_CFBundleDisplayName = //;s/;//;s/^[[:space:]]*//;s/"//g;p;q;}' ${PROJECT_PATH}/project.pbxproj)

# FIR_UPLOAD_RESULT=`curl \
# -F "key=$fir_key" \
# -F "token=$fir_token" \
# -F "file=@${IPA_PATH}/${SCHEME_NAME}.ipa" \
# -F "x:name=$APP_NAME" \
# -F "x:version=${VERSION_NUMBER}" \
# -F "x:build=${BUILD_NUMBER}" \
# -F "x:release_type=Adhoc" \
# -F "x:changelog=${UpdateDescription}" \
# https://up.qbox.me`

# is_completed=$(echo $FIR_UPLOAD_RESULT | jq -r '.is_completed')

# if [ "$is_completed" = true ]; 
#     then
#     echo "============ Upload fir.im Success ====================="
#     else
#     echo "============ Upload fir.im Fail ========================"
# fi


# 结束传包计时
end_time_3=$(date +%s)

elapsed_time_1=$((end_time_1 - start_time))
elapsed_time_2=$((end_time_2 - end_time_1))
elapsed_time_3=$((end_time_3 - end_time_2))
total_time=$((end_time_3 - start_time))

calculate_time() {
    elapsed_time=$1
    message=$2

    hours=$((elapsed_time / 3600))
    minutes=$((elapsed_time % 3600 / 60))
    seconds=$((elapsed_time % 60))

    if [[ $hours -eq 0 ]]; then
        if [[ $minutes -eq 0 ]]; then
            echo "${message}：${seconds} 秒"
        else
            echo "${message}：${minutes} 分 ${seconds} 秒"
        fi
    else
        echo "${message}：${hours} 小时 ${minutes} 分 ${seconds} 秒"
    fi
}

calculate_time $total_time "总用时"
calculate_time $elapsed_time_1 "Unity导出iOS工程用时"
calculate_time $elapsed_time_2 "打包用时"
calculate_time $elapsed_time_3 "上传用时"

echo

```

## Unity 的 C# 脚本

`ExportProject.cs` 脚本需要放在 Unity 工程的 Assets/Editor 目录下，脚本内容如下

<br>

``` c#
using UnityEditor;
using UnityEngine;

public class ExportProject : MonoBehaviour
{
    public static void ExportIOS()
    {
        string[] args = System.Environment.GetCommandLineArgs();
        
        string outputPath = null;
        for (int i = 0; i < args.Length; i++)
        {
            if (args[i] == "-outputPath" && i + 1 < args.Length)
            {
                outputPath = args[i + 1];
                break;
            }
        }

        string[] scenes = GetSelectedScenesPaths();

        BuildPlayerOptions buildPlayerOptions = new BuildPlayerOptions
        {
            scenes = scenes,
            locationPathName = outputPath,
            target = BuildTarget.iOS,
            options = BuildOptions.None
        };

        BuildPipeline.BuildPlayer(buildPlayerOptions);
    }
    
    private static string[] GetSelectedScenesPaths()
    {
        string[] selectedScenePaths = new string[0];

        for (int i = 0; i < EditorBuildSettings.scenes.Length; i++)
        {
            EditorBuildSettingsScene scene = EditorBuildSettings.scenes[i];

            if (scene.enabled)
            {
                ArrayUtility.Add(ref selectedScenePaths, scene.path);
            }
        }

        return selectedScenePaths;
    }
}

```

<br>

## 可选

自定义一个终端命令的别名，方便快速的执行打包命令，打开自己电脑的`/Users/电脑用户名/.zshrc`隐藏文件，将以下代码添加进去，保存，这样在终端只要执行`bar`命令即可开始打包

``` shell
alias bar='killall -9 sh; sh /Users/xxxxx.sh'
```

以上代码是两条命令合成一条:

* `killall -9 sh` 的作用是杀掉和`sh`有关的进程，因为有时执行打包失败后再次执行，需要杀掉前一个`sh`的进程才能重新开始执行
* `sh /Users/xxxxx.sh` 是真正打包的命令
