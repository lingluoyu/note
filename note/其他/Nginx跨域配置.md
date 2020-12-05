#### Nginx跨域配置


```
location ~ \.htm$ {

    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS,PUT,DELETE' always;
        add_header 'Access-Control-Allow-Headers' '*' always;
        add_header 'Access-Control-Max-Age' 1728000 always;
        add_header 'Content-Length' 0;
        add_header 'Content-Type' 'text/plain; charset=utf-8';
        return 204;
    }

    if ($request_method ~* '(GET|POST|DELETE|PUT)') {
        add_header 'Access-Control-Allow-Origin' '*' always;
    }

}
```