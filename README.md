# nginx_cheat_sheet

user  nginx;
worker_processes  5;

error_log  /var/log/nginx.error.log;
pid        /var/run/nginx.pid;

events {
  worker_connections  1024;
}

http {
  include       mime.types;
  default_type  application/octet-stream;

  log_format    main  '$remote_addr - $remote_user [$time_local] $request '
                      '"$status" $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

  access_log    /var/log/nginx.access.log  main;

  sendfile      on;

  keepalive_timeout  65;

  upstream thin_cluster {
    server unix:/tmp/thin.0.sock;
    server unix:/tmp/thin.1.sock;
    server unix:/tmp/thin.2.sock;
    server unix:/tmp/thin.3.sock;
    server unix:/tmp/thin.4.sock;
  }

  server {
    listen       80;
    server_name  www.myserver.com;

    root /var/rails/mysapp/public;

    location / {
      proxy_set_header  X-Real-IP  $remote_addr;
      proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
      proxy_redirect false;

      if (-f $request_filename/index.html) {
        rewrite (.*) $1/index.html break;
      }
      if (-f $request_filename.html) {
        rewrite (.*) $1.html break;
      }
      if (!-f $request_filename) {
        proxy_pass http://thin_cluster;
        break;
      }
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
      root   html;
    }
  }
}
