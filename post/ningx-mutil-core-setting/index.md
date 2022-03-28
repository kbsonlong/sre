### 单域名跨域

```bash
location /{
    #add_header Access-Control-Allow-Origin *; #允许所有域名不安全
    add_header 'Access-Control-Allow-Origin' 'https://www.sundayle.com';
    add_header 'Access-Control-Allow-Credentials' 'true';
    add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS';
    add_header 'Access-Control-Allow-Headers' 'Authorization,Content-Type,Accept,Origin,User-Agent,DNT,Cache-Control,X-Mx-ReqToken,X-Requested-With';
    if ($request_method = 'OPTIONS') {
        return 204;
    }
...
}
```

### 允许多域名跨域

#### 方法一 if
```conf
server {
set $allow_origin "";
if ( $http_origin ~ '^https?://(www|m).sundayle.com' ) {
      set $allow_origin $http_origin;
}
    location /{
        add_header 'Access-Control-Allow-Origin' $allow_origin;
        add_header 'Access-Control-Allow-Credentials' 'true';
        add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'Token,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,X_Requested_With,If-Modified-Since,Cache-Control,Content-Type';
        if ($request_method = 'OPTIONS') {
            return 204;
        }
...
}
```

#### 方法二 map

```conf

map $http_origin $allow_origin {
    default "";
    "~^(https?://localhost(:[0-9]+)?)" $1;
    "~^(https?://127.0.0.1(:[0-9]+)?)" $1; 
    "~^(https?://192.168.10.[\d]+(:[0-9]+)?)" $1; 
    "~^https://www.sunday.com" https://www.sundayle.com;
    "~^https://m.sundayle.com" https://m.sundayle.com;
    "~^(https?://[\w]+.open.sundayle.com)" $1;
    #"~^(https?://([\w]+.)?[\w]+.open.sundayle.com)" $1;  #允许一级和二级域名

}

server {
    location /{
        add_header 'Access-Control-Allow-Origin' $allow_origin;
        add_header 'Access-Control-Allow-Credentials' 'true';
        add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'Token,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,X_Requested_With,If-Modified-Since,Cache-Control,Content-Type';
        if ($request_method = 'OPTIONS') {
            return 204;
        }
...
}
```

### Nginx更多IF判断
```conf
location / {
     if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Allow-Origin' 'https://www.sundayle.com';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
        add_header 'Access-Control-Max-Age' 1728000; # 20days
        add_header 'Content-Type' 'text/plain; charset=utf-8';
        add_header 'Content-Length' 0;
        return 204;
     }
     if ($request_method = 'POST') {
        add_header 'Access-Control-Allow-Origin' 'https://www.sundayle.com';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
        add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
     }
     if ($request_method = 'GET') {
        add_header 'Access-Control-Allow-Origin' 'https://www.sundayle.com';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
        add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
     }
}
```

