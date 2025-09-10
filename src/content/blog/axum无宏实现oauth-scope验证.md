---
title: axum无宏实现oauth scope验证
date: 2024-09-30 21:23:39
tags: [rust, axum]
---

使用`axum`编写普通`jwt`校验的时候可以很方便的用`FromRequestParts`trait来实现。但是如果是带有`scope`的`oauth`就比较麻烦，一种方案是在每一个handler中从请求提取校验或者使用中间件匹配每个path，但是总是感觉这样不够简洁优雅，是否存在更方便的方法呢？

一种方法是使用宏。

另一种可以通过`常量泛型`来实现。将需要的`scope`写进泛型常量里，这样就可以在`from_request_parts`中得知需要的`scope`，从而变得和编写普通`jwt`一样简单。
利用`bitmap`储存所需的`scope`。

## 原理

例如这样定义一个`struct`
```rust
#[derive(Debug, Serialize, Deserialize)]
struct Claims<const S: u8> {
    sub: String,
    scope: Vec<String>,
    exp: usize,
}
```
假设读权限是`0b00000000`，写权限是`0b00000001`。那么一个需要读权限的`Claims`就可以写成`Claims<0b00000000>`，同时需要读写权限就可以写成`Claims<0b00000001>`。然后在`from_request_parts`中进行校验。

封装一下权限计算的逻辑
```rust
const fn to_scopes(scopes: &[Scope]) -> u8 {
    let mut result = 0u8;
    let mut i = 0;

    while i < scopes.len() {
        match scopes[i] {
            Scope::Read => result |= 1 << 0,
            Scope::Write => result |= 1 << 1,
        }

        i += 1;
    }

    result
}

#[derive(Debug)]
enum Scope {
    Read,Write
}
```
注意这里需要加上`#![allow(long_running_const_eval)]`来允许在常量函数中使用`while i < scopes.len()`。这里所有的权限都是在编译时确定的，并且不是死循环。

这样定义可以进一步简写成`Claims<{ to_scopes(&[Scope::Read])}>`和`Claims<{ to_scopes(&[Scope::Read, Scope::Write])}>`。

然后实现`FromRequestParts`trait，添加验证的逻辑，通过将传入的
```rust
#[async_trait]
impl<const S: u8, T> FromRequestParts<T> for Claims<S>
where
    T: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &T) -> Result<Self, Self::Rejection> {
        // Extract the token from the authorization header
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;
        // Decode the user data
        let token_data = decode::<Claims<S>>(bearer.token(), &KEYS.decoding, &Validation::default())
            .map_err(|_| AuthError::InvalidToken)?;

        let scope: Vec<Scope> = token_data.claims.scope.iter().filter_map(|s| {
            match s.as_str() {
                "read" => Some(Scope::Read),
                "write" => Some(Scope::Write),
                _ => None,
            }
        }).collect();

        // 这里将提供的scope与需要的scope进行按位于，实现scope的包含关系，例如需要[read]权限，提供[read,write]也能通过验证。
        if S == S & to_scopes(scope.deref()) {
            return Ok(token_data.claims);
        }

        Err(AuthError::MissingScope)
    }
}

/*
 * 省略部分关于AuthError的代码
 */
```

之后可以在handler中使用了
```rust
async fn read_protected(claims: Claims<{ to_scopes(&[Scope::Read])}>) -> Result<String, AuthError> {
    Ok(format!("read successfully! data: \n{:?}",claims))
}

async fn write_protected(claims: Claims<{ to_scopes(&[Scope::Read, Scope::Write])}>) -> Result<String, AuthError> {
    Ok(format!("write successfully! data: \n{:?}", claims))
}
```

## 测试
下面是完整代码
```rust
#![allow(long_running_const_eval)]

use axum::{async_trait, extract::FromRequestParts, http::{request::Parts, StatusCode}, response::{IntoResponse, Response}, routing::{get, post}, Json, RequestPartsExt, Router};
use axum_extra::{
    headers::{authorization::Bearer, Authorization},
    TypedHeader,
};
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use once_cell::sync::Lazy;
use serde::{Deserialize, Serialize};
use serde_json::json;
use std::ops::Deref;

static KEYS: Lazy<Keys> = Lazy::new(|| {
    Keys::new(b"test")
});

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/read-protected", get(read_protected))
        .route("/write-protected", get(write_protected))
        .route("/authorize", post(authorize));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::debug!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn read_protected(claims: Claims<{ to_scopes(&[Scope::Read])}>) -> Result<String, AuthError> {
    Ok(format!("read successfully! data: \n{:?}",claims))
}

async fn write_protected(claims: Claims<{ to_scopes(&[Scope::Read, Scope::Write])}>) -> Result<String, AuthError> {
    Ok(format!("write successfully! data: \n{:?}", claims))
}

async fn authorize(Json(payload): Json<AuthPayload>) -> Result<Json<String>, AuthError> {
    let claims = Claims::<0> {
        sub: payload.sub,
        scope: payload.scope,
        exp: 10000000000,
    };
    let token = encode(&Header::default(), &claims, &KEYS.encoding)
        .map_err(|_| AuthError::TokenCreation)?;

    Ok(Json(token))
}

#[async_trait]
impl<const S: u8, T> FromRequestParts<T> for Claims<S>
where
    T: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &T) -> Result<Self, Self::Rejection> {
        // Extract the token from the authorization header
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| AuthError::InvalidToken)?;
        // Decode the user data
        let token_data = decode::<Claims<S>>(bearer.token(), &KEYS.decoding, &Validation::default())
            .map_err(|_| AuthError::InvalidToken)?;

        let scope: Vec<Scope> = token_data.claims.scope.iter().filter_map(|s| {
            match s.as_str() {
                "read" => Some(Scope::Read),
                "write" => Some(Scope::Write),
                _ => None,
            }
        }).collect();

        if S == to_scopes(scope.deref()) {
            return Ok(token_data.claims);
        }

        Err(AuthError::MissingScope)
    }
}

impl IntoResponse for AuthError {
    fn into_response(self) -> Response {
        let (status, error_message) = match self {
            AuthError::TokenCreation => (StatusCode::INTERNAL_SERVER_ERROR, "Token creation error"),
            AuthError::InvalidToken => (StatusCode::BAD_REQUEST, "Invalid token"),
            AuthError::MissingScope => (StatusCode::UNAUTHORIZED, "Missing scope"),
        };
        let body = Json(json!({
            "error": error_message,
        }));
        (status, body).into_response()
    }
}

const fn to_scopes(scopes: &[Scope]) -> u8 {
    let mut result = 0u8;
    let mut i = 0;

    while i < scopes.len() {
        match scopes[i] {
            Scope::Read => result |= 1 << 0,
            Scope::Write => result |= 1 << 1,
        }

        i += 1;
    }

    result
}

#[derive(Deserialize)]
struct AuthPayload {
    sub: String,
    scope: Vec<String>,
}

struct Keys {
    encoding: EncodingKey,
    decoding: DecodingKey,
}

impl Keys {
    fn new(secret: &[u8]) -> Self {
        Self {
            encoding: EncodingKey::from_secret(secret),
            decoding: DecodingKey::from_secret(secret),
        }
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct Claims<const S: u8> {
    sub: String,
    scope: Vec<String>,
    exp: usize,
}
#[derive(Debug)]
enum AuthError {
    TokenCreation,
    InvalidToken,
    MissingScope,
}
#[derive(Debug)]
enum Scope {
    Read,Write
}
```

先获取只有`Read`权限的token
```bash
curl http://127.0.0.1:3000/authorize -d '{"sub": "test", "scope": ["read"]}' -H "Content-Type: application/json"
```
然后再请求需要`Read`权限的api
```bash
curl http://127.0.0.1:3000/read-protected -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ0ZXN0Iiwic2NvcGUiOlsicmVhZCJdLCJleHAiOjEwMDAwMDAwMDAwfQ.za3wMPdXvXLiLa7JZIEWbPGM17mXgx0D6-SHU-HclDc"
```
可以得到
```bash
read successfully! data: 
Claims { sub: "test", scope: ["read"], exp: 10000000000 }
```
请求需要`Read`和`Write`权限的api
```bash
curl http://127.0.0.1:3000/write_protected -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ0ZXN0Iiwic2NvcGUiOlsicmVhZCJdLCJleHAiOjEwMDAwMDAwMDAwfQ.za3wMPdXvXLiLa7JZIEWbPGM17mXgx0D6-SHU-HclDc"
```
得到
```bash
{"error":"Missing scope"}
```
验证成功！

## 总结
使用常量泛型和bitmap可以实现简单简洁的oauth scope验证和转换处理。宏也是一种很好的解决方法，可以深度定制，处理更复杂的情况。