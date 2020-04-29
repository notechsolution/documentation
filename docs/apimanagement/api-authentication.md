## Rest API的认证模式

微服务系统中，很多团队采用了API驱动设计开发，服务之间的调用都通过API来实现的。为了统一管理API，一般都会在前面部署一个API Gateway，然后由API Gateway对API的调用者进行权限认证。 常见的认证方式有下面几种：

- API Key
- OAuth
- JWT
- AppKey + Secret