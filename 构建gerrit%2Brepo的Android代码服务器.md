# 构建gerrit+repo的Android代码服务器
## 第一部分：gerrit
>一般在企业的Android开发中，公司会给程序员分配一个远程服务器，企业本身也有一个代码库（该代码库集合了公司各种不同项目的Android系统代码）。程序员可以在自己的远程服务器上，从公司的代码库下拉相应的项目代码，然后进行基于Android源码的开发、维护、模块单编译(mm)、全编译(make -j4)等各种企业需求的开发工作。
程序员开发完成需求时，需要将开发成果保存在公司的代码库，而这个保存的过程是需要一系列的安全性工作的。首先，程序员开发和维护的成果，需要经过项目负责人（主管、经理、总监之类）的评审、验证才能最终提交到公司的代码库。
这个代码评审、核查的过程，由Gerrit(代码评审工具)实现。


1. 安装Java环境

2. 下载gerrit安装包
gerrit的下载地址:
https://www.gerritcodereview.com/releases/README.md
我使用的gerrit版本是2.9.1。 下载完成之后会发现是一个war的包，所以我们的Java环境一定要安装正确。

3. 新建gerrit专用用户
新建一个用户用来专门管理gerrit相关的内容。
在root用户（或者使用sudo命令）下面输入下面的命令:
`adduser gerrit2`
`su gerrit2`
建好用户以后，我们可以把之前下载好的gerrit安装包(gerrit-2.9.1.war)拷贝到/home/gerrit2/目录下，一会方便gerrit2用户来安装。

4. 安装gerrit
在gerrit用户的目录(/home/gerrit/)下面，执行下面的命令：
`java -jar gerrit-2.9.1.war init -d ~/gerrit_site_http`
这个命令的意思是执行安装gerrit，会在当前目录下新建一个文件夹gerrit_site用来作为gerrit的根目录，在这个目录中，会安装git仓库，以及gerrit的web页面，还有gerrit的bin，etc等文件夹。
然后就开始安装过程了，安装的过程会询问很多问题，有一些判断性的问题会用[y/N]这样的形式，大写的字母表示默认，我们直接敲回车就表示采用默认的安装选项。我们安装的时候，可以只在Authentication method时输入http，其他全部回车用默认值，因为其他配置我们待会可以通过etc/gerrit.config文件进行修改。

5. 配置gerrit.config，可参考下面
```xml
[gerrit]
	basePath = git
	canonicalWebUrl = http://172.16.10.116:8081/
[database]
	type = h2
	database = db/ReviewDB
[index]
	type = LUCENE
[auth]
	type = HTTP
[sendemail]
	smtpServer = localhost
[container]
	user = gerrit2
	javaHome = /usr/lib/jvm/java-7-openjdk-amd64/jre
[sshd]
	listenAddress = *:29418
[httpd]
	listenUrl = proxy-http://172.16.10.116:8081/
[cache]
	directory = cache
[sendemail]
    smtpServer = smtpcom.263xmail.com
	smtpServerPort = 465 
	smtpEncryption = SSL
	smtpUser = liang.zhang@aispeech.com
	smtpPass = ×××××××××
	from = liang.zhang@aispeech.com
```

6. 安装配置apache2
`sudo apt-get install apache2`
反向代理可参考下面
/etc/apache2/httpd.conf
```xml
<VirtualHost *:80>
 ProxyRequests Off
 ProxyVia Off
 ProxyPreserveHost On
 <Proxy *>
Order deny,allow
Allow from all
 </Proxy>
 <Location /login></Location>
    AuthType Basic
    AuthName "Gerrit Code Review"
    AuthBasicProvider file
    AuthUserFile /home/gerrit2/gerrit_site_http/etc/passwords
    Require valid-user
 </Location>
 AllowEncodedSlashes On
 ProxyPass / http://172.16.10.116:8081/ 
</VirtualHost>
```

7. 设置第一个gerrit用户（管理员）的帐号和密码
进入/home/gerrit2/gerrit_site_http/etc目录下
touch passwords
htpasswd passwords admin
这里的admin就是以后用来登录gerrit的用户名。以后要为gerirt增加用户，也需要通过htpasswd命令来添加。
用htpasswd创建的用户时，并没有往gerrit中添加账号，只有当该用户通过web登陆gerrit服务器时，该账号才会被添加进gerrit数据库中。

8. 重启电脑然后重启服务
`sudo /etc/init.d/apache2 restart`
`su gerrit2`
`cd /home/gerrit2/gerrit_site_http`
`./bin/gerrit.sh stop`
`./bin/gerrit.sh start`

9. 打开浏览器，输入172.16.10.116即可登录

10. 删除git工程
如果你想删除一些无用的或者测试的git工程，只需要去/home/gerrit2/gerrit_site_http/git目录下删除掉相应的工程，然后重启一下gerrit服务，同时，把浏览器整个关闭，然后再重新登录，就可以了。至于为什么要关闭掉整个浏览器，接着往下看。

11. 退出gerrit网页
如果你已经成功登录了gerrit的网页，那么如果你想退出，请直接关闭整个浏览器，gerrit没有做logout的session清除，所以如果你直接点击网页右上角的logout，要么会出错，要么就会进入到之前那个提示需要反向代理的错误页面。关于登出，gerrit给出的原因是：
>You are using HTTP Basic authentication. There is no way to tell abrowser to quit sending basic authentication credentials, to logout with basicauthentication is to close the Webbrowser.

12. 关于gerrit的使用，分组/审查，权限的设置，这个内容比较多，可参考如下文档：
http://lipeng1667.github.io/2017/01/18/gerrit-guide/  (可忽略sourceTree部分介绍)

## 第二部分：repo
>repo是我们建立在 Git 顶部的一个存储库管理工具。Repo 在必要时可以统一许多个 Git 仓库，上传到我们的 版本控制系统，并且自动化 Android 部分开发工作流程。Repo 并不意味着取代 Git，只是帮助更容易地在 Android 环境中使用 Git。Repo 命令是一个可执行的 Python 脚本，你可以把它放在你的任何路径上。在使用 Android 源文件工作时，你将把 Repo 使用于跨网络操作。例如，使用一个单一的 Repo 命令，你就可以从多个存储库下载文件到你的本地目录下。

1. 下载repo可执行脚本（客户端）
`curl http://android.git.kernel.org/repo > /usr/bin/repo`
`chmod a+x /usr/bin/repo`


## 第三部分：上传Android代码到gerrit服务器
1. 配置default.xml
- 新建manifest.git
	可以直接通过gerrit网页新建

- clone到本地（客户端）
git clone ssh://admin@172.16.10.238:29418/manifest.git

- 配置default.xml（客户端）
`cd manifest`
`vi default.xml`
内容如下
```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  
  <remote  name="exdroid"
           fetch="android" />
  <default revision="r16-v1.y"
           remote="exdroid"
           sync-j="8" />
  
  <project path="build" name="platform/build" groups="pdk" >
    <copyfile src="core/root.mk" dest="Makefile" />
  </project>
  ...
</manifest>
```
-  上传到远程仓库
`git add .`
`git comm -am "add android.xml"`
`git push origin master`



2.创建git仓库并上传本地带git信息工程到服务器
- 进入Android代码目录，新建脚本文件
`vim GitPushAllLocalReposityToGerrit.sh`
内容如下：

```shell
#BEGIN:

#####################################################################################
##将本地代码全部上传到gerrit服务器进行管理，包括每一个仓库的本地所有分支和tag
#####################################################################################
LOCAL_PATH=`pwd`
MANIFEST_XML_FILE=/tmp/ttt/manifest/android.xml

#项目各个仓库的前缀，用于区分多个项目仓
PROJECT_NAME_PREFIX="android"
#git用户名字，gerrit服务器ip，gerrit服务器端口
USER_NAME="admin"
SERVER_IP="172.16.10.238"
SERVER_PORT="29418"

#用repo命令时表示各个仓库的变量
REPO_PROJECT_STRING="\$REPO_PROJECT"


OUTPUT_PROJECT_LIST_FILE_NAME=$LOCAL_PATH/project_list_name
OUTPUT_PROJECT_LIST_FILE_PATH=$LOCAL_PATH/project_list_path

#从.repo/manifest.xml中获取各个仓的名字和路径
function getNameAndPath()
{
    echo > $OUTPUT_PROJECT_LIST_FILE_NAME
    echo > $OUTPUT_PROJECT_LIST_FILE_PATH

    while read LINE
    do
        command_line=`echo $LINE | grep "<project"`
        if [ "$command_line" ] 
        then
            #echo $LINE

            reposity_name_sec=${LINE#*name=\"}
            reposity_path_sec=${LINE#*path=\"}

            if [ "$reposity_name_sec" ] && [ "$reposity_path_sec" ]
            then
                reposity_name=${reposity_name_sec%%\"*}
                reposity_path=${reposity_path_sec%%\"*}
                echo "$reposity_name" >> $OUTPUT_PROJECT_LIST_FILE_NAME
                echo "$reposity_path" >> $OUTPUT_PROJECT_LIST_FILE_PATH
            fi
        fi
    done  < $MANIFEST_XML_FILE
}


#在远程gerrit服务器建立各个仓
function creatEmptyGerritProject()
{
    for i in `cat $OUTPUT_PROJECT_LIST_FILE_NAME`;
    do
        echo $i
        echo "ssh -p $SERVER_PORT $USER_NAME@$SERVER_IP gerrit create-project -n $PROJECT_NAME_PREFIX/$i"

        #在gerrit服务器创建空项目

        ssh -p $SERVER_PORT $USER_NAME@$SERVER_IP gerrit create-project -n $PROJECT_NAME_PREFIX/$i
    done
}



#推送本地仓库到服务器，包括本地仓库的所有branch分支信息和tag标签信息
function pushLocalReposityToRemote()
{

    while read LINE
    do
        cd $LOCAL_PATH
        command_line=`echo $LINE | grep "<project"`
        if [ "$command_line" ] 
        then
            #echo $LINE
            reposity_name_sec=${LINE#*name=\"}
            reposity_path_sec=${LINE#*path=\"}

            if [ "$reposity_name_sec" ] && [ "$reposity_path_sec" ]
            then
                reposity_name=${reposity_name_sec%%\"*}
                reposity_path=${reposity_path_sec%%\"*}

                cd $LOCAL_PATH/$reposity_path
                echo `pwd`

                REMOTE_REPOSITY_NAME=`git remote`
                ALL_LOCAL_BRANCHS=`git branch -a | grep "remotes/$REMOTE_REPOSITY_NAME"`
                for branchLoop in `echo $ALL_LOCAL_BRANCHS`
                do
                    BRANCH_NAME=${branchLoop#*$REMOTE_REPOSITY_NAME\/}
                    echo "------------local branch name is------------:$BRANCH_NAME"
                    echo "------------local branch loop is------------:$branchLoop"
                
                    #提交本地所有分支信息到远程分支
                    git push  ssh://$USER_NAME@$SERVER_IP:$SERVER_PORT/$PROJECT_NAME_PREFIX/$reposity_name $branchLoop:refs/heads/$BRANCH_NAME
					
                    #提交本地所有tag信息到远程分支
                    git push --tag ssh://$USER_NAME@$SERVER_IP:$SERVER_PORT/$PROJECT_NAME_PREFIX/$reposity_name 
                done


            fi
        fi

    done  < $MANIFEST_XML_FILE

}

#调用函数执行建仓传代码的整个过程
getNameAndPath
creatEmptyGerritProject
pushLocalReposityToRemote

#END
```
- 运行脚本，创建git仓库
`. /GitPushAllLocalReposityToGerrit.sh`


3.下载代码
`mkdir android`
`cd android`
`repo init -u ssh://caijun.yang@172.16.10.238:29418/manifest.git  -m android.xml`
`repo sync -f -j8`
