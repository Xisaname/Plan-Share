# 网络问题
## repo sync出现问题
执行repo sync命令同步代码时，出现网络问题，即连不上GitHub等网站
## 解决方法
方法1:多执行几次repo sync，此问题的出现是概率问题

方法2:挂代理，使用proxychains工具。即在命令之前都加上proxychains字段，同时配置好git，curl等工具的代理接口，开启代理即可。不过本人试过这种方法，不是很稳定。怀疑可能是机场的问题，因此此方法不过多赘述，方法1即可解决。
# 编译问题
## 在编译整个OPTEE工程时出现问题
执行make -f qumu_v8.mk all命令编译工程时，由于权限问题，我在命令前方添加了sudo权限。但是在编译的过程中报错：`you should not run configure as root`，但是不用root权限会导致很多`mkdir`指令无法运行。
## 解决方法
网上说修改`/etc/profile`，在最后一行添加以下命令：

```
echo "export set FORCE_UNSAFE_CONFIGURE=1"
```
然后执行命令使上述命令全局生效:
``` 
source /etc/profile
```
但是我用了上述方法不行，可能还需要重启。但是我用了临时的方法，即在执行make指令时，直接在make字段后添加`FORCE_UNSAFE_CONFIGURE=1`，这样就可以保证本次make指令不会出现上述错误。