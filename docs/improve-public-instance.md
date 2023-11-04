# Improve your public instance performance and stability

This tutorial has been written by [unixfox](https://github.com/unixfox), owner of [yewtu.be](https://yewtu.be/). He is better suited when looking for help about this tutorial.

## Synopsis

This tutorial will explain how to improve the performance, and decrease the memory and CPU consumption of your public instance on your server.

Using Docker is recommended for this tutorial because the process is simpler with it. You can try it outside of Docker by adapting the ideas of this tutorial, but the main focus will be Docker.

## Instructions

### 1) Multiple Invidious processes

Invidious is single threaded, so by running multiple processes you better utilize the multiple threads of your server.  
Also past a certain amount of requests, Invidious becomes sluggish. Having multiple processes reduces this sluggishness.

This tutorial will explain how to run multiple containers of Invidious and use NGINX in Docker as a loadbalancer.  
If your reverse proxy is already in Docker you can also adapt it so that it sends the requests directly to the Invidious containers.

We assume that you have not changed the port `3000` from the default installation. But if you changed it then you will need to adapt it in the following configuration files and commands.

1. In your `docker-compose.yml`, remove these lines for binding the port of Invidious:
   ```
   ports:
    - "127.0.0.1:3000:3000"
   ```
2. Create a new bash script that you will now use to start Invidious, name it `start.sh`:
   ```
   #!/bin/sh
   docker compose up -d --scale invidious=6
   ```
   Explanation: The `--scale` parameter allows running multiple containers of the same Docker image.  
   Note: You can set more or less Invidious processes (6 in the example).  
   **Don't restart Invidious yet!**
3. Create a file called `nginx.conf` and add this content:
   ```
   user www-data;
   events {
        worker_connections 1000;
   }
   http {
        server {
            listen 3000;
            listen [::]:3000;
            access_log off;
            location / {
                proxy_pass http://invidious:3000;
                proxy_http_version 1.1; # to keep alive
                proxy_set_header Connection ""; # to keep alive
            }
        }
   }
   ```
4. Add a new service in your `docker-compose.yml` file:
   ```
   nginx:
      image: nginx:latest
      restart: unless-stopped
      volumes:
        - ./nginx.conf:/etc/nginx/nginx.conf:ro
      depends_on:
        - invidious
      ports:
        - "127.0.0.1:3000:3000"
   ```

5. Update your cronjobs to restart Invidious (if you use cron).
   Instead of restarting a single Docker container you will now need to restart 6 containers (adjust if you added or removed number of containers).
   Replace with these CRON lines:
   ```
   0 */1 * * * docker restart invidious-invidious-1
   1 */1 * * * docker restart invidious-invidious-2
   2 */1 * * * docker restart invidious-invidious-3
   3 */1 * * * docker restart invidious-invidious-4
   4 */1 * * * docker restart invidious-invidious-5
   5 */1 * * * docker restart invidious-invidious-6
   ```
   Each CRON line has a different schedule to avoid disrupting your entire Invidious instance due a restart.
6. Apply the new configuration:
   ```
   docker compose down
   chmod +x ./start.sh
   ./start.sh
   ```

### 2) Speed up video playback with http3-ytproxy

Kavin from the Piped team has developed a tool that is much faster at proxying the traffic for video playback and image loading than Invidious: https://github.com/TeamPiped/http3-ytproxy

NGINX configuration will be used for this tutorial, and it's highly recommended to setup this configuration on **your main reverse proxy**.

But if you do not have NGINX as **your main reverse proxy** you can either try to adapt the rules to your reverse proxy. Or you can also use the separate NGINX container that you created in the [first section](#1-multiple-invidious-processes).

1. Find the username of NGINX process, you can find it at the top of the file `/etc/nginx/nginx.conf`.  
   Get its uid and gid using the `id` command like so: `id www-data`.  
   Usually it's uid `33` and gid `33`.
2. Create a new directory for this new service and give the correct permissions for the user of NGINX:
   ```
   mkdir /opt/http3-ytproxy
   chown 33:33 -R /opt/http3-ytproxy
   ```
   Replace `33:33` with uid:gid if you have something different.
3. Add this new service to your `docker-compose.yml`:
   ```
   http3-ytproxy:
      image: 1337kavin/ytproxy:latest
      restart: unless-stopped
      user: 33:33
      environment:
        - DISABLE_WEBP=1
      volumes:
        - /opt/http3-ytproxy:/app/socket
   ```
   Replace `33:33` with uid:gid if you have something different.
4. Add these lines for the "server" section of Invidious in your NGINX configuration, just after the block `location /`:
   ```
   location ~ (^/videoplayback|^/vi/|^/ggpht/|^/sb/) {
        proxy_buffering on;
        proxy_buffers 1024 16k;
        proxy_set_header X-Forwarded-For "";
        proxy_set_header CF-Connecting-IP "";
        proxy_hide_header "alt-svc";
        sendfile on;
        sendfile_max_chunk 512k;
        tcp_nopush on;
        aio threads=default;
        aio_write on;
        directio 16m;
        proxy_hide_header Cache-Control;
        proxy_hide_header etag;
        proxy_http_version 1.1;
        proxy_set_header Connection keep-alive;
        proxy_max_temp_file_size 32m;
        access_log off;
        proxy_pass http://unix:/opt/http3-ytproxy/http-proxy.sock;
        add_header Cache-Control private always;
   }
   ```
   If you're using the NGINX from the [first section](#1-multiple-invidious-processes), you will need to add this new volume:
   ```
   volumes:
    - /opt/http3-ytproxy:/opt/http3-ytproxy
   ```
5. Reload the docker composition: `./start.sh` (if you followed the [first section](#1-multiple-invidious-processes))
6. Reload NGINX: `systemctl reload nginx`.