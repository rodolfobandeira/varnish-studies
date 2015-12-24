# varnish-studies
Collection of vcl files and configs that I've been using to study Varnish 4.* Cache

---

```
vcl 4.0;

backend default {
    .host = "127.0.0.1";
    .port = "8080";
    .max_connections = 300;
    .first_byte_timeout = 300s;  
}

sub vcl_recv {
    set req.http.Host = regsub(req.http.Host, ":[0-9]+", "");
    
    # SQL Injection Detector
    if (
        req.url ~ "(?i).+SELECT.+FROM" || 
        req.url ~ "(?i).+UNION\s+SELECT" || 
        req.url ~ "(?i).+UPDATE.+SET" || 
        req.url ~ "(?i).+INSERT.+INTO" ||
        req.url ~ "(?i).+DELETE.+FROM" ||
        req.url ~ "(?i).+ASCII\(.+SELECT" ||
        req.url ~ "(?i).+DROP.+TABLE" ||
        req.url ~ "(?i).+DROP.+DATABASE" ||
        req.url ~ "(?i).+SELECT.+VERSION" ||
        req.url ~ "(?i).+SHOW.+CUR(DATE|TIME)" ||
        req.url ~ "(?i).+SELECT.+SUBSTR" ||
        req.url ~ "(?i).+SELECT.+INSTR" ||
        req.url ~ "(?i).+SHOW.+CHARACTER.+SET" ||
        req.url ~ "(?i).+BULK.+INSERT" ||
        req.url ~ "(?i).+INSERT.+VALUES" ||
        req.url ~ "(?i).+\%2F\%2A.+\%2A\%2F" ||
        req.url ~ "(?i).+SELECT.+CONCAT"
    ) { 
        return (synth(403, "really?"));
    }

    # Remove cookies from static files
    if (req.url ~ "^[^?]*\.(7z|avi|bmp|bz2|css|csv|doc|docx|eot|flac|flv|gif|gz|ico|jpeg|jpg|js|less|mka|mkv|mov|mp3|mp4|mpeg|mpg|odt|otf|ogg|ogm|opus|pdf|png|ppt|pptx|rar|rtf|svg|svgz|swf|tar|tbz|tgz|ttf|txt|txz|wav|webm|webp|woff|woff2|xls|xlsx|xml|xz|zip)(\?.*)?$") {
        unset req.http.Cookie;
        return (hash);
    }
}

sub vcl_backend_response {

    # Add GZIP compression for text files as css/html/json/text
    if (beresp.http.content-type ~ "text") {
        set beresp.do_gzip = true;
    }

    # Enables a good cache for static files
    if (beresp.ttl > 0s) {
        unset beresp.http.expires;
        set beresp.http.cache-control = "max-age=31536000";
        set beresp.ttl = 1w;
        set beresp.http.magicmarker = "1";
    }
}

sub vcl_deliver {
    if (resp.http.magicmarker) {
        unset resp.http.magicmarker;
        set resp.http.age = "0";
    }
}

# Remove that ugly "Guru Meditation" Varnish Error Page :)
sub vcl_synth {
    set resp.http.Content-Type = "text/html; charset=utf-8";
    set resp.http.Retry-After = "5";
    synthetic( {"<!DOCTYPE html>
<html>
  <head>
    <title>"} + resp.status + " " + resp.reason + {"</title>
  </head>
  <body>
    <div align="center">
        <p><img src="http://www.quickmeme.com/img/c2/c2e96daa9eb1fc8750bfbdb0e52f7586366a36deee97ae6d7f449dbfd5ee3d25.jpg" border="0" /></p>
    </div>
        <hr>
        <p>rodolfo.io web firewall</p>
  </body>
</html>
"} );
    return (deliver);
}
```
