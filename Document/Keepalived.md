## Keepalive là gì
* Keepalive là một kỹ thuật được dùng để gửi message từ một thiết bị đến các thiết bị khác để kiểm tra liên kết giữa chúng có còn hoạt động nhằm đảm bảo các thiết bị vẫn hoạt động . 
* Keepvalive được viết bằng ngôn ngữ C.
* Khi người dùng cần truy cập vào dịch vụ, người dùng chỉ cần truy cập vào địa chỉ IP ảo dùng chung này thay vì phải truy cập vào những địa chỉ IP thật của các thiết bị kia.

## Mô hình: 
      
 ![](/image/keepalived.jpg)

* Thêm giải quyết vấn đề băng thông.

 ![](/image/extra.jpg)
      
## Một số đặc điểm của  Keepalived:
* Keepalived nó chỉ đảm bảo rằng sẽ luôn có ít nhất một máy chủ chịu trách nhiệm cho IP dùng chung khi có sự cố xảy ra.
* Cho phép nhiều hosts cùng chia sẻ một địa chỉ IP ảo với nhau . Keepalived thường được dùng để dựng các hệ thống HA (High Availability) dùng nhiều server để đảm bảo hệ thống được hoạt động liên tục.
* Keepalived dùng giao thức VRRP (Virtual Router Redundancy Protocol) để liên lạc giữa các thiết bị trong nhóm.

## Cơ chế hoạt động 
* Khi dịch vụ keepalive chạy trên các server, một địa chỉ Virtual IP sẽ được tạo ra. Các server sẽ có 2 trạng thái là MASTER/ACTIVE và BACKUP/SLAVE. Cơ chế failover được xử lý bởi giao thức VRRP, khi khởi động dịch vụ, toàn bộ các server dùng chung VIP sẽ gia nhập vào một nhóm multicast. Nhóm multicast này dùng để gởi/nhận các gói tin quảng bá VRRP. Các server sẽ quảng bá độ ưu tiên (priority) của mình, server với độ ưu tiên cao nhất sẽ được chọn làm MASTER. Một khi nhóm đã có 1 MASTER thì MASTER này sẽ chịu trách nhiệm gởi các gói tin quảng bá VRRP định kỳ cho nhóm multicast. Và khi request truy cập vào V.ip thì MASTER sẽ chịu trách nhiêm xử lý các request này
Nếu vì một sự cố mà các server BACKUP không nhận được các gói tin quảng bá từ MASTER trong một khoảng thời gian nhất định thì cả nhóm sẽ bầu ra một MASTER mới. MASTER mới này sẽ tiếp quản địa chỉ VIP của nhóm và gởi các gói tin ARP báo là nó đang giữ địa chỉ VIP này. Khi MASTER cũ hoạt động bình thường trở lại thì server này có thể lại trở thành MASTER hoặc trở thành BACKUP tùy theo cấu hình độ ưu tiên của các server

## Cấu hình
### Master:
      
      ### global_defs: cấu hình thông tin toàn cục (global) cho keepalived như gởi email thông báo tới đâu, tên của cluster đang cấu hình.

    

          global_defs {

             notification_email {

               abcdefdg@gmail.com

             }

             notification_email_from xyzyzxyz@gmail.com

             smtp_server smpt.gmail.com

             smtp_connect_timeout 30

             router_id 192.168.10.11  #IP or hostname serrver

          }

    

    ### vrrp_script: chứa script, lệnh thực thi hoặc đường dẫn tới script kiểm tra dịch vụ (Ví dụ: nếu dịch vụ này down thì keepalived sẽ tự chuyển VIP sang 1 server khác).



    vrrp_script chk_haproxy {

          script "pidof haproxy"

          interval 2

          weight 2

    }



    vrrp_script chk_nginx {

          script "pidof nginx"

          interval 2

          weight 2 

    }



    vrrp_instance VI_WAN {

          interface ens160

          state MASTER

          virtual_router_id 51

          priority 100            # 100 on master, 99 on backup

          smpt_alert # theo hd them doan nay vaof

          virtual_ipaddress {

               192.168.10.100

     }



         track_script {

               chk_nginx

         }

    }

      ### vrrp_instance: thông tin chi tiết về 1 server vật lý trong nhóm dùng chung VRRP. Gồm các thông tin như interface dùng để liên lạc của server này, độ ưu tiên để, virtual IP tương ứng với interface, cách thức chứng thực, script kiểm tra dịch vụ….

### Backup

       global_defs {
           notification_email {
             abcabcabc@gmail.com
           }
           notification_email_from xyxyxyxyxy@gmail.com
           smtp_server smpt.gmail.com
           smtp_connect_timeout 30
           router_id 192.168.10.11  #IP or hostname serrver
        }

        vrrp_script chk_haproxy {
              script "pidof haproxy"
              interval 2
              weight 2
      }

        vrrp_script chk_nginx {
              script "pidof nginx"
              interval 2
              weight 2 
        }

        vrrp_instance VI_WAN {
              interface ens160
              state BACKUP
              virtual_router_id 51
              priority 99       	# 100 on master, 99 on backup
              smpt_alert # theo hd them doan nay vaof
              virtual_ipaddress {
                   103.53.170.205
              }

       track_script {
             chk_nginx
       }
      }

        vrrp_instance VI_LAN {
            state MASTER
            interface ens192
            virtual_router_id 110
            priority 99		# 101 on master, 100 on backup
            virtual_ipaddress {
                192.168.43.205
            }
            track_script {
                   chk_haproxy
            }
        }



### Enable iptables rule vrrp trên cả Master và Slave

           iptables -I INPUT -p vrrp -j ACCEPT
           
 ## Ý nghĩa file cấu hình:
 
+ state (MASTER|BACKUP): chỉ trạng thái MASTER hoặc BACKUP được sử dụng bởi máy
chủ. Nếu là MASTER thì máy chủ này có nhiệm vụ nhận và xử lý các gói tin từ host đi lên.
Nếu con MASTER tèo, những con BACKUP này sẽ dựa vào 1 cơ chế bầu chọn và nhảy lên
làm Master.
+ interface: chỉ định cổng mạng nào sẽ sử dụng cho hoạt động IP Failover – VRRP
+ mcast_src_ip: địa chỉ IP thực của card mạng Interface của máy chủ tham gia vào VRRP.
Các gói tin trao đổi giữa các VRRP Router sử dụng địa chỉ thực này.
+ virtual_router_id: định danh cho các router (ở đây là máy chủ dịch vụ) thuộc cùng 1 nhóm
VRRP. Hiểu nôm na là 1 router có thể tham gia nhiều nhóm VRRP (các nhóm hoạt động
động lập nhau), và VRRP-ID là tên gọi của từng nhóm.
+ priority: chỉ định độ ưu tiên của VRRP router (tức độ ưu tiên máy chủ dịch vụ trong quá
trình bầu chọn MASTER). Các VRRP Router trong cùng một VRRP Group tiến hành bầu
chọn Master sử dụng giá trị priority đã cấu hình cho máy chủ đó. Priority có giá trị từ 0 đến 255. Nguyên tắc có bản: Priority cao nhất thì nó là Master, nếu priority bằng nhau thì IP cao hơn là Master.
+ interval 2: Sẽ check dịch vụ nginx 2s/lấn
+ weight 2:  Tham số được thêm với priority để bầu ra Master
+ advert_int: thời gian giữa các lần gởi gói tin VRRP advertisement (đơn vị giây).
+ smtp_alert: kích hoạt thông báo bằng email SMTP khi trạng thái MASTER có sự thay đổi.
+ authentication: chỉ định hình thức chứng thực trong VRRP. ‘auth_type‘, sử dụng hình thức
mật khẩu plaintext hay mã hoá AH. ‘auth_pass‘, chuỗi mật khẩu chỉ chấp nhận 8 kí tự.
virtual_ipaddress: Địa chỉ IP ảo của nhóm VRRP đó (Chính là địa chỉ dùng làm gateway
cho các host). Các gói tin trao đổi, làm việc với host đều sử dụng địa chỉ ảo này.
+ notify_master: chỉ định chạy shell script nếu có sự kiện thay đổi về trạng thái MASTER.
+ notify_backup: chỉ định chạy shell script nếu có sự kiện thay đổi về trạng thái BACKUP.
+ notify_fault: chỉ định chạy shell script nếu có sự kiện thay đổi về trạng thái thất bại (fault)

* Giải thích: Theo file cấu hình: 
 File cấu hình của cả 2 server sẽ tạo ra một Virtual IP. Server 1 sẽ ở trạng thái MASTER ( giữ VIP và toàn bộ request từ người dùng tới địa chỉ VIP sẽ được Server1 xử lý và phản hồi lại người dùng ), Server2 sẽ ở trạng thái BACKUP. Trong file cấu hình Keepalived trên server1, server2 đều có track_script ( sẽ check process ID (PID) của chương trình nginx nếu dịch vụ nginx trên Server1  bị chết Keepalived sẽ trừ trọng số (priority 100 - 2 =98) trên server1. Trọng số này nhỏ hơn trọng số trên file cấu hình Keepalived của Server 2(99). Do đó Keepalived sẽ chuyển trạng thái của server 1 từ Master sang Backup và trạng thái của server2 từ Backup sang Master ( giữ VIP). Lúc này toàn bộ yêu cầu của người dùng tới địa chỉ VIP sẽ được server2 xử lý và phản hồi tới người dùng. Keepalived sẽ tiếp tục check 2s 1 lần. Nếu dịch vụ nginx trên server1 khởi động lại, priorty của server1 sẽ là 100 lớn hơn priority của server1 priroty 99. Keepalived sẽ chuyển trạng thái của server1 sang Master và server1 sang Backup
 
 ## Check log:
 * Check ip: ip a
      
          eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc pfifo_fast state UP qlen 1000
          link/ether 42:01:0a:92:00:04 brd ff:ff:ff:ff:ff:ff
          inet 1192.168.10.199./32 brd 10.146.0.4 scope global dynamic eth0
          valid_lft 82348sec preferred_lft 82348sec
          inet  192.168.10.100/32 scope global eth0 # Virtual IP
          
 * Check log : tailf /var/log/message
  Master
            Feb 19 15:35:49 localhost Keepalived_vrrp[15089]: VRRP_Instance(VI_1) Transition to MASTER STATE
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: VRRP_Instance(VI_1) Entering MASTER STATE
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: VRRP_Instance(VI_1) setting protocol VIPs.
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: Sending gratuitous ARP on eth0 for 103.53.170.205
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on eth0 for 
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: Sending gratuitous ARP on eth0 for 103.53.170.205
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: Sending gratuitous ARP on eth0 for
            
  Backup: 
            
            Feb 19 15:35:49 localhost Keepalived_vrrp[15089]: VRRP_Instance(VI_1) Transition to BACKUP STATE
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: VRRP_Instance(VI_1) Entering BACKUP STATE
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: VRRP_Instance(VI_1) removing protocol VIPs.
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: Sending gratuitous ARP on eth0 for 103.53.170.205
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on eth0 for 
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: Sending gratuitous ARP on eth0 for 103.53.170.205
            Feb 19 15:35:50 localhost Keepalived_vrrp[15089]: Sending gratuitous ARP on eth0 for
            
Khi Master fail service nginx -> priority =100

            Feb 19 22:59:48 web1 Keepalived_vrrp[26772]: VRRP_Script(chk_nginx) failed
            Feb 19 22:59:48 web1 Keepalived_vrrp[26772]: VRRP_Instance(VI_1) Changing effective priority from 102 to 100
            Feb 19 22:59:49 web1 Keepalived_vrrp[26772]: VRRP_Instance(VI_1) Received advert with higher priority 101, ours 100
            Feb 19 22:59:49 web1 Keepalived_vrrp[26772]: VRRP_Instance(VI_1) IPSEC-AH : Syncing seq_num - Decrement seq
            Feb 19 22:59:49 web1 Keepalived_vrrp[26772]: VRRP_Instance(VI_1) Entering BACKUP STATE
            Feb 19 22:59:49 web1 Keepalived_vrrp[26772]: VRRP_Instance(VI_1) removing protocol VIPs.
Khi đó server Backup làm Master và Loadbanlancing vẫn tiếp được được đảm nhận bởi server Backup
