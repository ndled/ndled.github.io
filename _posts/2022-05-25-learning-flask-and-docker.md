---
layout: post
title:  "Learning Flask and Docker for inspire.ndled.us"
categories: python
---

You can find the current iteration of the app at (inspire.ndled.us)[inspire.ndled.us] and code (https://github.com/ndled/youtube_screen_grab/)[here]


Three months ago my fiancee expressed an interest in learning some practical scripting in python. I had her brainstorm ideas for something simple she’d like to accomplish and she came up with the inspiration for this project. I will note that while yes, I did technically steal this idea from her, she had already stopped working on it and I thought it was cool so…

Her idea was simple - She loves animal crossing, and puts on island tour videos like (https://www.youtube.com/watch?v=1HPlHuz3k2I)[this one] while she putts around the house. She constantly gets design ideas from these videos and thought it would be cool to create a script that would grab a random image from the video so she could use it for inspiration.

While she did learn some scripting, she only got as far as downloading youtube videos with a script before going off on a tangent to extract audio only from her music playlists. Lame. I thought it was a great idea so I kept with it!

The original script looked a little something like this, minus the references to static/single because initially this was all a command line tool.


```
def download(playlist_url):
    ydl_opts = {
        'outtmpl': resolvepath(f"tmp/videos/{'%(title)s-%(id)s.%(ext)s'}"),
        'download_archive': resolvepath("tmp/download.txt"),
        'overwrites': False,
        'format':'135' # Fun fact, has to be string
    }
    with YoutubeDL(ydl_opts) as ydl:
        ydl.download([playlist_url])
 
def single_cut():
    for file in os.listdir(resolvepath("static/single")):
        video_path = resolvepath(f"static/single/{file}")
    image_name = os.path.basename(video_path)
    cap = cv2.VideoCapture(video_path)
    length = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    randomframe = random.randint(0,length)
    while(cap.isOpened()):
        frameId = int(cap.get(1))
        print(frameId)
        ret, frame = cap.read()
        if (ret != True):
            break
        if (frameId == randomframe):
            cv2.imwrite(resolvepath(f"static/single/{image_name}.jpg"), frame)
            image_path = f"static/single/{image_name}.jpg"
            print("wrote")
            cap.release()
            return image_path
    cap.release()
 

```
This worked fine, but I thought I could take this opportunity to learn flask and build a web app! The first iteration of the webapp even worked okay… As long as you didn’t refresh the page or have someone else attempt to process a video while your process was still on going. See, everything lived in the http request cycle which meant LONG load times for the result webpage.

 
```
def new():
    if request.method == 'POST':
        if request.form.get("submit"):
            form_data = request.form
            url = form_data['new_video']
            remove_old(dir = "static/new")
            down.single_download(url, path = "static/new")
            cap.new_cut(path = "static/new")
            image = random_image(img_dir = "./static/new")
            return render_template('new.html', image = image)
        elif request.form["random_image"]:
            image = random_image(img_dir = "./static/new")
            return render_template('new.html', image = image)
    else:
        return render_template('new.html')

```


Okay, we have a webapp that will do its thing, but I really need to offload that into something that lives outside of the actual serving of the webpage cycle. Enter Redis and Celery.

Redis does a lot of things, but really all I’m using it for is as a message broker. Celery is the workforce that runs the processing outside of the webserve cycle. Since I was already learning Flask and I needed to run Redis and Celery locally, I decided that I definitely wasn’t learning enough and needed to add learning Docker to my project. This has widely been considered a mistake and much suffering ensued.

But! I eventually got a system that would work for me. In dealing with all the fun added bits, the original project was shaved down from its scripted form. Minimum viable project and all that. Now I had a working prototype and multiple people could even use it at the same time! In my hubris I was not finished.

Okay so Digital Ocean fucking rules but I’m still going to complain about it anyways. I’m a pretty risk adverse person in both my personal and professional life. There is no fucking way to limit my financial risk with any cloud provider I know of. I just want a fucking toggle that will shut off AND KILL (I don’t care) my personal projects if I go over a billing threshhold. That threshold can be high! I’m cool with a max bill of like, five hundred dollars which would make me feel like a dumbass if I ever hit, but I don’t care! I just need some ability to limit my ability to fuck myself over. Anyways, Digital Ocean, like every other cloud provider I know of, doesn't let you do that. Their documentation and tutorials are bomb though, so I can’t be too mad. It was surprisingly easy to integrate with Digital Ocean and I had a fully deployed flask app with about an hour’s worth of work. I even brought in ANOTHER Docker container for nginx to handle serving static content. The beautiful http site was up and running and by god it worked. I felt powerful.

Getting an ssl cert for https was a little more painful. If you’re unaware, The Electronic Frontier Foundation tries to make this easy for dumb people like me. They provide a neat package called certbot that is designed to make it easy as two enters, an email address, and a lonely y to grant a cert and configure nginx with it… Bummer I had Dockerized nginx on alpine linux, so certbot needed to be a little more manual. After spinning up the containers, I ran certbot with

```
 certbot certonly --webroot --webroot-path /var/www/certbot/
```

Next I needed to let nginx know about the cert, so I moved a new config file over

```
docker cp nginx/nginx.conf youtube_screen_grab_nginx_1:/etc/nginx/conf.d
```
With the config file looking like:

```
upstream youtube_screen_grab {
    server web:5000;
}
 
 
server {
    listen 80;
    listen [::]:80;
 
    server_name inspire.ndled.us www.inspire.ndled.us;
    server_tokens off;
 
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
 
    location / {
        return 301 https://inspire.ndled.us$request_uri;
    }
}
 
server {
    listen 443 default_server ssl http2;
    listen [::]:443 ssl http2;
 
    server_name inspire.ndled.us;
 
    ssl_certificate /etc/nginx/ssl/live/inspire.ndled.us/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/inspire.ndled.us/privkey.pem;
   
    location / {
        proxy_pass http://youtube_screen_grab;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }
}

```

And it’s up and running! This was a really neat process that walked me through my first time deploying something I wanted up in the cloud. Digital Ocean makes it very easy to get started and I’m thrilled with all the Flask and Docker I’ve learned while going through this.

This work would not be possible without the following blog posts and tutorials:

(https://flask.palletsprojects.com/en/2.1.x/)[Official Flask Tutorial]
(https://testdriven.io/courses/flask-celery/docker/)[Dockerizing Celery and Flask]
(https://learndocker.online/courses/fundamentals/)[Best Docker Tutorial I’ve found]

Until next time,

Noah
