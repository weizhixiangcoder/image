## **背景**

    在使用指纹进行用户登录后， keyring没有被正常解锁， 会弹窗提示用户输入解锁keyring密码， 很大程度影响用户观感体验。

    正常使用密码登录时，不会出现上述问题， 因为keyring默认密码即用户登录密码，用户使用密码登录时，系统在登录验证用户身份后会使用登录密码继续进行keyring解锁验证，keyring锁会打开，从而当用户进入桌面后，可以直接使用keyring，但不会弹窗提示用户解锁keyring。
	
## **方案一 修改Master key**

    在新建用户或是用户修改密码时，同步获取当前用户的加密密码， 并通过混淆加密算法A进行混淆加密得到ObfuscatedPasswd， 并将ObfuscatedPasswd设置为Keyring的登录密码。

    lightdm的PAM认证流程分多个模块。 当用户登录模块中， 使用密码登录、指纹认证登录，或是U盾认证登录认证成功后， 接下来进行keyring登录认证，默认行为是keyring认证模块使用上一个认证模块传递过来的的用户密码passward进行认证， 解锁keyring。   

    现修改用户登录认证模块传递给keyring登录认证模块的passward。 在用户登录认证通过以后， 获取当前用户的加密密码， 并进行混淆运算，得到ObfuscatedPasswd， 将得到的ObfuscatedPasswd传递给keyring认证模块。 基于keyring的登录密码已经被修改为ObfuscatedPasswd， keyring中自行认证密码ObfuscatedPasswd， 相同即解锁。
	
    该方案安全保证的关键点是混淆加密算法A， 保证A的绝对保密即能保障用户keyring的私密性。

### **方案一疑难点**

    lightdm认证的PAM流程文件：/etc/pam.d/lightdm

    用户密码文件：/etc/shadow, 系统级别

    混淆加密思路： https://gist.github.com/megayu/50a6741875148613e38dd58a0a6a0972

    设置keyring登录密码: /etc/pam.d/common-password中修改传给pam_gnome_keyring.so的token值


### **方案一实现步骤**

#### **——， 添加用户和修改密码时更新Keyring登录密码ObscatedPasswd**




#### **二， 登录认证后读取密码文件获取用户加密密码并进行混淆加密得到ObscatedPasswd**

    a system程序获取当前用户的加密密码

    b system程序提供混淆算法接口

    c 将a中获取到的加密密码进行混淆加密


    读取/etc/shadow文件， 获取到用户密码后进行混淆加密

#### **三， 将上述计算的ObscatedPasswd当做token传递给keyring认证*

    a 将ObscatedPasswd传递给keyring

    pam_set_item    it is used to pass the most recent authentication token from one stacked module to another.


## **方案二 认证设备保存Master key**


### **方案二疑难点**

### **方案二实现步骤**

#### **一，用户设置密码时，修改Master key**

    a 用户设置密码
![](https://github.com/weizhixiangcoder/image/blob/main/keyring/swrz_changepasswd.png)

#### **二，添加生物认证时，将Master key保存到生物设备中**

#### **三，用户进行认证时，密码认证传入Master key, 生物认证则从生物设备上取出Master key传入**


## **方案一和方案二优缺点比较**

    方案一：修改Master key

    优点：
    ```
        只需要在软件层做修改, 相对容易实现
    ```

    缺点：
    ```
        并非绝对安全，混淆算法曝光可能导致用户信息泄漏
    ```

    方案二：认证设备保存Master key
    优点
    ```
        非常安全，保障用户的私密信息
    ```

    缺点
    ```
        需要硬件相关支持，目前的认证设备基本都不具备物理上存取Master key的功能，暂时难以实现
        涉及软硬件，比较复杂
    ```

## **小结**

    比较两个方案， 方案二中也包括方案一中修改Master key步骤，相对更复杂。现阶段暂时可以使用方案一实现生物认证登录解锁keyring。在后期硬件都具备存储功能后，再开发对应接口保存Master key，提高用户安全等级。


## **参考资料**
    
