# Authentication 是什么

接口 Authentication 用来表示用户认证，有如下 6 个方法：

```
public interface Authentication extends Principal, Serializable {

	Collection<? extends GrantedAuthority> getAuthorities();

	Object getCredentials();

	Object getDetails();

	Object getPrincipal();

	boolean isAuthenticated();

	void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
```

getAuthorities() 返回用户的权限，比如读取、写入等

getCredentials() 返回用户身份确认信息，比如密码、验证码等

getDetails() 返回用户身份具体的信息。Principal 是用户身份唯一标识的信息，Details 是用户身份具体信息，是一个 User 对象。

getPrincipal() 用户身份唯一标识的信息，像 userId、mobile 这样的唯一可以查找出用户身份的数据。

isAuthenticated() 返回当前认证信息是否是已经认证的。AuthenticationProvider 经常在 authenticated() 会调用该方法，如果 isAuthenticated() 返回了 true，表明用户身份已经认证过，不需要再认证，一般会直接 return。一般有多个 AuthenticationProvider 组合在 AuthenticationManager 之中，大多数情况下只只要其中之一身份认证通过，其余的就不需要通过了，比如，用户可以进行密码身份认证、验证码身份认证，这两种身份认证之一通过就足够了。

setAuthenticated() 设置身份认证。拿上面的例子来说，如果 AuthenticationProvider 的实现 UsernamePasswordAuthenticationProvider 身份认证通过了，那么 AuthenticationProvider 的实现 ValidationCodeAuthenticationProvider 就不需要再进行身份认证了。

## Authentication 实现

类 UsernamePasswordAuthenticationToken。很简单的一个类，实现了用户名密码身份认证。

在 UsernamePasswordAuthenticationFilter 是这么使用该类的：

```
		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
				username, password);

		// Allow subclasses to set the "details" property
		setDetails(request, authRequest);

		return this.getAuthenticationManager().authenticate(authRequest);
```

首先实例化了 UsernamePasswordAuthenticationToken，然后设置了 Details，最后调用 AuthenticationManager 对用户输入的用户名和密码组成的身份认证信息进行身份认证。

如上面所说，AuthenticationManager 包括了很多的 AuthenticationProvider，各个 AuthenticationProvider 都可以去对 AuthenticationManager 进行身份认证，如果认证成功，由于 UsernamePasswordAuthenticationToken 的 setAuthenticated(boolean isAuthenticated) 在把参数值设成 true 时会抛出异常，所以大多数情况可以重新实例化一个身份认证过的 UsernamePasswordAuthenticationToken（参见构造函数）替换当前 Authentication 实例化的对象并返回。一旦 AuthenticationManager 中的某个 AuthenticationProvider 认证成功，其余 AuthenticationProvider 就可以调用 Authentication 的 isAuthenticated() 方法发现身份认证成功，从而不再进行身份认证。
