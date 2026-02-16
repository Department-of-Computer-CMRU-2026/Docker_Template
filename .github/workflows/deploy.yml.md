name: Smart-Class Auto Deploy (Self-hosted)

on:
  push:
    branches: [ "main" ]

jobs:
  deploy:
    runs-on: [self-hosted, web-deploy] 

    steps:
      - name: Fix Permissions and Backup .env
        run: |
           # Fix permissions first so we can read/write
           docker run --rm -v $(pwd):/work -w /work busybox chown -R $(id -u):$(id -g) . || true
           chmod -R 777 . || true
           
           # Backup .env if it exists
           if [ -f .env ]; then
             cp .env /tmp/.env.backup
             echo "Backed up .env to /tmp/.env.backup"
           fi

      - uses: actions/checkout@v4

      - name: Restore .env
        run: |
          if [ -f /tmp/.env.backup ]; then
            cp /tmp/.env.backup .env
            echo "Restored .env from backup"
            
            # Force update ports from repo's .env.docker to the restored .env
            # This ensures that if we change ports in git, they apply to the runner
            
            # 1. Get new ports from .env.docker
            NEW_PORT=$(grep '^STUDENT_PORT=' .env.docker | cut -d '=' -f2 | tr -d '\r ')
            NEW_DB_PORT=$(grep '^FORWARD_DB_PORT=' .env.docker | cut -d '=' -f2 | tr -d '\r ')
            
            # 2. Update .env if values exist
            if [ ! -z "$NEW_PORT" ]; then
               sed -i "s/^STUDENT_PORT=.*/STUDENT_PORT=$NEW_PORT/" .env
               echo "Updated STUDENT_PORT to $NEW_PORT in .env"
            fi
            
            if [ ! -z "$NEW_DB_PORT" ]; then
               # Check if key exists, if not append it, if yes update it
               if grep -q "^FORWARD_DB_PORT=" .env; then
                   sed -i "s/^FORWARD_DB_PORT=.*/FORWARD_DB_PORT=$NEW_DB_PORT/" .env
               else
                   echo "FORWARD_DB_PORT=$NEW_DB_PORT" >> .env
               fi
               echo "Updated FORWARD_DB_PORT to $NEW_DB_PORT in .env"
            fi
          fi

      - name: Docker Deploy
        run: |
          # 2. ตรวจสอบ/สร้างไฟล์ .env (ถ้า restore ไม่มา)
          if [ ! -f .env ]; then
            if [ -f .env.docker ]; then
              cp .env.docker .env
              echo "Created .env from .env.docker"
            elif [ -f .env.example ]; then
              cp .env.example .env
              echo "Created .env from .env.example"
            else
              echo "Warning: No .env setup file found!"
            fi
          fi

          # 3. กำหนด STUDENT_NAME
          # ลองอ่านจาก STUDENT_ID ใน .env ก่อน
          STUDENT_ID=$(grep '^STUDENT_ID=' .env | cut -d '=' -f2 | tr -d '\r ' | head -n 1)
          
          if [ ! -z "$STUDENT_ID" ]; then
             export STUDENT_NAME=$STUDENT_ID
             echo "Using STUDENT_NAME from .env: ${STUDENT_NAME}"
          else
             # ถ้าไม่มีให้ใช้ GitHub Actor (Lowercased)
             export STUDENT_NAME=$(echo "${{ github.actor }}" | tr '[:upper:]' '[:lower:]')
             echo "Using STUDENT_NAME from GitHub: ${STUDENT_NAME}"
          fi
          
          # 3. ดึงค่า STUDENT_PORT
          # ลองดึงจาก .env ก่อน
          if [ -f .env ]; then
            export STUDENT_PORT=$(grep '^STUDENT_PORT=' .env | cut -d '=' -f2 | tr -d '\r ' | head -n 1)
          fi

          # ถ้ายังไม่มีค่า ลองดึงจาก .env.docker
          if [ -z "$STUDENT_PORT" ] && [ -f .env.docker ]; then
            echo "STUDENT_PORT not found in .env, trying .env.docker..."
            export STUDENT_PORT=$(grep '^STUDENT_PORT=' .env.docker | cut -d '=' -f2 | tr -d '\r ' | head -n 1)
          fi

          # 4. ถ้ายังไม่มีค่าอีก ให้ใช้ Default 80
          if [ -z "$STUDENT_PORT" ]; then
            echo "Warning: STUDENT_PORT not found. Using default port 80."
            export STUDENT_PORT=80
          else
             echo "Using STUDENT_PORT=${STUDENT_PORT}"
          fi
          
          echo "Deploying for ${STUDENT_NAME} on port ${STUDENT_PORT}..."

          # 5. แก้ไข Permission ของโฟลเดอร์ Storage และ Cache บน Host
          chmod -R 777 storage bootstrap/cache public
          chmod 666 .env

          # 6. สั่งรัน Docker Compose (ส่งตัวแปรไปให้ .yml เห็น)
          STUDENT_NAME=${STUDENT_NAME} STUDENT_PORT=${STUDENT_PORT} docker compose up -d --build

          # 6. รอ DB (ใช้ Swap 8GB พยุงระบบ)
          echo "Waiting for database to be ready..."
          sleep 10

          # 7. Laravel Commands
          docker exec ${STUDENT_NAME}-smartclass-app php artisan key:generate --force
          docker exec ${STUDENT_NAME}-smartclass-app php artisan migrate --force
          docker exec ${STUDENT_NAME}-smartclass-app php artisan config:cache
          docker exec ${STUDENT_NAME}-smartclass-app php artisan route:cache
          docker exec ${STUDENT_NAME}-smartclass-app php artisan view:cache
          docker exec ${STUDENT_NAME}-smartclass-app php artisan event:cache
          docker exec ${STUDENT_NAME}-smartclass-app php artisan storage:link || true
          docker exec ${STUDENT_NAME}-smartclass-app php artisan fix:postgres-sequences || true

          # 8. รักษาพื้นที่ 28GB
          docker image prune -f
          docker container prune -f --filter "until=24h"