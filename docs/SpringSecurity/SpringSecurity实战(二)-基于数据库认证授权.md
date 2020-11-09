[TOC]

## 目标

* 了解基于数据库的 spring security 方式

  参考： spring security 实战书籍

关于spring security 基本部分，可以直接看  spring security 实战书籍 或者[官方文档](https://docs.spring.io/spring-security/site/docs/5.4.1/reference/html5/#samples)参考。

## 基于数据库的认证授权

一个后台管理系统，最基础的功能就是权限管理。通常情况下会设计5张表来实现，三张基本表：用户表，角色表，权限表(也可以理解为菜单表)。两张表关联表：用户角色表，角色权限表。表与表之间的关系多对多，  一个用户可有多个角色(多身份)，一个角色也可以被多个用户拥有(同身份)，一个角色可赋予多个权限，一个权限也被多个角色拥有。

重要的类：

UserDetailsService ：用户详细信息服务，可以根据用户名获取用户信息。用于用户身份认证时，获取用户信息，供认证使用。

UserDetails ：用户信息，包含了一系列在验证时会用到的信息，包括用户名、密码、权限以及其他信息，Spring Security会根据这些信息判定验证是否成功。

无论数据库表怎么变换，只要实现UserDetailsService  接口，实现loadUserByUsername 方法，该方法根据用户名（username）获取用户详情（UserDetails）。spring security 就可以根据 UserDetails 来实现认证。

### 数据库设计

具体建表语句，参考地址：  [gengzi_spring_security](https://github.com/gengzi/gengzi_spring_security/tree/master/gengzi_spring_security/src/main/resources/db)

```shell
sys_permission 权限表
sys_role 角色表
sys_users 用户表
user_role 用户角色表
role_permission 角色权限表
```

 

### 代码实践

项目环境： spring boot 2.2.7  Jpa  java8  mysql5.7 

代码参考： https://github.com/gengzi/gengzi_spring_security

#### 依赖

核心依赖

```xml
           <!--security-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <!--security单元测试-->
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
```

#### 实现 UserDetails 用户详情实体

代码参考：[UserDetail](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/user/UserDetail.java) 代码太长了，不粘了

关注几个比较重要的字段：

```java
// 返回当前用户的权限集合
Collection<? extends GrantedAuthority> getAuthorities();
// 返回用户名
String getUsername();
// 返回密码
String getPassword();
// ...
```

实现UserDetails  接口的目的，就是将这些字段的数据提供给 spring security 使用。

用户名和密码就不说了，说下  authorities 权限集合这个字段。

GrantedAuthority ： 是一个只有 **getAuthority** 方法的接口。

```java
// 返回一个String 类型的权限
String getAuthority();
```

也就是说，每个权限，可以认为就是一个字符串来代表，在数据库表中，就是权限字段存储的文本。spring security 根据这个权限集合，来实现鉴权操作。

```java
// 比如 admin用户 包含  系统测试查询权限 和 系统管理查询权限 ，体现在 spring security  中，就是两个字符串
"authorities":[{"authority":"sys:test:qry"},{"authority":"sys:manager:qry"}],"
```

GrantedAuthority  提供了一个 SimpleGrantedAuthority ，方便创建一个授权类。

#### 实现 UserDetailsService 用户详情服务

UserDetailsServiceImpl ： 根据用户名获取用户详情

代码参考：[UserDetailsServiceImpl ](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/service/impl/UserDetailsServiceImpl.java)

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {
 
    @Autowired
    private UsersService usersService;
 
    @Autowired
    private PasswordEncoder passwordEncoder;
 
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
//        ReturnData result = new ReturnData();
        // 构造一个简单的 用户信息
//        UserDetail userDetailByDb = new UserDetail();
//        userDetailByDb.setUsername("user");
//        userDetailByDb.setPassword(passwordEncoder.encode("111"));
//        userDetailByDb.setStatus(1);
//        HashSet<GrantedAuthority> roleSet = new HashSet<>();
          // 设置一个权限
//        SimpleGrantedAuthority simpleGrantedAuthority = new SimpleGrantedAuthority("ADMIN");
//        roleSet.add(simpleGrantedAuthority);
//        userDetailByDb.setAuthorities(roleSet);
//        result.setInfo(userDetailByDb);
 
        // 从数据库中获取
        ReturnData result = usersService.loadUserByUsername(username);
        if (RspCodeEnum.NOTOKEN.getCode() == result.getStatus()) {
            throw new RrException(RspCodeEnum.ACCOUNT_NOT_EXIST.getDesc());
        }
        UserDetail userDetail = (UserDetail) result.getInfo();
        if (userDetail == null) {
            throw new RrException(RspCodeEnum.ACCOUNT_NOT_EXIST.getDesc());
        }
        //账号不可用
        if (userDetail.getStatus() == UserStatusEnum.DISABLE.getValue()) {
            userDetail.setEnabled(false);
        }
        return userDetail;
    }
}
```

 

具体的数据库查询用户信息操作：

代码参考：[UsersServiceImpl](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/sys/service/impl/UsersServiceImpl.java)

```java
    /**
     * 根据用户名查询用户信息
     * @param userName 用户名
     * @return
     */
    @Override
    public ReturnData loadUserByUsername(String userName) {
        ReturnData ret = ReturnData.newInstance();
        SysUsers sysUsers = sysUsersDao.findByUsername(userName);
        if (sysUsers == null || sysUsers.getId() == null) {
            ret.setFailure("系统中无此用户");
            ret.setStatus(RspCodeEnum.NOTOKEN.getCode());
            return ret;
        }
        UserDetail userDetail = ConvertUtils.sourceToTarget(sysUsers, UserDetail.class);
        userDetail.setStatus(sysUsers.getIsEnable());
        // 查询此用户对应的权限（菜单）
        List<SysPermission> sysPermissions = sysPermissionDao.qryPermissionInfoByUserId(sysUsers.getId());
        Set<String> permsSet = new HashSet<>();
        sysPermissions.forEach(sysPermission -> {
            if (StringUtils.isNotBlank(sysPermission.getPermission())) {
                permsSet.add(sysPermission.getPermission());
            }
        });
        //封装权限标识
        Set<GrantedAuthority> authorities = new HashSet<>();
        authorities.addAll(permsSet.stream().map(SimpleGrantedAuthority::new).collect(Collectors.toSet()));
        userDetail.setAuthorities(authorities);
        ret.setSuccess();
        ret.setInfo(userDetail);
        return ret;
    }
```

Dao 层：

1，根据用户名查询用户信息

```java
@Repository
public interface SysUsersDao extends JpaRepository<SysUsers, Long> {
    // 根据用户名查询用户信息
    SysUsers findByUsername(String username);
}
```

2，根据userid 查询该用户的权限信息

```java
@Repository
public interface SysPermissionDao extends JpaRepository<SysPermission, Long> {
 
    @Transactional
    @Modifying
    @Query("SELECT t3 FROM UserRole t1 LEFT JOIN RolePermission t2 ON t1.roleId = t2.roleId LEFT JOIN SysPermission t3 ON t3.id = t2.permissionId WHERE t1.userId = ?1")
    List<SysPermission> qryPermissionInfoByUserId(Long userId);
 
}
```

 

#### 核心配置类

代码参考： [WebSecurityConfig](https://github.com/gengzi/gengzi_spring_security/blob/master/gengzi_spring_security/src/main/java/fun/gengzi/gengzi_spring_security/config/WebSecurityConfig.java)

需要增加  userDetailsService 配置

```java
//其他配置，请直接参考代码
 
    // ------------  用户详细服务 -------------
    @Autowired
    private UserDetailsService userDetailsService;
 
  /**
     * 认证管理器配置方法
     * <p>
     * 用户身份的管理者（AuthenticationManager），认证的入口
     * 比如需要使用 userDetailsService 的用户信息，设置 auth.userDetailsService(userDetailsService)
     * 比如需要使用 内存用户信息，设置 auth.inMemoryAuthentication()
     *
     * @param auth
     * @throws Exception
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService);
    }
 
```

 

## 注意

对于上文中出现的多表关联查询操作，注意增加表与表之间关联字段的索引，以提高查询速度。具体建议，请参阅 阿里巴巴java开发规范。

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 
