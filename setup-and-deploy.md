# Setup and Deploy

Here is a quick description of how I got started with the application by setting everything up and deploying it.

# Docker compose 

The decision to run the application inside Docker containers was made due to the practicality of the Docker image system and its ability to isolate different parts of the application from each other, providing environment consistency and clean dependency isolation.

With that in mind, this instance uses the [decidim-generator image repository](https://github.com/decidim/docker) to generate its own Decidim application.
The docker-compose file defines four services: the application itself, PostgreSQL, Redis, and a background worker (Sidekiq). Both PostgreSQL and Redis have persistent volumes to ensure data isn't lost between container restarts. Each service can be scaled, restarted, or debugged independently without affecting the others.

For production restart policies are set to `unless-stopped`, meaning containers automatically restart unless explicitly stopped by the user.

`depends_on` only guarantees that containers start in order, it doesn't mean the database is actually ready to accept connections. To fix this, in the production environment pg has a healthcheck and app/worker wait for it to report `service_healthy` before starting. This avoids a common deploy failure: the app trying to connect before the database has finished initializing.

Also, development and production use separate `env_files` and separate compose files `docker-compose.yml` and `docker-compose-prod.yml`. This avoids the risk of accidentally running production credentials locally, and keeps environment specific values out of the compose files themselves, which are version-controlled.

Deployment is currently done manually, but CI/CD automation is planned for a future iteration.

# Maintenance and Hosting

There is a repository dedicated to keeping track of all the changes made in the Decidim-RS application, you can clone it and access it yourself. Note, however, that we try to keep the maintenance as similar as possible to the original Decidim application, due to the difficulty of updating to each new version of Decidim otherwise.

Speaking of versions, this application currently runs on version 0.30.9, due to the perceived stability of a version that is early but not the earliest available, which will, of course, change over time, and so will our application.

It is also important to note that this application is hosted on an Ubuntu VM server, acessed via ssh, hosted by my university, with thease especifications:
- Ubuntu version - 22.04
- RAM - 4.0Gi

# Backups 

For the backups, at first there was only a crontab command that would copy the entire database each day:

```
0 3 * * * /usr/bin/docker exec production-pg-1 pg_dump -U postgres decidim_production > /home/backups/backup_decidim_$(date +\%F).sql
```

However, this approach was soon realized to be a mistake, dumps accumulated indefinitely in storage with no cleanup policy, and the plain text .sql format used more space than necessary. The solution was to automate backups using a crontab + bash script approach. It was decided to switch the plain text dumps for PostgreSQL custom format `-Fc`, also that dumps would be made every day for 14 days and then recycled. Here is a quick recap of the bash instructions that made this possible:

The script executes a full dump inside the app's container for the PostgreSQL database, checks the exit status of the operation, and logs the result to the log file:

```
if docker exec production-pg-1 pg_dump -U postgres -Fc decidim_production > "$FILE"; then
	logger -t backup-postgres "BACKUP COMPLETED: $FILE ($(du -h "$FILE" | cut -f1))"
else
	logger -t backup-postgres -p user.err "BACKUP ERROR"
	rm -f "$FILE" 
	exit 1
fi
```

As the platform progresses, it is worth noting that it might be a good idea to consider switching from full dumps to incremental ones, in order to scale more efficiently, minimize space consumption, and lose the least amount of data possible. This comes with the caveat since incremental backups tend to be more complex to implement.

## a brief on crontab atomations 

Decidim recommends some automations of its own:

```
# Remove expired download your data files
0 0 * * * cd /home/user/decidim_application && RAILS_ENV=production bundle exec rake decidim:delete_download_your_data_files

# Compute open data
2 0 * * * cd /home/user/decidim_application && RAILS_ENV=production bundle exec rake decidim:open_data:export

# Delete old registrations forms
3 0 * * * cd /home/user/decidim_application && RAILS_ENV=production bundle exec rake decidim_meetings:clean_registration_forms

# Generate reminders
4 0 * * * cd /home/user/decidim_application && RAILS_ENV=production bundle exec rake decidim:reminders:all

# Send notification mail digest daily
5 0 * * * cd /home/user/decidim_application && RAILS_ENV=production bundle exec rake decidim:mailers:notifications_digest_daily

# Send notification mail digest weekly on saturdays
5 0 * * 6 cd /home/user/decidim_application && RAILS_ENV=production bundle exec rake decidim:mailers:notifications_digest_weekly

# Change active step in participatory processes
*/15 * * * * cd /home/user/decidim_application && RAILS_ENV=production bundle exec rake decidim_participatory_processes:change_active_step

# Delete inactive participants accounts
0 0 * * * cd /home/user/decidim_application && RAILS_ENV=production bundle exec rake decidim:participants:delete_inactive_participants
```

All of these are currently in use in the system, except for the deletion of inactive participant accounts, which was deemed unnecessary since the platform is still in its early stages.

# Network and Firewall Environment

The network is configured in multiple layers. The domain and DNS configuration were provided by the university. The DNS is set to point to a Cloudflare-managed proxy, which sits between visitors and the server, acting as a CDN and providing DDoS/WAF protection.

There is also a reverse proxy layer configured with Nginx.

We have two server blocks. The first handles HTTP requests, redirecting them to HTTPS:

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
  # ...
```


The second block is where the actual traffic gets processed. The `ssl` parameter tells Nginx to perform the TLS handshake before processing requests on that port:

```
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    # ...
```

## Let's Encrypt certificate authentication

```
ssl_certificate     /etc/letsencrypt/live/decidim.lad.pucrs.br/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/decidim.lad.pucrs.br/privkey.pem;
```

## Restoring the original visitor IPs from Cloudflare

```
set_real_ip_from 192.0.2.1;  # full IP list...

real_ip_header CF-Connecting-IP;
```

By default, Nginx would log every request as coming from Cloudflare's IP addresses, since Cloudflare is the one connecting directly to the server. The `real_ip` module tells Nginx to trust requests coming from Cloudflare's known IP ranges and extract the real visitor IP from the `CF-Connecting-IP` header instead, so logs and application-level IP handling reflect the actual client.

## Reverse proxy path

```
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Ssl on;
        proxy_set_header X-Forwarded-Port 443;
    }
```

This block forwards all incoming requests to the Decidim application, which runs inside a Docker container and is exposed locally on port 3000. Since Nginx terminates TLS at this layer, the connection between Nginx and the application itself happens over plain HTTP internally this is known as TLS termination.

# CI/CD process
(section in progress)
