## Giới thiệu: 

*HAproxy (Hight Availability Proxy): là giải pháp proxy cân bằng tải mã nguồn mở phổ biến hiện nay, có thể chạy trên Linux, Solaris, và FreeBSD. 

*Nó dùng một số thuật toán để cân bằng tải lưu lượng cho các server backend đằng sau nó như web, application, database.

Mô hình: Vị trí Haproxy LB trong mô hình 

![](/image/haproxy.jpg)

Một thuật toán hay dùng trong HA proxy:

* Round Robin:  Mối kết nối mới sẽ được đẩy cho backend tiếp theo xử lý. Nếu máy chủ backend cuối cùng trong danh sách được đạt tới, nó sẽ bắt đầu lại từ đầu danh sách backend. Ngoài ra có thể cấu hinh track session trên thuật toán này để  Haproxy  biết dược  sẽ đẩy rrqest nào server hợp lý.  VÍ dụ:

        
        backend web13
        balance roundrobin
        cookie SERVERID insert indirect nocache # Haproxy gắn môt set-cookie là SERVERID="name backend" vào bản tin respone về Client. Ví dụ SERVERID=web1
        server web1 10.140.0.2:80 check cookie web1 # Check set-cookie, nếu SERVERID=web1 thì request đẩy về web1
        server web3 10.146.0.3:80 check cookie web2 # Check set-cookie, nếu SERVERID=web3 thì request đẩy về web3


* Least Connections: Các kết nối mới sẽ được xử lý bởi các máy chủ backend với số lượng kết nối ít nhất. Điều này rất hữu ích khi thời gian và các request rất lớn.

* Source: Các IP của client sẽ được băm để xác định máy chủ backend đã nhận được request từ IP này. Như vậy một sẽ luôn được xử lý bởi duy nhất một backend. Nhưng Source có nhược điểm là: nếu  khi đang sử dụng  thuật  toán  source  client chỉ được một server backend xử lý request, nếu   server đó bị lỗi thì  mọi request xau từ client đều được đẩy đến backend bị ỗi dó gây nên lỗi 500  nternal server error 500 do không có backend nào xử lý request  .cho cho dù có cấu hình "option redispatch" (tùy chọn cho phép đẩy request sang BE khác khi request không được BE ban đầu .Do đó  có thể cấu hình track session trong khi sử dụng Round Robin như trên

-----------------------------------------------------------------------------------------------------------------------------
## Cấu hình: 
* Các thành phần trong cấu hình HAproxy: global, default, listen, fontend, backend
global: Khu vực setup các thông tin chugn cho service haproxy như log, user, group...
default: Setup các thông tin mặc định cho áp dụng cho các section listen, fontend, backend.
fontend: Định nghĩa cách thức các request sẽ được chuyển hướng đến backend.
backend: Định nghĩa thuật toán cách thức forward request đến backend và các backend nhận và xử lý request.
listen:  có thể định nghĩa một proxy đầy đủ với cả thành phần của fontend và backend. Thường là chỉ cho lưu lượng TCP. mode: tcp

Ví dụ: 

    global
        log         127.0.0.1 local2     #
        chroot      /var/lib/haproxy     #
        pidfile     /var/run/haproxy.pid 
        maxconn     4000
        user        haproxy
        group       haproxy
        daemon
        stats socket /var/lib/haproxy/stats 
    #########################################################################################################33333
    defaults 
        mode                    http
        log                     global
        option                  dontlognull
        #option forwardfor except 127.0.0.0/8
        option                  redispatch 
        retries                 3 
        timeout http-request    10s 
        timeout queue           1m 
        timeout connect         10s
        timeout client          1m 
        timeout server          1m 
        timeout http-keep-alive 10s 
        timeout check           10s 
    ######################################################################################################################3

    #HAProxy Monitoring Config # monitor hệt thống haproxy và các backend sau mó 
    listen monitoring *:8080             
        mode http
        #option forwardfor 
        option httpclose  
        stats enable                          
        stats refresh 5s 
        stats uri /stats  
        stats auth abc:abc
    #####################################################################################################################3
    frontend http
        bind *:80  
        redirect scheme https if !{ ssl_fc } 

    ########################################################################################################################
    frontend https
    bind *:443 ssl crt /etc/haproxy/certs/systemtip.tk.pem crt /etc/haproxy/certs/tipsystem.tk.pem  
    option httplog                     
    option dontlognull                  
    option http-server-close            
    #option forwardfor                   
    #reqadd X-Forwarded-Proto:\ http     
    acl url_private  path_beg -i /private/
    use_backend web45 if url_private
    default_backend web13
    ###########################################################################################################################
    backend web13
        balance roundrobin
        cookie SERVERID insert indirect nocache # Haproxy gắn môt set-cookie là SERVERID="name backend" vào bản tin respone về Client. Ví dụ SERVERID=web1
        server web1 10.140.0.2:80 check cookie web1 # Check set-cookie, nếu SERVERID=web1 thì request đẩy về web1
        server web3 10.146.0.3:80 check cookie web2 # Check set-cookie, nếu SERVERID=web3 thì request đẩy về web3
        
    backend web45
        balance roundrobin
        server web4 10.146.0.4:80 weight 2
        server web5 10.140.0.3:80 weight 3

Ở trên có cấu hình https với hai domain:

Với kiểu HAProxy with SSL Termination: Cài đặt SSL trên Haproxy để mã hóa dữ liệu giữa client và HA, còn kết nối từ Haproxy đến BE là http.

Ban đầu khi tạo ra chứng chỉ SSL được 2 file: fullchain.pem and privkey.pem, nhưng haproxy không hỗ trợ cài đặt ssl qua 2 file này nên cần gộp nó lại thành 1 file với lệnh dươ để được 1 file  với tên  systemtip.tk.pem và  tipsystem.kem. 
          
          mkdir -p /etc/haproxy/certs
          DOMAIN='example.com' sudo -E bash -c 'cat /etc/letsencrypt/live/$DOMAIN/fullchain.pem                                          
          /etc/letsencrypt/live/$DOMAIN/privkey.pem > /etc/haproxy/certs/$DOMAIN.pem'v 
 
 
### Test:

Trong file cấu hình có định nghĩa backend web13 và web 45 xử lý các request đến 2 doamin: systemtip.tk và tipsystem.tk. Theo đó mặc định có request đến sẽ dùng backend web13. Với cấu hình:

    acl url_private  path_beg -i /private/
    use_backend web45 if url_private
Nếu request có thêm đường dẫn /private thì sẽ forward backend web45.

### Kết quả: test với domain: systemtip.tk 

Ở đây web 3 được lựa chọn để xử lý  các request  từ máy  này

![](/image/a.png)


* Khi dùng đường dẫn /private

Web 4 xử lý 2 request
![](/image/c.png)

Web 5 xư lý 3 request
![](/image/d.png)

### Monitor Haproxy 

File cấu hình đã cài đặt để monitor hệ thống haproxy theo địa chỉ  http://35.200.91.217:8080/stats

![](/image/e.png)
