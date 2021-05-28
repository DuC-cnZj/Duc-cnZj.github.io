---
title: OAuth2 & OpenID connect
date: '2021-02-10 11:11:14'
sidebar: false
categories:
 - 技术
tags:
 - OAuth2
 - OpenID connect
publish: true
---

## oauth2 参考

https://oauth.net/2/

http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html

https://github.com/go-oauth2/oauth2

https://github.com/golang/oauth2

https://github.com/llaoj/oauth2

https://www.rfc-editor.org/rfc/rfc6749.html

https://colobu.com/2017/04/28/oauth2-rfc6749/



## openid connect

https://www.cnblogs.com/newton/p/13220207.html

https://github.com/ory/fosite

https://github.com/ory/hydra

https://www.infoq.cn/article/euvhttyf3jmfakmm8cmn

https://www.cnblogs.com/linianhui/p/openid-connect-core.html

https://openid.net/specs/openid-connect-core-1_0.html

https://openid.net/specs/openid-connect-discovery-1_0.html

https://www.mdeditor.tw/pl/peiZ

https://developers.google.com/identity/protocols/oauth2/openid-connect#discovery

https://github.com/jumbojett/OpenID-Connect-PHP

https://www.loginradius.com/blog/async/pkce/



https://tools.ietf.org/html/rfc7009#section-2.1



### Api设计文档参考



[openid + oauth2 api 设计文档](https://developer.okta.com/docs/reference/api/oidc/)

[openid connect api 设计文档](https://openid.net/specs/openid-connect-core-1_0.html)

[oauth2 api 设计文档](https://colobu.com/2017/04/28/oauth2-rfc6749/)



### 在线测试工具

[oauth2](https://oauthdebugger.com/)
[openid](https://oidcdebugger.com/)



### PKCE


> 需要在客户端处理
>
> 一共三个参数 `code_challenge`、`code_challenge_method`、`code_verifier`
> 
> 其中 `code_challenge` `code_challenge_method` 是需要跳转 `url` 需要带上的
>
> `code_verifier` 通过 `code_challenge_method` 加密后生成 `code_challenge`
>
> 通过  code 换取 /userinfo 时，服务端会用 `code_challenge_method` 校验 `code_verifier`，是否等于 `code_challenge` 
>
> 主要用于 没有 `cookie` 和 `session` 机制的 app 的客户端(public)，因为 app 端保存 secret 不安全，原理和 `CSRF token` 一致

1. 需要在跳转授权的url后面加上 `&code_challenge="+pkceCodeChallenge + "&code_challenge_method=S256`
2. 然后在 exchange 的 AuthCodeOption 中带上 code_verifier 参数


## Session logout

> https://openid.net/specs/openid-connect-backchannel-1_0.html
>
> https://openid.net/specs/openid-connect-frontchannel-1_0.html



```go
// Logout
func (oauth *OAuthController) Logout(ctx *gin.Context) {
	var (
		hint                 = ctx.Query("id_token_hint")
		requestedRedirectUrl = ctx.Query("post_logout_redirect_uri")
	)

	session.DeleteIdentifier(ctx)

	if hint != "" {
		if _, e := oauth.jwtStrategy.Decode(context.Background(), hint); e == nil {
			var clientIds []string
			keys, _ := session.Get(ctx, OAuth2ClientIdsSessionKey)
			if keys != nil {
				clientIds = strings.Split(keys.(string), ",")
			}

			type Task struct {
				BackChannelLogoutURI string
				ClientID             string
				Token                string
			}
			var tasks []Task
			for _, id := range clientIds {
				for _, u := range utils.Config().GetClient(id).LogoutUrls {
					if u.Channel == config.BackChannel {
						token, _, _ := oauth.jwtStrategy.Generate(ctx, jwt2.MapClaims{
							"iss":    utils.FullUrlFromRequest("", ctx.Request),
							"aud":    []string{id},
							"iat":    time.Now().UTC().Unix(),
							"jti":    uuid.New(),
							"events": map[string]struct{}{"http://schemas.openid.net/event/backchannel-logout": {}},
						}, &jwt.Headers{})
						tasks = append(tasks, Task{
							BackChannelLogoutURI: u.Url,
							ClientID:             id,
							Token:                token,
						})
					}
				}
			}

			slog.Debug(tasks)
			wg := sync.WaitGroup{}
			wg.Add(len(tasks))
			for _, task := range tasks {
				go func(task Task) {
					defer wg.Done()

					var e error
					httpClient := http_client.NewClientRetryWithTimeout(2 * time.Second)
					uv := url.Values{"logout_token": {task.Token}}
					req, e := http.NewRequest(http.MethodPost, task.BackChannelLogoutURI, strings.NewReader(uv.Encode()))
					if e != nil {
						slog.Error(e)
						return
					}
					req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
					_, e = httpClient.Do(req)
					if e != nil {
						slog.Error(e)
					}
				}(task)
			}
			wg.Wait()

			frontendLogoutUrls := utils.Config().GetFrontendLogoutUrlsByClientIDs(clientIds...)
			if requestedRedirectUrl == "" {
				requestedRedirectUrl = "/login"
			}

			slog.Debugf("requestedRedirectUrl %s GetFrontendLogoutUrlsByClientIDs %v frontendLogoutUrls %v", requestedRedirectUrl, clientIds, frontendLogoutUrls)
			response.Html(ctx, http.StatusOK, "session-logout.html", &LogoutResult{
				RedirectTo:             requestedRedirectUrl,
				FrontChannelLogoutURLs: frontendLogoutUrls,
			})
			return
		} else {
			slog.Error(e)
		}
	}

	response.Back(ctx)
}
```



### backend logout

```php
$oidc = new OidcClient(
  'http://127.0.0.1:9001',
  'app1',
  'app1_secret'
);
$oidc->setRedirectURL('http://127.0.0.1:8001/oauth/callback');
if ($oidc->verifyJWTsignature($request->logout_token)) {
  //do logout
	// token info:   {"aud":["app1"],"events":{"http://schemas.openid.net/event/backchannel-logout":[]},"iat":1618995375,"iss":"http://127.0.0.1:9001","jti":"70687f04-80f0-4c49-8a64-34b58f42bfdf","sub":"唯一标识符"}}
  // 根据 sub 来注销用户
  // decodeToken 自行实现
  info("dec", [$oidc->decodeToken($request->logout_token, 1)]);
  info('do logout');
}
```


