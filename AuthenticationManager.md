# AuthenticationManager

## AuthenticationManager 是什么

AuthenticationManager只有一个方法，如下所示。它主要的作用是认证一个认证信息是否正确。一般的做法是传入一个 isAuthenticated = false 的 Authentication 对象， 然后根据信息进行认证，如果信息正确，返回一个 isAuthenticated = true 的 Authentication 对象，否则返回 null。

```
	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;
```
## AuthenticationManager 实现

AuthenticationManager 实现很少，主要是 ProviderManager。ProviderManager 保存了 List<AuthenticationProvider>。认证的做法按照在 List 中的顺序，让每一个 AuthenticationProvider 认证 Authentication 对象，如果有一个 AuthenticationProvider 的认证得出的 Authentication 对象的 isAuthenticated = true， 就返回该对象。如果没有任何 AuthenticationProvider 的认证得出的 Authentication 对象的 isAuthenticated = true，则抛出 ProviderNotFoundException，标明认证失败。
```
		for (AuthenticationProvider provider : getProviders()) {
			if (!provider.supports(toTest)) {
				continue;
			}

			if (debug) {
				logger.debug("Authentication attempt using "
						+ provider.getClass().getName());
			}

			try {
				result = provider.authenticate(authentication);

				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}
			catch (AccountStatusException e) {
				prepareException(e, authentication);
				// SEC-546: Avoid polling additional providers if auth failure is due to
				// invalid account status
				throw e;
			}
			catch (InternalAuthenticationServiceException e) {
				prepareException(e, authentication);
				throw e;
			}
			catch (AuthenticationException e) {
				lastException = e;
			}
		}
```

拿现在常用的登录系统来说，一般会有用户名密码登录、验证码登录。一般的做法是，首先创建一个 Filter，该 Filter 组建好 Authentication 对象，包括需要的认证信息，交由 AuthenticationManager 处理。AuthenticationManager 用其 List<AuthenticationProvider> 去认证 Authentication 对象，这里的 AuthenticationProvider 包括用户名密码登录、短信验证码登录。认证成功，在 SecuirtyContextHolder 中设置认证对象，表明认证成功。

