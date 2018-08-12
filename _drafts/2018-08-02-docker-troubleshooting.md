---
layout: post
title: "Docker Troubleshooting"
date: 2018-08-02
categories: programming devops
---
Getting started with Docker and .NET Core was a little rough. Here are some of the snags I hit, and how I untangled them.
1. I deployed my container! ... And it's broken? (Ensure your server is listening on port 8080 and try again.)
    It runs fine locally using dotnet run, but something stupid happens in production. Ideally, you have Docker installed on your dev machine, and this issue never comes up. However, I do my dev work in a VM, and Docker is super flaky when nested virualization comes into the picture. So I'm basically doing this blind and hoping the deployment comes out right. This is less than ideal, but it's forced me to get more comfortable working in a console.
        How to inspect Docker build artifacts: docker run -it --rm image_name sh
            The -it flag starts the container in "interactive" mode, i.e. when you run the command, you end up in a bash shell
            The --rm flag will destroy the container when you exit
        So now we can verify that everything was built and copied over the way we anticipated. In my case, it was..
    I specified the wrong port (8080) when I started the container. My app was listening on port 80 (the default value).
        So, instead of docker run -d -p 8080:8080 gcr.io/${PROJECT_ID}/whatever:v1, the command should be docker run -d -p 8080:80 ...
    /facepalm
        Alternatively, you could set the ASPNETCORE_URLS environment variable in your dockerfile: ENV ASPNETCORE_URLS http://0.0.0.0:8080
2. I can hit my API! ... But nothing comes back from my endpoint.
    My next roadblock was that the ValuesController that comes out of the box worked fine (it just returns a simple array of strings), but my own controller which returns data from a SQLite database failed to return any data.
    kubectl get pods
    kubectl logs pod-name
        In the logs, there was a stack trace from .NET indicating it wasn't able to hit the database. Once again, the issue was irritatingly simple. The connection string I was using was: $@"Data Source={AppContext.BaseDirectory}_SqliteDatabase\rd-database.db"
        It should have been $@"Data Source={AppContext.BaseDirectory}_SqliteDatabase/rd-database.db"
        ... yeah, with a forward slash. That was it.
