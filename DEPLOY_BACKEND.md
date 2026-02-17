# TÀI LIỆU HƯỚNG DẪN TRIỂN KHAI HỆ THỐNG BACKEND (AWS EC2, NGINX & CLOUDFLARE)

![Status](https://img.shields.io/badge/Status-Learning-blue) ![Topic](https://img.shields.io/badge/Topic-Deploy%20Backend-orange)

Triển khai hệ thống backend cho người mới bắt đầu.
Những kiến thức cơ bản giúp bạn có thể triển khai một hệ thống backend lên server một cách hiệu quả.

> **Lưu ý:** Đây là tài liệu ghi chép, tổng hợp và tóm tắt lại kiến thức cá nhân của tôi trong quá trình học.

---

## Mục lục

- [Lời mở đầu](#lời-mở-đầu)
- [Giới thiệu](#giới-thiệu)
- [Chương 1: Tổng quan và Kiến trúc hệ thống](#chương-1-tổng-quan-và-kiến-trúc-hệ-thống)
  - [Mục tiêu tài liệu](#mục-tiêu-tài-liệu)
  - [Kiến trúc triển khai (Deployment Architecture)](#kiến-trúc-triển-khai-deployment-architecture)
  - [Yêu cầu tiên quyết (Prerequisites)](#yêu-cầu-tiên-quyết-prerequisites)
- [Chương 2: Đóng gói ứng dụng (Containerization)](#chương-2-đóng-gói-ứng-dụng-containerization)
  - [Chuẩn bị ứng dụng để đóng gói](#chuẩn-bị-ứng-dụng-để-đóng-gói)
  - [Xây dựng Dockerfile](#xây-dựng-dockerfile)
  - [Quản lý đa dịch vụ với Docker Compose](#quản-lý-đa-dịch-vụ-với-docker-compose)
  - [Lưu trữ Image trên Container Registry](#lưu-trữ-image-trên-container-registry)
- [Chương 3: Khởi tạo và Cấu hình Môi trường Đám mây (AWS EC2)](#chương-3-khởi-tạo-và-cấu-hình-môi-trường-đám-mây-aws-ec2)
  - [Khởi tạo máy chủ ảo (EC2 Instance)](#khởi-tạo-máy-chủ-ảo-ec2-instance)
  - [Cấu hình bảo mật mạng (Security Groups)](#cấu-hình-bảo-mật-mạng-security-groups)
  - [Cài đặt môi trường Runtime trên Linux](#cài-đặt-môi-trường-runtime-trên-linux)
- [Chương 4: Triển khai và Vận hành Ứng dụng](#chương-4-triển-khai-và-vận-hành-ứng-dụng)
  - [Khởi chạy hệ thống Container](#khởi-chạy-hệ-thống-container)
  - [Quản lý và Debug (Xử lý sự cố)](#quản-lý-và-debug-xử-lý-sự-cố)
- [Chương 5: Thiết lập Web Server và Reverse Proxy (Nginx)](#chương-5-thiết-lập-web-server-và-reverse-proxy-nginx)
  - [Cài đặt Nginx](#cài-đặt-nginx)
  - [Cấu hình Reverse Proxy](#cấu-hình-reverse-proxy)
- [Chương 6: Tên miền và Chống DDoS (Cloudflare)](#chương-6-tên-miền-và-chống-ddos-cloudflare)
  - [Thiết lập DNS Cơ bản](#thiết-lập-dns-cơ-bản)
  - [Kích hoạt lớp bảo vệ Cloudflare](#kích-hoạt-lớp-bảo-vệ-cloudflare)
- [Chương 7: Bảo mật SSL/TLS (Giao thức HTTPS)](#chương-7-bảo-mật-ssltls-giao-thức-https)
  - [Khởi tạo chứng chỉ số](#khởi-tạo-chứng-chỉ-số)
  - [Cấu hình chứng chỉ lên Server](#cấu-hình-chứng-chỉ-lên-server)
- [Chương 8: Kiểm thử và Vận hành (Testing & Operation)](#chương-8-kiểm-thử-và-vận-hành-testing--operation)
  - [Kiểm thử toàn diện](#kiểm-thử-toàn-diện)
  - [Bảo trì và Cập nhật (Hướng phát triển)](#bảo-trì-và-cập-nhật-hướng-phát-triển)

---

## Lời mở đầu
*(Ghi chú ngắn về lý do viết tài liệu, mong muốn chia sẻ kiến thức...)*

## Giới thiệu
*(Giới thiệu ngắn gọn về công nghệ sử dụng: Docker, AWS EC2, Nginx, Cloudflare...)*

---

## Chương 1: Tổng quan và Kiến trúc hệ thống

### Mục tiêu tài liệu
*(Trình bày các bước đưa API từ localhost ra internet, có tên miền chuyên nghiệp)*

### Kiến trúc triển khai (Deployment Architecture)

*(Giải thích luồng đi của request từ Client -> Cloudflare -> EC2 -> Nginx -> Docker Container)*

### Yêu cầu tiên quyết (Prerequisites)
*(Các tài khoản cần có, ví dụ: domain, VPS/EC2, mã nguồn backend đã chạy ổn định)*

---

## Chương 2: Đóng gói ứng dụng (Containerization)

### Chuẩn bị ứng dụng để đóng gói
*(Ví dụ: Cấu hình biến môi trường, build ra file thực thi như `.jar` đối với Spring Boot)*

### Xây dựng Dockerfile
*(Các bước viết file `Dockerfile` để tạo Image)*

### Quản lý đa dịch vụ với Docker Compose
*(Cấu hình `docker-compose.yml` để chạy nhiều service cùng lúc như Backend và Database)*

### Lưu trữ Image trên Container Registry
*(Lệnh `docker push` lên Docker Hub)*

---

## Chương 3: Khởi tạo và Cấu hình Môi trường Đám mây (AWS EC2)

### Khởi tạo máy chủ ảo (EC2 Instance)
*(Hướng dẫn chọn OS Ubuntu, tạo Key Pair `.pem`)*

### Cấu hình bảo mật mạng (Security Groups)
*(Mở port 22, 80, 443)*

### Cài đặt môi trường Runtime trên Linux
*(Các lệnh cài đặt Docker, Docker Compose trên Ubuntu)*

---

## Chương 4: Triển khai và Vận hành Ứng dụng

### Khởi chạy hệ thống Container
*(Sử dụng lệnh `docker-compose up -d`)*

### Quản lý và Debug (Xử lý sự cố)
*(Cách xem log bằng `docker logs` và truy cập container bằng `docker exec`)*

---

## Chương 5: Thiết lập Web Server và Reverse Proxy (Nginx)

### Cài đặt Nginx
*(Lệnh `sudo apt install nginx`)*

### Cấu hình Reverse Proxy
*(Điều hướng traffic từ port 80/443 vào port của Container đang chạy ứng dụng)*

---

## Chương 6: Tên miền và Chống DDoS (Cloudflare)

### Thiết lập DNS Cơ bản
*(Trỏ bản ghi `A record` về IP của EC2)*

### Kích hoạt lớp bảo vệ Cloudflare
*(Bật đám mây màu cam)*

---

## Chương 7: Bảo mật SSL/TLS (Giao thức HTTPS)

### Khởi tạo chứng chỉ số
*(Cách tạo Origin Certificate trên Cloudflare)*

### Cấu hình chứng chỉ lên Server
*(Cấu hình Nginx đọc file `.pem` và `.key`, thiết lập ép buộc HTTPS)*

---

## Chương 8: Kiểm thử và Vận hành (Testing & Operation)

### Kiểm thử toàn diện
*(Test gọi API qua Postman, test các luồng nhận Webhook thanh toán (ví dụ: Tingee) qua mạng public)*

### Bảo trì và Cập nhật (Hướng phát triển)
*(Cách update phiên bản mới, định hướng dùng GitHub Actions)*