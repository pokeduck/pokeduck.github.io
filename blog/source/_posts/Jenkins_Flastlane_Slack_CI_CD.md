---
title: 用 Jenkins+Fastlane+Slack 達成 CI/CD
date: 2019-03-02 16:00:01
categories: 
- CI/CD
tags: 
- iOS
- Jenkins
- Fastlane
- CI/CD
---


因為Jenkins的Xcode Integration有點難用
而且專案一多，要個別維護有點麻煩
如果Provisioning Profile有新增裝置 就要重新上傳

用Fastlane用相同設定 帶入不同的Scheme就可以完成
ProvisioningProfile可直接讀取/Library底下的內容
CodeSign用Xcode的自動下載就好可以完成

## Jenkins

* 負責角色

> Bitbucket <-> **Jenkins** -> Fastlane -> Slack ...

安裝 Jenkins

```shell
brew install java8
brew install Jenkins
```

### Configure

1. Source Code Management
* Git
    * Repositories
        * URL : ...
        * Credentials : ...
    * Branches to build
        * Brahch : ...

2. Execute shell

    ```bash=
    #!/bin/sh
    cd ${WORKSPACE}

    source ~/.bash_profile

    git checkout Your_Branch_Name
    #先切換 Branch 不然 increase commit 會跑到 HEAD detached
    
    git fetch
    
    git merge your_remote/your_branch

    fastlane your_lane
    ```

Fastlane
---
Ruby語言寫成的

### 安裝

```
brew cask install Fastlane
```

安裝完要更新一下  **bash_profiles**
```shell=
#~/.bash_profiles

export PATH="$HOME/.fastlane/bin
```
### 執行
在專案資料夾初始化 Fastlane
```
fastlane init
```

在專案目錄下執行
```bash
fastlane your_lane
```
> 在 /your/proj/path/fastlane 下執行也可以



執行Fastlane 主要是根據 Fastfile會在
```
/your/proj/path/fastlane/Fastfile
```

> 還有其他安裝與執行方式
> rbenv,bundler etc.,

### Code Signing
打包要用的 Provision Profile and Cert. 有很多種方式，這邊使用[手動的方法](https://docs.fastlane.tools/codesigning/getting-started/#manually)
> 或是使用 [match](https://docs.fastlane.tools/codesigning/getting-started/#using-match)

在Xocde > Preference > Accounts 登入，然後下載全部的 Provisioning Profile
Provisioning Profile 預設位置在 **~/Library/MobileDevice/Provisioning Profiles**
Keychain 匯入 Distribuion Cert.
Project Target > Gerenal > Signing 憑證必須選對 

### 設定檔
* Fastfile Quick Start

```bash=
update_fastlane

default_platform(:ios)

platform :ios do
    lane :your_lane do
       build_ios_app(
          scheme: "YourScheme",
          workspace: "YourProject.xcworkspace",
          configuration: "Release", # or Debug
          export_options: {
            method: "ad-hoc",
            provisioningProfiles: {
              "your.domain.udid" => "project1_adhoc_name"
            },
            iCloudContainerEnvironment: "Production", # or 'Development'
          },
          clean: true,
          output_directory: "build",
          output_name: "ota.ipa",
          include_bitcode: false,
        ) 
    end
end
```

* 切換目錄
```bash=
Dir.chdir ".." do
    # your script   
end
#效果等於 "cd .."
```

* 執行shell
```bash=
#在一般command line
sh abcd.sh arg1 arg2
pwd
echo AA
#在Fastfile
sh "bash 'arg1' 'arg2'"
sh 'pwd'
sh "echo 'AA'"
```

> 使用 sh 起始位置會在/your/proj/path/fastlane
> 使用 fastlane 其他 action 起始位置在 /your/proj/path

* 使用變數
```bash=
#宣告
var1 = "foo"
var2 = "bar"
var3 = var1 + "123" + var2
#var3 = foo123bar

#傳入參數到 bash 雙引號加上單引號的 Nested
sh "bash script.sh '#{var1}' ''$PWD'/'#{var2}'/' '#{var3}'"

#在Fastfile內 用雙引號就可以
yourFunc("ZXC#{var1}QSD")

def yourFunc(A)
#傳入參數 A 為 -> ZXCfooQSD
end


```

* 使用自定義 function
```bash=
def yourfuncA(arg1,arg2,arg3)
    #some script
end

#使用
lane :your_lane do
    yourfuncA("A","B","C")
end
```

* 自動增加 build number
```bash=
 #宣告成function
 def increase_build_number_and_git_push(schemeName,target)
    increment_build_number_in_plist(target: schemeName)
    info_plist_path = get_info_plist_path(xcodeproj: 'YourProj.xcodeproj', # optional
                    target: schemeName, # optional, or `scheme`
                    # optional, must be specified if you have different Info.plist build settings
                    # for different build configurations
                    build_configuration_name: 'Release')
    app_name = get_info_plist_value(path: info_plist_path, key: "CFBundleDisplayName")
    ver_number = get_info_plist_value(path: info_plist_path, key: "CFBundleShortVersionString")
    build_number = get_info_plist_value(path: info_plist_path, key: "CFBundleVersion")
    git_commit(path:info_plist_path, message:"[CI-Skip] Updated [#{app_name}] v#{ver_number} Build #{build_number} for #{target}")
    push_to_git_remote(
      remote: "origin",         # optional, default: "origin"
      # local_branch: "release",  # optional, aliased by "branch", default is set to current branch
      remote_branch: "release", # optional, default is set to local_branch
      force: true,    # optional, default: false
      force_with_lease: true,   # optional, default: false
      tags: false,    # optional, default: true
      no_verify: true,# optional, default: false
      set_upstream: true        # optional, default: false
    )
  end
  
# 執行
increase_build_number_and_git_push("Your_Proj_Scheme","iTunesConnect")

# # 執行結果：
# 在Release Branch 會自動 git push -f 
# commit 訊息：[CI-Skip] Updated [AppName] v1.9.18 Build 1 for iTunesConnect

```
> 也可以使用 **increment_build_number** 
> 但是會把 xcworkspace 裡面全部 *scheme* 的 build number 都加一
> 個別指定 target 就用 **increment_build_number_in_plist** [參考](https://github.com/fastlane/fastlane/issues/9229#issuecomment-319903690)
> 必須先安裝第三方外掛：**fastlane add_plugin versioning** [套件網址](https://github.com/SiarheiFedartsou/fastlane-plugin-versioning)


* 錯誤擷取
```Fastfile
begin
    functionA("arg1","arg2")
rescue => exception
    on_error_Func(exception)  
end
```

* 上傳dSYM檔案
```bash
 upload_symbols_to_crashlytics(
    dsym_path: "./build/yourapp.app.dSYM.zip",
    gsp_path: "./your/firebase/GoogleService-info.plist"
  )
```

### 除錯
* **dSYM上傳失敗的問題**
    根據這兩個討論
    https://github.com/fastlane/fastlane/issues/13096
    https://github.com/fastlane/fastlane/issues/14176
    使用 upload_symbols_to_crashlytics時候會出現
    **invalid byte sequence in UTF-8**
    之後雖然 Fastlane 表示上傳成功
    但是Firebase後台還是沒有收到dSYM
    **2019-03-16** 沒有解 先用手動上傳

* 執行 **increment_build_number** 出現 
    ```
    Please remove $(SRCROOT) in your Xcode target build settings
    ```

    解決方式： info.plist 不使用 $(SRCROOT) 改用絕對路徑 [參考](https://github.com/fastlane/fastlane/issues/329#issuecomment-236923632)

    ```bash
    #make this
    INFOPLIST_FILE = "$(SRCROOT)/Project/info.plist";

    #into this
    INFOPLIST_FILE = "Project/info.plist";

    ```
* 同時匯出AppStore and AdHoc
    可以使用 resign 的方法，[但是 Fastlane 建議跑兩次 lane](https://github.com/fastlane/fastlane/issues/11272#issuecomment-352667631)

Slack
---
傳訊息進去 Slack 是利用 Webhook 方式
再依照他的ＡＰＩ傳入對應參數
Fastlane 有封裝好的 plugin
只要填入 Slack Webhook就可以
https://docs.fastlane.tools/actions/slack/

### 幾個 Slack 範例

* 上傳 iTunes Connect 成功
```Fastfile

def slack_itunes(appName,appScheme,ipaPath)
  ver = get_ipa_info_plist_value(ipa: ipaPath, key: "CFBundleShortVersionString")
  build = get_ipa_info_plist_value(ipa: ipaPath, key: "CFBundleVersion")
  slack(
   message: "["+appName + "] uploaded to iTunesConnect. 🚀🚀🚀",
   success: true,
   channel: "#your_channel",
   slack_url: "https://your/slack/webhook",
   username: "Jenkins_CI",
   icon_url: "http://Bot大頭貼/icon/jenkins.png",
   default_payloads: [],
   link_names: true,
   payload: { 
      "Version" =>  "#{ver}" + " (" + "#{build}" + ")",
      "Upload Date" => Time.new.to_s,
    },
  attachment_properties: {
    thumb_url: "https://i.imgur.com/STnXPFy.png",
  }
)
end
```
效果:
![](https://i.imgur.com/JYISx6c.png)


> attachment_properties 
> 代表包裝好的參數之外
> 可傳入Slack API有的參數
> https://api.slack.com/docs/message-attachments

* 上傳OTA
```Fastfile
def slack_ota(appName,appScheme,ipaPath)
  ver = get_ipa_info_plist_value(ipa: ipaPath, key: "CFBundleShortVersionString")
  build = get_ipa_info_plist_value(ipa: ipaPath, key: "CFBundleVersion")
  slack(
    slack_url: "https://your/slack/webhook",
    message: "["+appName + "] uploaded to OTA ! 🎉",
    username: "Jenkins_CI",
    link_names: "http://Your.CI.Server.domain",
    icon_url: "http://Jenkins.icon/jenkins.png",
    channel: "#jenkins_bot",  
    success: true, 
    payload: { 
      "Version" =>  "#{ver}" + " (" + "#{build}" + ")",
      "Release Date" => Time.new.to_s,
    },
    default_payloads: [], 
    attachment_properties: {
      color: "#2eb886",
      title: "OTA Download Page",
      title_link: "https://Your.OTA.download/page",
     },
    )
end
```
* 效果：
![](https://i.imgur.com/xe1Y5Sh.png)
```Fastfile
def on_error(exception,appName,appScheme)

  slack(
      message: "[" +appName+ "] some thing goes wrong",
      success: false,
      channel: "#Your_Error_Msg_Channel",
      slack_url: "https://Your.webhook",
      username: "Jenkins_CI",
      icon_url: "http://Jenkins.Thumbnail.icon/jenkins.png",
      default_payloads: [],
      payload: {
        "Version" =>  get_version_number(xcodeproj: "YourProj.xcodeproj",target: appScheme),
      },
      attachment_properties: {
          fields: [
              {
                  title: "Error message",
                  value: exception.to_s,
                  short: false
              }
          ],
          title: "Jenkins",
          title_link: "http://Your.CI.Server.domain/",  
      }
  )
end

```
效果:
![](https://i.imgur.com/rnaZIGD.png)
