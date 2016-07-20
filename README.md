# 崩溃日志上传框架
当App崩溃的时，把崩溃信息和保存到本地的同时，自动通过邮件或者HTTP发送出去。

使用邮件形式发送崩溃信息，配合第三方工具使用，可以自动提交崩溃信息到GitHub issue中，方便开发者利用GitHub的issue系统来对开源项目的Bug进行进行追踪管理，更多细节请见下面介绍
   
![enter image description here](http://ww1.sinaimg.cn/mw690/691cc151gw1f5zb0qor9nj208p092gm1.jpg)
   
特性介绍  
    
| 特性|简介|
| ------ | ------ |
|自定义日志保存路径 |默认保存在Android/data/com.xxxx.xxxx/log中|
|自定义日志缓存大小|默认大小为30M，超出后会自动清空文件夹|
|支持多种上传方式|目前支持邮件上传与HTTP上传，会一并把文件夹下的所有日志打成压缩包作为附件上传|
|日志加密保存|提供AES，DES两种加密解密方式支持，默认不加密|
|日志按天保存|目前崩溃日志和Log信息是按天保存，你可以继承接口来实现更多的保存样式|
|携带设备与OS信息|在创建日志的时候，会一并记录OS版本号，App版本，手机型号等信息，方便还原崩溃|
|自定义日志上传的时机|默认只在Wifi状态下上传支持|
|支持保存Log日志|在打印Log的同时，把Log写入到本地（保存的时候会附带线程名称，线程id，打印时间），还原用户操作路径，为修复崩溃提供更多细节信息|
|GitHub自动提交issue|使用邮件发送的形式，把接受崩溃日志的邮箱和GitHub特定的开源项目绑定在一起即可，更多细节请看下面介绍|


## 初始化
自定义Application文件，默认使用Email发送
``` java
public class MyApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        initEmailReporter();
        LogReport.getInstance()
                .setCacheSize(30 * 1024 * 1024)//支持设置缓存大小，超出后清空
                .setLogDir(getApplicationContext(), Environment.getExternalStorageDirectory().getPath() + "/LogReport/")//支持自定义日志保存路径
                .setWifiOnly(true)//支持
                //.setEncryption(new AESEncode()) //支持日志到AES加密或者DES加密，默认不开启
                .setLogSaver(new CrashWriter(getApplicationContext()))//支持自定义保存崩溃信息的样式
                .init(getApplicationContext());
    }

    /**
     * 使用EMAIL发送日志
     */
    private void initEmailReporter() {
        EmailReporter email = new EmailReporter(this);
        email.setReceiver("wenmingvs@gmail.com");//收件人
        email.setSender("wenmingvs@163.com");//发送人邮箱
        email.setSendPassword("apptest1234");//用于登录第三方的邮件授权码
        email.setSMTPHost("smtp.163.com");//SMTP地址
        email.setPort("465");//SMTP 端口
        LogReport.getInstance().setUploadType(email);
    }    
}
```

如果您有自己的服务器，想往服务器发送本地保存的日志文件，请使用以下方法替换initEmailReporter方法

``` java
    /**
     * 使用HTTP发送日志
     */
    private void initHttpReporter() {
        HttpReporter http = new HttpReporter(this);
        http.setUrl("http://crashreport.jd-app.com/your_receiver");//发送请求的地址
        http.setFileParam("fileName");//文件的参数名
        http.setToParam("to");//收件人参数名
        http.setTo("你的接收邮箱");//收件人
        http.setTitleParam("subject");//标题
        http.setBodyParam("message");//内容
        LogReport.getInstance().setUploadType(http);
    }
```
## 保存Log到本地
使用以下方法，打印Log的同时，把Log信息保存到本地（保存的时候会附带线程名称，线程id，打印时间），并且随同崩溃日志一起，发送到特定的邮箱或者服务器上。帮助开发者还原用户的操作路径，更好的分析崩溃产生的原因
``` java
LogWriter.writeLog("wenming", "打Log测试！！！！");
```

## 上传
在任意地方，调用以下方法即可，崩溃发生后，会在下一次App启动的时候使用Service异步打包日志，然后上传日志，发送成功与否，Service都会自动退出释放内存
``` java
LogReport.getInstance().upload(context);
```

## 关于使用邮件上传的注意事项
- 强烈建议使用163邮箱作为发送崩溃的邮箱，并且此邮箱务必要开启SMTP服务，如下图所示，才能发送邮件成功！
![enter image description here](http://ww1.sinaimg.cn/mw690/691cc151gw1f5zafbkamrj20fl05kaa8.jpg)
- 邮箱的密码，请输入163邮箱的客户端授权码，而不要输入邮箱的登录密码
- 请不要使用QQ邮箱作为发件人的邮箱，因为QQ邮箱的多重验证原因，并不支持这种形式的邮件发送
- 如果你需要自动提交崩溃日志到GitHub中，您需要使用Gmail邮箱作为收件人邮箱
- 提供以下测试邮箱给大家体验，此邮箱可以发送邮箱   
测试帐号   
帐号：wenmingvs@163.com   
登录密码：apptest123   
客户端授权码：apptest1234   

## GitHub自动提交issue的配置
请保证崩溃日志的邮件能到达您设定的Gmail邮箱中。如下图所示，才继续下去
![enter image description here](http://ww2.sinaimg.cn/mw690/691cc151gw1f5zg1m39koj212f06ydh9.jpg)

通过发送邮件来创建issue，在没有GitHub API的支持下，是不可能实现的。但是我们可以通过一些已经得到GitHub授权的第三方应用来实现。通过设置邮件监听，一旦邮箱收到了包含特定关键字的邮件，就会读取邮件的内容和标题，自动帮你提交issue。我们需要做的，就是把邮箱和GitHub的开源项目绑定起来，设置触发器即可，每次新邮件到达的时候，会检查此邮件是否满足触发器要求，如果满足，就会帮你自动提交issue。

[快速教程](https://zapier.com/zapbook/zaps/10314/create-github-issues-from-new-emails-on-gmail/)

1. 首先去注册一个帐号，如下图所示 [链接](https://zapier.com/app/min/10314/start)   
![enter image description here](http://ww1.sinaimg.cn/mw690/691cc151gw1f605xtdfudj20fg0gmab5.jpg)
2. 绑定Gmail邮箱，设置监听的邮件的类型，我们选择unread，即监听所有未读邮件（后面可以再根据个人要求修改）
![enter image description here](http://ww1.sinaimg.cn/mw690/691cc151gw1f6085s90r3j20hi0nwwge.jpg)
3. 绑定GitHub
4. 设置创建issue的样式，在3个输入框中指定以下内容   
指定在哪个项目下创建issue   
issue的标题与邮件的标题相同   
issue的内容和邮件的内容相同   
![enter image description here](http://ww4.sinaimg.cn/mw690/691cc151gw1f6085rjwcnj20hi0jrmy6.jpg)
5. 设定成功！！   
![enter image description here](http://ww3.sinaimg.cn/mw690/691cc151gw1f6085qy072j20gl0cv0tb.jpg)
6.  进入个人面板管理[链接](https://zapier.com/app/dashboard)，面板管理的Home标题下的每一个框框，都代表一个触发器，其中每个触发器都绑定了一个GitHub的开源项目   
![enter image description here](http://ww3.sinaimg.cn/mw1024/691cc151gw1f608iv7j2zj20kl04pwen.jpg)

后面我们还需要调整一下触发器，点击触发器的Edit，进入下面界面，做如下设置

1. 设定监听的邮件类型为New Email Matching Search
![enter image description here](http://ww1.sinaimg.cn/mw1024/691cc151gw1f609h9ptsnj21280hmdj6.jpg)
2. 设定按照邮件的标题，对邮件做进一步筛选。设定的subject:LogReport，意思是对标题包含有LogReport关键字的邮件做捕获。通过设定不同的关键字，我们就可以给不同的GitHub上的开源项目提交issue了
![enter image description here](http://ww3.sinaimg.cn/mw690/691cc151gw1f609mtbm4cj21290d6go6.jpg)
3. 对触发的动作做指示。我们需要自动提交issue，所以应该按照以下来设定，不再赘述了，看图：
![enter image description here](http://ww4.sinaimg.cn/mw690/691cc151gw1f609uz3aswj21220jpq6c.jpg)
![enter image description here](http://ww2.sinaimg.cn/mw690/691cc151gw1f609uy5bjbj211v0jk0w6.jpg)

至此，自动提交崩溃issue的配置就完成了   

