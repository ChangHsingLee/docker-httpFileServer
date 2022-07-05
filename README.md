# HTTP File Server (go-http-file-server)
Please refer to <https://gitee.com/mjpclab/go-http-file-server><br>
This installation guide is for version 1.13.7

## Installation
1. Check out source code
	```bash
	git clone https://gitee.com/mjpclab/go-http-file-server.git && \
	cd go-http-file-server && \
	git switch -c v1.13.7-go1.9to1.15
	```
2. Build application in docker contianer(Golang compilation environment)
	2.1 Assume you have installed docker engine, type below command to build application
	```bash
	./build/build-all-by-docker.sh
	```
3. Modify the dockerfile(build/build-docker-image-dockerfile) which used to create docker image of go-http-file-server
	3.1 Due to permission issue, I add new account 'dms'(Administrator, UID/GID is 1000/1000) and change default user from 'nobody' to 'dms'<br>
	3.2 For flexibility and easy to configure file server, I add parameter '--config' to startup command
	3.3 I have docker image of alpine v3.15 in locally, so I use it for file server.
	`WAS:`
	```dockerfile
	FROM alpine
	COPY --from=builder /output /
	RUN mkdir /lib64 /var/ghfs; ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2
	VOLUME /var/ghfs
	EXPOSE 8080 8443
	USER nobody
	CMD [ \
			"/usr/local/bin/ghfs", \
			"--listen-plain", "8080", "-r", "/var/ghfs/", \
			",,", \
			"--listen-tls", "8443", "-c", "/etc/server.crt", "-k", "/etc/server.key", "-r", "/var/ghfs/" \
	]
	```
	`IS:`
	```dockerfile
	FROM alpine:3.15
	COPY --from=builder /output /
	RUN mkdir /lib64 /var/ghfs; ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2 && \
	    adduser -D -u 1000 dms
	VOLUME /var/ghfs
	EXPOSE 8080 8443
	USER dms
	CMD [ \
			"/usr/local/bin/ghfs", \
			"--config", "/etc/ghfs.conf", \
			"--listen-plain", "8080", "-r", "/var/ghfs/", \
			",,", \
			"--listen-tls", "8443", "-c", "/etc/server.crt", "-k", "/etc/server.key", "-r", "/var/ghfs/" \
	]
	```
4. Create docker image
	4.1 Modify script 'build/build-docker-image.sh', due to I don't want to use two tags for same docker image.
	`WAS:`
	```bash
	docker build -t "$TAG_PREFIX:latest" -t "$TAG_PREFIX:$VERSION" -f ./build-docker-image-dockerfile ../
	```
	`IS:`
	```bash
	docker build -t "$TAG_PREFIX:$VERSION" -f ./build-docker-image-dockerfile ../
	```
	4.2 add lable 'stage' in dockerfile(build/build-docker-image-dockerfile) that can help to identify intermediate image.
	`WAS:`
	```dockerfile
	FROM golang AS builder
	ENV GO111MODULE=auto
	COPY .git/ /mnt/ghfs/.git/
	COPY src/ /mnt/ghfs/src/
	COPY build/ /mnt/ghfs/build/
	RUN ["/bin/bash", "-c", "cd /mnt/ghfs/build/; source ./build.inc.sh; go build -ldflags \"$LDFLAGS\" -o /tmp/ghfs /mnt/ghfs/src/main.go"]
	RUN mkdir -p /output/usr/local/bin/; cp /tmp/ghfs /output/usr/local/bin/;
	COPY conf/docker-image/ /output/
	```
	`IS:`
	```dockerfile
	FROM golang AS builder
	LABEL stage=builderIntermediateImg
	ENV GO111MODULE=auto
	COPY .git/ /mnt/ghfs/.git/
	COPY src/ /mnt/ghfs/src/
	COPY build/ /mnt/ghfs/build/
	RUN ["/bin/bash", "-c", "cd /mnt/ghfs/build/; source ./build.inc.sh; go build -ldflags \"$LDFLAGS\" -o /tmp/ghfs /mnt/ghfs/src/main.go"]
	RUN mkdir -p /output/usr/local/bin/; cp /tmp/ghfs /output/usr/local/bin/;
	COPY conf/docker-image/ /output/
	```
	4.3 create docker image
	```bash
	./build/build-docker-image.sh
	```
5. Start container
	5.1 To decide location of your configuration file, certificate files and root directory of file server
	|In container|In host side|Description|
	|-|-|-|
	|/var/ghfs|/home/dockerContainer/httpFileServer/storage|root directory of file server|
	|/var/ghfs/upload|/home/dockerContainer/httpFileServer/storage/upload|allow user to create folder and upload file under this directory|
	|/etc/ghfs.conf|/home/dockerContainer/httpFileServer/config/fileServer.conf|configuration file|
	|/etc/server.crt|/home/dockerContainer/httpFileServer/config/server.crt|certificate file for HTTPS protocol|
	|/etc/server.key|/home/dockerContainer/httpFileServer/config/server.key|certificate file for HTTPS protocol|
	5.2 To generate certificate file
	Refer to <https://docs.microsoft.com/zh-tw/azure/application-gateway/self-signed-certificates#create-a-root-ca-certificate>
	```bash
	openssl ecparam -genkey -name prime256v1 -out ca.key && \
	openssl req -new -sha256 \
		-subj "/C=TW/ST=HsinChu/O=Mitrastar/OU=DMS/CN=localhost" \
		-key ca.key -out ca.csr && \
	openssl x509 -req -sha256 -days 365 -signkey ca.key -in ca.csr -out ca.crt
	
	openssl ecparam -genkey -name prime256v1 -out server.key && \
	openssl req -new -sha256 -key server.key -out server.csr \
		-subj "/C=TW/ST=HsinChu/O=Mitrastar/OU=DMS/CN=172.23.75.32" && \
	openssl x509 -req -sha256 -days 365 -CA ca.crt -CAkey ca.key \
		-CAcreateserial -in server.csr -out server.crt

	cp server.crt server.key /home/dockerContainer/httpFileServer/config/
	```
	5.3 Create configuration file
	```bash
	SERVER_CONFIG_DIR=/home/dockerContainer/httpFileServer/config; \
	SERVER_CONFIG_DIR=.; \
	CONFIG_FILE=fileServer.conf; \
	echo -e "--root /var/ghfs\n--default-sort /n\n--upload-dir /var/ghfs/upload/\n--mkdir-dir /var/ghfs/upload/" > $SERVER_CONFIG_DIR/$CONFIG_FILE
	```
	5.4 Start container
	`HTTPS enabled:`
	```bash
	FILE_SERVER_ROOT=/home/dockerContainer/httpFileServer; \
	docker run -itd --name httpFileServer --restart always -p 80:8080 -p 443:8443 \
	-v $FILE_SERVER_ROOT/storage:/var/ghfs \
	-v $FILE_SERVER_ROOT/config/fileServer.conf:/etc/ghfs.conf \
	-v $FILE_SERVER_ROOT/config/server.crt:/etc/server.crt \
	-v $FILE_SERVER_ROOT/config/server.key:/etc/server.key \
	-v /etc/localtime:/etc/localtime:ro \
	-v /etc/timezone:/etc/timezone:ro \
	mjpclab/ghfs:1.13.7
	```
	`HTTPS disabled:`
	```bash
	FILE_SERVER_ROOT=/home/dockerContainer/httpFileServer; \
	docker run -itd --name httpFileServer --restart always -p 80:8080 \
	-v $FILE_SERVER_ROOT/storage:/var/ghfs \
	-v $FILE_SERVER_ROOT/config/fileServer.conf:/etc/ghfs.conf \
	-v /etc/localtime:/etc/localtime:ro \
	-v /etc/timezone:/etc/timezone:ro \
	mjpclab/ghfs:1.13.7
	```
6. To remove intermediate files
	I don't need the docker image of golang, so remove it.
	```bash
	docker image prune --filter label=stage=builderIntermediateImg && \
	docker rmi golang:latest
	```
