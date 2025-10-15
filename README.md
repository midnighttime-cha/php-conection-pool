# วิธีตั้งค่า PHP-FPM และ NGINX ให้รองรับ Concurrent

## ปรับ PHP-FPM ให้รองรับ Concurrent Connections มากขึ้น
- แก้ไขไฟล์ `www.conf`
```
sudo nano /etc/php/[PHP Version]/fpm/pool.d/www.conf
```

- ตั้งค่าตามต่อไปนี้

```
; ใช้ Dynamic process management
pm = dynamic

; จำนวน process ที่จะเปิดเริ่มต้น
pm.start_servers = 10

; จำนวน process ขั้นต่ำ (idle)
pm.min_spare_servers = 10

; จำนวน process สูงสุด (idle)
pm.max_spare_servers = 30

; จำนวน process สูงสุดทั้งหมด
pm.max_children = 250

; จำนวน request ต่อ worker ก่อน restart
pm.max_requests = 500

; เพิ่ม timeout ป้องกันการ hang
request_terminate_timeout = 60s
request_slowlog_timeout = 5s

; เพิ่ม limit
rlimit_files = 4096
rlimit_core = unlimited

; log file
slowlog = /var/log/[PHP Version]-fpm.slow.log
```

-- pm.max_children คือจำนวน concurrent PHP-FPM workers ถ้าคุณต้องการรองรับ 200 user พร้อมกัน แนะนำตั้งค่า: 200–250 worker (ขึ้นกับ CPU/RAM) ตรวจสอบหน่วยความจำว่า worker ละ ~40–60 MB

## ปรับค่าใน php.ini
- แก้ไขไฟล์ตามตัวอย่างต่อไปนี้
```
sudo nano /etc/php/[PHP Version]/fpm/php.ini
```

- แก้ไขตามนี้
```
max_execution_time = 60
memory_limit = 512M
post_max_size = 50M
upload_max_filesize = 50M
max_input_vars = 5000

; ปรับ persistent connection ให้ทำงานดีขึ้น
oci8.persistent_timeout = 120
oci8.ping_interval = 60
oci8.events = On
```

- oci8.persistent_timeout = ระยะเวลาที่ connection จะอยู่ใน pool ก่อน timeout
- oci8.ping_interval = ตรวจสอบ connection ที่ยัง active ทุกกี่วินาที

## ปรับ Nginx ให้สอดคล้องกับจำนวน PHP workers
- แก้ไขไฟล์ต่อไปนี้
```
sudo nano /etc/nginx/nginx.conf
```

- แก้ไขตามตัวอย่างนี้
```
worker_processes auto;

events {
    worker_connections 4096;
    multi_accept on;
}

http {
    keepalive_timeout 65;
    client_max_body_size 50M;

    upstream php-handler {
        server unix:/run/php/php5.6-fpm.sock;
        keepalive 256;
    }

    server {
        listen 80;
        server_name yourdomain.com;

        root /var/www/yourapp;
        index index.php;

        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass php-handler;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
    }
}
```

## ตรวจสอบ Performance
- รันคำสั่งดูสถานะ PHP-FPM:
```
sudo systemctl status php5.6-fpm
sudo service php5.6-fpm reload
```

- ตรวจสอบจำนวน process ที่เปิดอยู่:
```
ps -ef | grep php-fpm | wc -l
```

- ดู real-time usage:
```
top -c | grep php-fpm
```

## Monitoring (Optional)
### สร้าง status page:
- ใน /etc/php/[PHP Version]/fpm/pool.d/www.conf:
```
pm.status_path = /status
```

- ในไฟล์ nginx
```
location ~ ^/(status)$ {
    fastcgi_pass php-handler;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

- เปิดดูที่:
👉 http://yourdomain.com/status


