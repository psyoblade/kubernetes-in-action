FROM golang:1.6-onbuild
VOLUME ["/git"]
ENV GIT_SYNC_DEST /git
ENTRYPOINT ["/go/bin/git-sync"]
