# Home Web Server
#### Why?
AWS is expensive. RPis are cheap. Linux is free. You probably already pay for the internet.

### Server setup
If you are planning on using a RPi, do yourself a flavor and go ahead and download this their imager. https://www.raspberrypi.com/software/

I choose the most recent Ubuntu server image and etched it onto a spare M.2 SSD I had laying around. You don't need to use a SD card anymore. Any bootable image via USB is good enough. I used the M.2 because it is way faster than any SD card I had laying around.

#### Considerations
I'm just running a rails app and a reverse proxy on this thing and it's using 475Mb/4GB of RAM. So trying this with a pi zero is a waste of time IMHO.

##### Stuff they don't tell you:
When you first start up the ubuntu server it will ask for a username and password.

`username => ubuntu, password => ubuntu`

### Server setup continued
From there you need to determine what you're doing with it and the various packages you'll need.

Here's what I installed to run Ruby 3.2.2 & Rails 7. YMMV

```
sudo apt update
sudo apt upgrade
sudo apt install build-essential libssl-dev libreadline-dev zlib1g-dev libsqlite3-dev libyaml-dev nginx certbot python3-certbot-nginx
```

Install rbenv 
```
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec $SHELL
```

Install ruby-build plugin
```
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
exec $SHELL
```

Install Ruby
```
rbenv install <desired Ruby version>
rbenv global <desired Ruby version>
```

Install more packages FTW
```
sudo apt install ruby-dev ruby-devel rubygems
```
Now you can probably successfully run `sudo gem install rails` YMMV (It's a roll your own linux server after all)

### Reverse proxy setup
First things first, you're going to need to configure your http and https ports in your router's port forwarding menu. You probably already know how to do this but if you don't, there should be a million different videos online on how to do that.
For me I just added two seperate entries. One for http and one for https. You need to know servers local ip address and port ranges (port 80 for http and 443 for https). It doesn't really matter what port you choose as long as it matches with your nginx config and is unused by other processes on the server.

To let certbot do the heavy lifting for you, you first need a basic reverse proxy setup. Probably easiest to get http working then tackle https

This should get ya going, don't sweat the details in this as all of it will change once we tackle https. Annoyingly certbot needs a working nginx proxy to kick off it's what-have-yous.
```
sudo vim /etc/nginx/sites-available/default
```
```
server {
    listen 80;
    server_name your_domain.com;

    location / {
        proxy_pass http://192.168.0.<servers_local_ip>:3000;  # Change the port number to whatever you start up the rails app with.
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Double check there are no syntax errors
```
sudo nginx -t
```

Restart nginx
```
sudo systemctl restart nginx
```

If that checks out now is the time to go into your DNS records for whatever domain you want to use (if you want to use a domain name, you may be cool with just referencing the public ip address)

Add an A name record. **NOTE** if you are using godaddy delete the parked A name record they default ya with.
![Screenshot 2023-06-03 at 12 38 00 PM](https://github.com/TheDread-Pirate-Roberts/random_stuff_I_will_forget/assets/60859971/951878e5-0f07-4e45-8a65-db2f059e922d)

You can find your public ip [at this website](https://www.whatismyip.com/)

They say DNS record changes can take 24-48 hours but with godaddy it was immediate. YMMV

#### Now we are at point the we can use certbot to automagically manage our Let's Encrypt certs.
Follow the certbot prompts
```
sudo certbot --nginx
```
Once that is done, check syntax and restart
```
sudo nginx -t
sudo systemctl restart nginx
```
#### YAY. You've got a reverse proxy setup for http and https with a valid cert that gets automatically updated via certbot 🤯
It's likely this will get ya all the GET requests a human can dream of, but if you want full CRUD operations there is some built in security measures in Rails you need to address at the app level and nginx level.

In your Rails app
`sudo vim config/environments/production.rb`
Uncomment the following
```
  # config.force_ssl = true
```
Add the following
```
config.action_controller.default_url_options = { protocol: 'https', host: '<Your DNS name that also matches your nginx settings>' }
```
In the reverse proxy you need to add some headers (note the proxy_set_header lines in the location block)
```
server {
    server_name <Your domanin name>;

    location / {
        proxy_pass http://<server's local ip>:3000;
	      proxy_set_header  Host $host;
  	    proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
  	    proxy_set_header  X-Forwarded-Proto $scheme;
  	    proxy_set_header  X-Forwarded-Ssl on; # Optional
  	    proxy_set_header  X-Forwarded-Port $server_port;
  	    proxy_set_header  X-Forwarded-Host $host;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/<Your domanin name>/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/<Your domanin name>/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}

server {
    if ($host = <Your domanin name>) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    server_name <Your domanin name>;
    return 404; # managed by Certbot

}
```
Check syntax and restart nginx
```
sudo nginx -t
sudo systemctl restart nginx
```
Start up that beautiful Rails app on your server!
```
RAILS_ENV=production bundle exec rails server -b 0.0.0.0 -p 3000
```
**This is where you can visit it via the web, and check to make sure everything is working and you have a valid cert.**

#### Random and probably useful stuff
Check memory
```
free -h
```
Check storage
```
df -h
```
Remote reboot
```
sudo reboot
```
Run the rails app in the background
```
screen
RAILS_ENV=production bundle exec rails server -b 0.0.0.0 -p 3000
<ctrl a + d>
```
Reattach to the backgrounded rails app
```
screen -r
```

#### CI/CD
TBD
