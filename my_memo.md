
```bash
go build -o minio .
docker build -t minio-baseurl:20251211-3 .
```

Dockerfile
```
FROM minio/minio:latest
COPY ./minio /usr/bin/minio
```

docker-compose.yaml
```
version: '3.8'

services:
  minio:
    image: minio-baseurl:20251211-3
    command: server --api-base-path /minio-api --console-base-path /minio --address ":9000" --console-address ":9001" /data
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio123
    ports:
      - "19000:9000"  # S3 API
      - "19001:9001"  # Web Console
    volumes:
      - baseurl_data:/data

volumes:
  baseurl_data:
```


nginx
```
        # minio
        # /minio-api → /minio-api/ に揃える
        location = /minio-api {
                return 302 /minio-api/;
        }

        location /minio-api/ {
                # /minio-api/ を剥がして MinIO API に流す
                proxy_pass http://127.0.0.1:19000/minio-api/;

                proxy_set_header Host              $host;
                proxy_set_header X-Real-IP         $remote_addr;
                proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }
        # /minio → /minio/ に揃える
        location = /minio {
                return 302 /minio/;
        }
        location /minio/ {
                # 重要: /minio/* を /* にリライト
                rewrite ^/minio/(.*) /$1 break;

                proxy_pass http://127.0.0.1:19001;

                proxy_set_header Host              $host;
                proxy_set_header X-Real-IP         $remote_addr;
                proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
                # WebSocket対応
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
        }
    ```