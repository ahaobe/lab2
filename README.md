# 实验目的
-  EJB调用和练习
# 实验任务
- 理解EJB，利用wildfly服务器容器进行远程调用。
- 建立有状态的Java Bean，实现以下功能：
操作用户（录入人员）登陆后，显示本次登陆的次数和上一次登陆的时间；
操作用户登录后，可进行校友的检索、修改、删除、统计等功能；
5分钟如果没有操作，则自动登出系统；
操作用户退出时，显示用户连接的时间长度，并把此次登陆记录到数据库。
在2台机器上模拟2个录入员，生成1000个校友用户，并进行各种增删改的操作。
# 环境设置、工具
- java jdk （1.6或者以上）
- IDEA 或者 eclipse 等开发环境
- Wildfly服务器
- mysql数据库
# 具体过程


### 1. 安装Wildfly服务器
#####a. 下载解压配置环境变量
#####b. 启动JBOSS
运行Wildfly下bin目录下的standalone.bat文件，访问localhost:8080，出现Wildfly的欢迎界面。

![安装成功界面](https://i.loli.net/2019/06/21/5d0c3a8e877ae98668.png
)
#####c. 添加管理用户
运行bin下的add-user文件，根据命令行的提示信息填写自己的用户名和密码。然后启动JBOSS，访问localhost:9990，输入设置的账号密码，访问后台页面。

![登录后台页面](https://i.loli.net/2019/06/21/5d0c3def5a5b179332.png)

![后台页面](https://i.loli.net/2019/06/21/5d0c3d2dcb4e675415.png
)

####2. 配置mysql数据库的数据源
#####a. 为Wildfly添加mysql的驱动jar包的依赖
在官网寻找mysql连接jar包（mysql-connector-java-5.1.34.jar）下载。在JBOSS_HOME/modules/system/layers/base/com路径下，创建mysql文件夹，在mysql下创建main文件夹，在mysql/main下创建一个module.xml，其内容为：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<module name="com.mysql" xmlns="urn:jboss:module:1.5">
    <resources>
        <resource-root path="mysql-connector-java-5.1.34-bin.jar"/>
    </resources>
    <dependencies>
        <module name="javax.api"/>
        <module name="javax.transaction.api"/>
    </dependencies>
</module>
```

#####b. 修改Wildfly的配置
打开JBOSS_HOME/standalone/configuration/standalone.xml文件，找到配置datasource的subsystem标签，里面包含datasource标签，Wildfly初始配置了一个样例数据源ExampleDS，其驱动是h2，修改该标签为：

```xml
<subsystem xmlns="urn:jboss:domain:datasources:5.0">
  <datasources>
    <datasource jndi-name="java:jboss/datasources/MysqlDS" pool-name="MysqlDS" enabled="true" use-java-context="true">
      <connection-url>jdbc:mysql://localhost:3306/schoolmates?useSSL=false</connection-url>
      <driver>mysql</driver>
      <pool>
        <min-pool-size>1</min-pool-size>
        <max-pool-size>20</max-pool-size>
      </pool>
      <security>
        <user-name>username</user-name>
        <password>password</password>
      </security>
    </datasource>
    <drivers>
      <driver name="mysql" module="com.mysql">
        <driver-class>com.mysql.jdbc.Driver</driver-class>
        <xa-datasource-class>com.mysql.jdbc.jdbc2.optional.MysqlXADataSource</xa-datasource-class>
      </driver>
    </drivers>
  </datasources>
</subsystem>
```
把ExampleDS对应修改为MysqlDS，修改connection-url为对应的数据库名，以及security标签内的用户名和密码。

#####c. 进入JBOSS后台页面，修改默认设置。启动JBOSS，进入后台页面，找到Configuration->Subsystem->EE->View，在左侧菜单栏找到Default Bindings，点击Edit，将ExampleDS改为MysqlDS，保存。

![修改后台配置](https://i.loli.net/2019/06/21/5d0c3fd8223cc26604.png
)


####3. 创建项目

#####a. 数据库建表。
![数据库建表](https://i.loli.net/2019/06/21/5d0c422f9ac5d11910.png)
admin表的字段有id、account、password、登录次数login_count、上次登录时间last_login_time；log表的字段有id、adminId、info。

#####b. idea创建一个空的Java项目。
#####c. 创建server子模块。
New->Module，选择Java Enterprise，选中Web Application和EJB: Enterprise Java Bean。
######① 创建Login接口
```
import javax.ejb.Remote;

@Remote
public interface Login {
    public int login(String account, String password);
}
```
######② LoginBean实现类
```
import javax.ejb.Stateless;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

@Stateless
public class LoginBean implements Login {
    @Override
    public int login(String account, String password) {
        int adminId = 0;

        Connection connection = null;
        PreparedStatement preparedStatement = null;
        ResultSet rs = null;

        try {
            connection = DataAccess.getConnection();

            String sql = "select * from admin where name='" + account + "' and password='" + password + "';";
            preparedStatement = connection.prepareStatement(sql);
            rs = preparedStatement.executeQuery();
            while (rs.next()) {
                adminId = rs.getInt("id");
                return adminId;
            }
            return adminId;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (rs != null) {
                try {
                    rs.close();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            if (preparedStatement != null) {
                try {
                    preparedStatement.close();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }

        return adminId;
    }
}
```
设置为无状态Bean。登录功能验证管理员是否存在和密码是否正确，若登录成功，则返回管理员的id。

######③创建Admin接口及其实现bean

![Admin接口](https://i.loli.net/2019/06/21/5d0c44faedf4d85055.png)
![AdminBean](https://i.loli.net/2019/06/21/5d0c45f1b451093412.png)

######④创建Alumni类
![Alumni类](https://i.loli.net/2019/06/21/5d0c46dce31c815446.png)

######⑤创建校友生成器：
![AlumniCreatorBean](https://i.loli.net/2019/06/21/5d0c49129b9ea58535.png
)
######⑥创建DataAccess类，建立和mysql数据库的连接
![DataAccess类](https://i.loli.net/2019/06/21/5d0c49ab4cf3742001.png)

#####d. 创建client子模块。
client端包含server端定义的接口——Login接口、Admin接口、AlumniCreator接口。
用户界面有登录页login.html、主页home.jsp、查找结果页searchAlumniByName.jsp、统计结果页searchAlumni.jsp，判定表单调用接口的login.jsp、addAlumni.jsp、deleteAlumni.jsp、modifyAlumni.jsp、createAlumni.jsp、logOut.jsp、autoLogOut.jsp。

将AdminBean存储在session中，登录成功后创建的AdminBean先存储在session中：
```
session.setAttribute("adminBean", admin);
```

不同的jsp页面要调用Admin接口的方法时，必须先获取当前已登录的AdminBean：
```
Admin admin = (Admin) session.getAttribute("adminBean");
```

在前端实现管理员一定时间未操作则自动退出的功能。
```
var lastTime = new Date().getTime();
var currentTime = new Date().getTime();

//设置超时时间：30秒
var timeOut = 30 * 1000;

//鼠标移动事件
document.onmouseover = function (ev) {
  //更新操作时间
  lastTime = new Date().getTime();
};
function autoLogStatus(){
  //更新当前时间
  currentTime = new Date().getTime();
  //判断是否超时
  if (currentTime - lastTime > timeOut) {
    window.location.href = "autoLogOut.jsp";
  }
}

/* 定时器  间隔1秒检测是否长时间未操作页面  */
window.setInterval(autoLogStatus, 1000);
```

在主页设置计时器，每隔一秒更新一次，若鼠标在一定时间内未移动，则超过规定时间后调用autoLogOut.jsp，自动退出。


#####e. 添加jboss-ejb-client.properties文件，放在client端的src文件夹下

```properties
endpoint.name=client-endpoint
remote.connectionprovider.create.options.org.xnio.Options.SSL_ENABLED=false
remote.connections=default
remote.connection.default.host=localhost
remote.connection.default.port=8080
remote.connection.default.connect.options.org.xnio.Options.SASL_POLICY_NOANONYMOUS=false
remote.connection.two.host=localhost
remote.connection.two.port=8080
remote.connection.two.connect.options.org.xnio.Options.SASL_POLICY_NOANONYMOUS=false
remote.connection.default.username=root
remote.connection.default.password=19980618
```


#####f. 配置JBoss
在右上角找到Edit Configurations，在左边菜单栏找到JBoss Server，如果没有可以点击左上角的"+"添加JBoss Server->Local，然后填写Name等信息。在Deployment选项卡中，添加server和client的war包。

![配置JBoss](https://i.loli.net/2019/06/21/5d0c4c46705ce78910.png
)

![配置JBoss](https://i.loli.net/2019/06/21/5d0c4c46492d658275.png
)

#####g. 运行
![校友目录](https://i.loli.net/2019/06/21/5d0c4db49649a20674.png
)

![添加校友](https://i.loli.net/2019/06/21/5d0c4e40936a696807.png
)

![删除校友](https://i.loli.net/2019/06/21/5d0c4e845087355102.png
)

![退出登录](https://i.loli.net/2019/06/21/5d0c4f14ed85e93702.png
)
