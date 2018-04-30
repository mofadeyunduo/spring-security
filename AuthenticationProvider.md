# AuthenticationProvider

## AuthenticationProvider 是什么

AuthenticationProvider 是对认证 Authentication 的服务提供商。一个 Authentication 对象，可能是别人实例化出来的、保存了一些认证信息（比如用户名和密码）、未经认证过（只是保存了用户名和密码，但没有校验，isAuthenticated = true）的对象。这个时候需要有个对象去认证 Authentication 对象。

这样写的好处是，实际应用中，大多数认证方法流程都是固定的，比如密码登录，步骤无非是找用户，比较密码。这样写死的流程，Spring Security 完全可以提前写好密码登陆流程代码，开发者只需要提供一个寻找用户名密码的类即可。此外，一般系统中，认证方式可能有多种多样，比如密码登录、验证码登录、邮箱登陆，这个时候一般会有很多的认证服务提供商，一般需要组合其中两个或多个使用，这时利用 AuthenticationManager 添加 AuthenticationProvider，AuthenticationManager 调用所有 AuthenticationProvider 的 authenticate 方法进行认证。

看一下他的接口：

```
public interface AuthenticationProvider {

	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;
      
	boolean supports(Class<?> authentication);
  
}
````

authenticate() 方法需要实现认证。Authentication 存储了认证信息，AuthenticationProvider 的实现类可以根据认证信息修改 Authentication 对象，甚至于返回一个新的认证对象。
supports() 方法表明 AuthenticationProvider 的实现类是否支持认证信息，Class<?> authentication 表明认证对象的类型。比如说密码登录是 UsernamePasswordAuthentication，验证码登录是 ValidationCodeAuthentication，显然，ValidationCodeAuthenticationProvider 的实现类只支持 ValidationCodeAuthentication，这个时候就可以用类型去判断是否支持该认证服务商。一般会写成形如下面的形式。
```
	public boolean supports(Class<?> authentication) {
		return (RememberMeAuthenticationToken.class.isAssignableFrom(authentication));
	}
```

## AuthenticationProvider 的实现

例如 AbstractUserDetailsAuthenticationProvider。supports 方法就不贴上来了，很简单的类型判断。authenticate 方法也很简单，首先从认证信息中提取用户认证信息，再根据用户认证信息查找存在的用户信息，查找可以通过缓存或者别的方式，查找前后会做很多校验操作，比如验证读取到用户信息等。

```
	public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
		Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
				messages.getMessage(
						"AbstractUserDetailsAuthenticationProvider.onlySupports",
						"Only UsernamePasswordAuthenticationToken is supported"));

		// Determine username
		String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
				: authentication.getName();

		boolean cacheWasUsed = true;
		UserDetails user = this.userCache.getUserFromCache(username);

		if (user == null) {
			cacheWasUsed = false;

			try {
				user = retrieveUser(username,
						(UsernamePasswordAuthenticationToken) authentication);
			}
			catch (UsernameNotFoundException notFound) {
				logger.debug("User '" + username + "' not found");

				if (hideUserNotFoundExceptions) {
					throw new BadCredentialsException(messages.getMessage(
							"AbstractUserDetailsAuthenticationProvider.badCredentials",
							"Bad credentials"));
				}
				else {
					throw notFound;
				}
			}

			Assert.notNull(user,
					"retrieveUser returned null - a violation of the interface contract");
		}

		try {
			preAuthenticationChecks.check(user);
			additionalAuthenticationChecks(user,
					(UsernamePasswordAuthenticationToken) authentication);
		}
		catch (AuthenticationException exception) {
			if (cacheWasUsed) {
				// There was a problem, so try again after checking
				// we're using latest data (i.e. not from the cache)
				cacheWasUsed = false;
				user = retrieveUser(username,
						(UsernamePasswordAuthenticationToken) authentication);
				preAuthenticationChecks.check(user);
				additionalAuthenticationChecks(user,
						(UsernamePasswordAuthenticationToken) authentication);
			}
			else {
				throw exception;
			}
		}

		postAuthenticationChecks.check(user);

		if (!cacheWasUsed) {
			this.userCache.putUserInCache(user);
		}

		Object principalToReturn = user;

		if (forcePrincipalAsString) {
			principalToReturn = user.getUsername();
		}

		return createSuccessAuthentication(principalToReturn, authentication, user);
	}
```
