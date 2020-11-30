 docker ps --format '{{.ID}} {{.Command}} {{.Names}}' | grep pause

需要找容器的pause容器即可，代表了pod的网络接入情况