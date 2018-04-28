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
        stats socket /var/lib/haproxy/stats #thu thap thong tin ve haproxy 
            #########################################################################################################33333
            defaults #thiet lap tham so mac dinh cho tat ca thanh phan khac
                mode                    http
                log                     global
               #option                  httplog
                option                  dontlognull
                #option http-server-close
                #option forwardfor except 127.0.0.0/8
                option                  redispatch # giong health backend, neu 1BE bi loi thi request day den cai kia
                retries                 3 # sl ket noi lai khi khi 1 connect bi tu choi or timeout
                timeout http-request    10s # thời gian đợi cho 1 request mới 
                timeout queue           1m #khi sl connect max thi connect tiep theo se vao hang doi(queue), tham so xac dinh time hang doi connect, qua thi loai bo va bao loi 503: Server không thể trả lời vì quá bận hoặc đang được bảo trì.
                timeout connect         10s #thời gian đợi cho 1 kết nối thành công
                timeout client          1m # sl time tối đa không hoạt động bên phía client, không có request nó đóng két nối
                timeout server          1m # max inactivity time on the server side
                timeout http-keep-alive 10s # sau khi respone đươc gửi, nó là thời gian để đợi một http request đến 
                timeout check           10s # 
                maxconn                 5000 # so ket noi toi da dong thoi 1 core co the xy ly
            ######################################################################################################################3

    #HAProxy Monitoring Config
    listen monitoring *:8080             #Haproxy Monitoring port 8080
        mode http
        option forwardfor # cho phép sử dụng the X-Forwarded-For header,để người dùng chỉ nhìn thấy phản hồi từ HAproxy 
    option httpclose  # đong kết nối http khi không sử dụng vi default co 1 so ketnoi idle keepalive connect san, do ton resource
        stats enable                                 # enable status của haproxy
        #stats show-legends # reporting thong tin bo sung
        stats refresh 5s #
        stats uri /stats  #                           #URL for HAProxy monitoring
        #stats realm Haproxy\ Statistics #
        stats auth abc:abc #User and Password 
        stats admin if TRUE # cho phep su dung level admin de enable or disable servers tren web GUI
        #####################################################################################################################3
        frontend http_frontend
            bind *:80  # listen trên port 80, khac nhau khi ghi ro dia chi
            mode http
            option httplog                      #  cho phep log format nhieu thong tin hon default
            option dontlognull                  #  ket noi null connetc gui ve client khong ghi log cho phep ghi log null connect la connect khong co data 
            option http-server-close            # 
            option forwardfor                   # cho phep chen X-Forwarded-For header (IP goc)  den backend 
            reqadd X-Forwarded-Proto:\ http     #  Them mot header vao cuoi http request: o day string la: X-Forwarded-Proto:http
            acl url_private  path_beg -i /private/
            use_backend web45 if url_private
            default_backend web13
        ########################################################################################################################
        backend web13
            balance roundrobin
            server web1 10.140.0.2:80 weight 1 #check : neu check thi server nay nhu la master, request chi vao no,
            server web3 10.146.0.3:80 weight 2 #check backup: con cai nay nhu backup, khi nao master chet thi request day vao no

        backend web45
            balance roundrobin
            server web4 10.146.0.4:80 weight 1 #check #
            server web5 10.140.0.3:80 weight 2 #check backup 

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

### Cài đặt: Ép buộc kết nối http, thêm fontend_https,

     frontend http
         bind *:80  # listen trên port 80, khac nhau khi ghi ro dia chi
         redirect scheme https if !{ ssl_fc } # ep buoc ket noi http
         mode http
         option httplog                     
         option dontlognull                  
         option http-server-close             
         option forwardfor                   
         reqadd X-Forwarded-Proto:\ http

           frontend https
            bind *:443 ssl crt /etc/haproxy/certs/systemtip.tk.pem crt /etc/haproxy/certs/tipsystem.tk.pem  
            # muon bao nhieu doamin them bấy nhiêu
            mode http
            option httplog                      
            option dontlognull                  
            option http-server-close            
            option forwardfor                   
            reqadd X-Forwarded-Proto:\ http 
            acl url_private  path_beg -i /private/
            use_backend web45 if url_private
            default_backend web13

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
