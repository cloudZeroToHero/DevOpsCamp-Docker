# LAB 02


## Create a Python app (without using Docker)

First step is to create a Python file with simple app. How complicated it can be ? :wink:

File looks like this:
``` py
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "hello world!"

if __name__ == "__main__":
    app.run(host="0.0.0.0")
```



BTW - source files are available here:
> https://github.com/cloudZeroToHero/DevOpsCamp-Docker/tree/main/Code/Lab02

## Dockerfile

A dockerfile lists the instructions necessary to build a Docker image. As a base layer I will use python:3.8.5-alpine image. According to Lab manual "The alpine version means that it uses the alpine distribution, which is significantly smaller than an alternative flavor of Linux. A smaller image means it will download (deploy) much faster, and it is also more secure because it has a smaller attack surface."

On top of that is **RUN** command that executes commands - in this case install package
Next is the **CMD** command that describes what will run once container starts.
Last but not least - **COPY** command which in this case copies file from the same directory where dockerfile is into a new layer.

All and all - file looks like this:

``` dockerfile
FROM python:3.8.5-alpine
RUN pip install flask
CMD ["python","app.py"]
COPY app.py /app.py
```

## Build the Docker image

After all those preparations is time to crate an image.
```
docker image build -t python-hello-world .
```
(don't forget . a the end fo command above)

Few moments later I can enjoy new image

![new Docker image](./images/L02-003-new-docker-image.jpg)

## Run the Docker image

Let's run the container
```
docker run -p 5001:5000 -d --name pythonapp python-hello-world 
```

![start the container](./images/L02-004-run-container.jpg)

This is how it looks on Docker desktop

![running container with app](./images/L02-005-running-container.jpg)

And if I open web browser to see what is on localhost port 5001 I can see that app is running

![running app](./images/L02-006-app-is-runnig.jpg)

## Output log

With 
```
docker container logs <container_id>
```
I can check what was sent to standard output by the application:

![app output](./images/L02-007-app-output.jpg)


## Push image to Docker hub

First step - crate a account on [Docker hub](https://hub.docker.com/). Fortunately there is possibility to crate a free account :smile:


To login to Docker hub use 
```
docker login
```
![docker hub login](./images/L02-008-docker-hub-login.jpg)

Now the fun part - I have to add tag to the image to "tag it" as the one which should be stored in Docker Hub.

The command from Lab manual is 
```
docker tag python-hello-world [dockerhub username]/python-hello-world
```
Works fine (now I have two images with the same ID but with different tags), but it also shows one flaw - all the images are tagged as "latest". So - a bit too late I decided to add another tag - this time with version tag.
```
docker tag python-hello-world cloudzerotohero/python-hello-world:1.0
```

![image tag](./images/L02-010-tag-version.jpg)

Command to push the image to Docker Hub:
```
docker push cloudzerotohero/python-hello-world:1.0
```

What a success - image is now on the hub with proper version tag
![on the Docker Hub](./images/L02-012-on-the-hub.jpg)

## Deploy a change

Time to see how to deploy changes. 
First step - change app source file. Now it looks like this:

``` py
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello Beautiful World!"

if __name__ == "__main__":
    app.run(host="0.0.0.0")
```


Rebuild the app
```
docker image build -t cloudzerotohero/python-hello-world:1.0 .
```
Push it to Docker Hub
```
docker push cloudzerotohero/python-hello-world:1.0
```

And brand new image is uploaded to Docker hub :smile:

## Image layers

It is possible to check what layers constitute the image. 
To list it simply run
```
 docker image history python-hello-world
```

As it was expected layers listed in command output correspond to what can be found on official Python3 image plus what was the content of the Dockerfile

![image layers](./images/L02-013-layers.jpg)


## Cleanup

Now it is time to cleanup environment
Check if eny container is running with 
```
docker container ls
```
If so - stop it with 
```
docker container stop <container-id>
```
And remove containers with 
```
docker system prune
```


