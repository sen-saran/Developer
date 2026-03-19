แก้ไข Memory limit: 512 MB ยังไง

docker exec nextcloud_app php -r "echo ini_get('memory_limit');"

nano ~/nextcloud/docker-compose.yml

เพิ่มใน app service:

app:

    image: nextcloud:33

    container_name: nextcloud_app

    restart: always

    environment:

      - PHP_MEMORY_LIMIT=1024M

      - PHP_UPLOAD_LIMIT=10G



docker compose down

docker compose up -d

# ตรวจสอบ

docker exec nextcloud_app php -r "echo ini_get('memory_limit');"

```

ผลที่ควรได้:

```

1024M
