
## 1️⃣ Cập nhật thông tin RDS MySQL

Giả sử RDS MySQL của bạn có thông tin:

* Endpoint: `mydb.xxxxx.ap-southeast-1.rds.amazonaws.com`
* Port: `3306`
* Database: `wordpressdb`
* User: `wp_user`
* Password: `admin@123`

---

## 2️⃣ Docker Compose mới (chỉ còn WordPress)

```yaml
services:
  wordpress:
    image: wordpress:php8.2-fpm-alpine
    container_name: wordpress
    environment:
      WORDPRESS_DB_HOST: mydb.xxxxx.ap-southeast-1.rds.amazonaws.com:3306
      WORDPRESS_DB_NAME: wordpressdb
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: admin@123
    volumes:
      - ./wordpress_data:/var/www/html
    restart: always
```

👉 **Không cần network custom** nếu chỉ có 1 container.

---

## 3️⃣ Những thứ BẮT BUỘC phải kiểm tra trên RDS ⚠️

### ✅ 1. Security Group

* Cho phép **IP của máy chạy Docker** truy cập port `3306`
* Nếu Docker chạy trên EC2 → cho phép SG của EC2

### ✅ 2. Database đã tồn tại

* `wordpressdb` phải được tạo sẵn
* User `wp_user` có quyền:

```sql
GRANT ALL PRIVILEGES ON wordpressdb.* TO 'wp_user'@'%';
FLUSH PRIVILEGES;
```

### ✅ 3. Charset (khuyến nghị)

```sql
ALTER DATABASE wordpressdb
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

```bash
docker compose up -d
```