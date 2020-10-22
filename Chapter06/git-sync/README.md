Forked from [kubernetes contrib]

## build local docker image
```bash
bash> docker build -t local/git-sync .
```

## create docker-compose.yml
```yml
version: '2'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - html:/usr/share/nginx/html:z
    depends_on:
      - git-sync
    restart: always
  git-sync:
    image: local/git-sync
    environment:
      GIT_SYNC_REPO: "https://github.com/psyoblade/kubia-website-example.git"
      GIT_SYNC_DEST: "/git"
      GIT_SYNC_BRANCH: "master"
      GIT_SYNC_REV: "FETCH_HEAD"
      GIT_SYNC_WAIT: "10"
    volumes:
      - html:/git:z
    restart: always
volumes:
  html:
    driver: local
```

## start docker-compose
```bash
docker-compose up -d
```

## modify index.html
* let's modify [index.html](https://github.com/psyoblade/kubia-website-example/blob/master/index.html)
