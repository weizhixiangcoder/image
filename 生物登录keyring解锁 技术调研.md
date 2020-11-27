## **背景**

在使用指纹进行用户登录后， keyring没有被正常解锁， 会弹窗提示用户输入解锁keyring密码， 很大程度影响用户观感体验。

正常使用密码登录时，不会出现上述问题， 因为keyring默认密码即用户登录密码，用户使用密码登录时，系统在登录验证用户身份后会使用登录密码继续进行keyring解锁验证，keyring锁会打开，从而当用户进入桌面后，可以直接使用keyring，但不会弹窗提示用户解锁keyring。

设置密码时设定keyring Master key流程：

![图片](https://github.com/weizhixiangcoder/image/blob/main/keyring/normal_masterkey.png)

密码登录时验证keyring流程：

![图片](https://github.com/weizhixiangcoder/image/blob/main/keyring/normal_login.png)

根据PAM机制， 现提供两种非密码登录解锁keyring的方案。

## **方案一 修改Master key**

Master key可以理解为keyring的登录密码。

在设置改密码时，同步获取当前用户的加密密码， 并通过混淆加密算法A进行混淆加密得到ObfuscatedPasswd， 并将ObfuscatedPasswd设置为Keyring的登录密码。

lightdm的PAM认证流程分多个模块。 当用户登录模块中， 使用密码登录、指纹认证登录，或是U盾认证登录认证成功后， 接下来进行keyring登录认证，默认行为是keyring认证模块使用上一个认证模块传递过来的的用户密码passward进行认证， 解锁keyring。   

现修改用户登录认证模块传递给keyring登录认证模块的passward。 在用户登录认证通过以后， 获取当前用户的加密密码， 并进行混淆运算，得到ObfuscatedPasswd， 将得到的ObfuscatedPasswd传递给keyring认证模块。 基于keyring的登录密码已经被修改为ObfuscatedPasswd， keyring中自行认证密码ObfuscatedPasswd， 相同即解锁。

修改Master key方案解锁keyring流程：
![图片](https://github.com/weizhixiangcoder/image/blob/main/keyring/swrz1.png)


该方案安全保证的关键点是混淆加密算法A， 保证A的绝对保密即能保障用户keyring的私密性。


### **方案一疑难点**

密码认证PAM流程文件：/etc/pam.d/common-passwd

登录认证的PAM流程文件：/etc/pam.d/lightdm

用户密码文件/etc/shadow为系统级别

混淆加密思路： https://gist.github.com/megayu/50a6741875148613e38dd58a0a6a0972

### **方案一实现步骤**

#### **一，修改密码认证流程**

a 获取当前用户的加密密码(/etc/shadow文件中当前用户加密密码)

b 调用混淆算法接口进行混淆得到ObfuscatedPasswd

c 在设置密码时， 用户密码认证通过后，将b中得到的ObfuscatedPasswd替代默认密码传入gnome-keyring，设置为Master key


#### **二， 修改登录认证流程**

a 获取当前用户的加密密码(/etc/shadow文件中当前用户加密密码)

b 调用混淆算法接口进行混淆得到ObfuscatedPasswd

c 在用户登录时， 登录认证通过后，将b中的ObscatedPasswd传递给keyring进行认证， 认证通过则解锁keyring

------

## **方案二 认证设备保存Master key**

此方案要求生物认证设备能够安全存储数据， 并按照UOS的认证接入规范提供对应的接口。 

这种方案下Master key是用户密码的摘要，在添加生物认证时，要求输入用户密码， 待生物认证添加成功后， 将密码的摘要传递给生物认证接口， 由生物认证接口存储到设备中。

在用户通过生物认证登录后， 由生物认证接口读取设备中的摘要数据并传递给UOS认证服务， UOS认证服务则将摘要传递给gnome-keyring进行解锁。


认证设备保存Master key方案解锁keyring流程：
![图片](https://github.com/weizhixiangcoder/image/blob/main/keyring/swrz2.png)

### **方案二疑难点**

密码认证PAM流程文件：/etc/pam.d/common-passwd

lightdm认证的PAM流程文件：/etc/pam.d/lightdm

生物认证设备需要具备存储数据

生物认证设备需要提供符合UOS规范的接口

### **方案二实现步骤**

#### **一，修改密码认证流程**

在用户修改没密码时， 用户密码认证通过后，将原本传递给keyring的用户密码进行hash后再传入keyring

#### **二，添加生物认证设备** 

a 添加生物设备密码认证通过后,  获取密码并进行hash

b 将a中获取的hash值通过特定接口存入生物认证设备中

#### **三，修改登录认证流程**

在用户登录时， 登录认证通过后，如果是密码登录则将密码hash值传入keyring; 如果是生物设备登录则调用生物设备接口获取存储的hash值，传入keyring。 keyring验证通过即解锁。


## **方案一和方案二优缺点比较**

方案一：修改Master key
优点：
```
只需要在软件层做修改, 无需硬件支持
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

比较两个方案， 方案二中也包括方案一中修改Master key步骤，实现起来相对更复杂。现阶段暂时可以使用方案一实现生物认证登录解锁keyring。在后期硬件都具备存储功能后，再开发对应接口保存Master key，提高用户安全等级。


## **参考资料**

PAM介绍：https://wiki.archlinux.org/index.php/PAM

PMA模块编写指南：http://www.linux-pam.org/Linux-PAM-html/Linux-PAM_MWG.html

Gnome/keyring介绍：https://wiki.archlinux.org/index.php/GNOME/Keyring

Gnome/keyring指南：https://wiki.gnome.org/Projects/GnomeKeyring
