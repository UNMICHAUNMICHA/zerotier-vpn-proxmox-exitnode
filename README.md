
**✅ สรุปวิธีทำ ZeroTier → LAN Bridge แบบถาวร
🧱 สิ่งที่ต้องมี
เครื่อง Linux (เช่น Ubuntu/Debian)**




 
มี LAN interface (เช่น ens18)
เชื่อม ZeroTier แล้ว ได้ IP เช่น 192.168.192.x
ZeroTier Network ต้องอนุญาตเครื่อง (authorized)
Network LAN จริง: เช่น 192.168.0.0/24

🪜 ขั้นตอน
## host
nano /etc/pve/lxc/100.conf
unprivileged: 1
lxc.apparmor.profile: unconfined
lxc.cgroup.devices.allow: a
lxc.cap.drop:
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file


## 1. Join ZeroTier และตรวจสอบ IP

sudo zerotier-cli join <network-id>
sudo zerotier-cli listnetworks
ตรวจสอบว่า status เป็น OK

มี interface เช่น ztxxxxxxx ได้ IP เช่น 192.168.192.109

## 2. เปิด IP Forwarding

echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

## 3. ตั้งค่า NAT (ZeroTier → LAN)

sudo iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
🟡 สำคัญ: เปลี่ยน ens18 ให้ตรงกับชื่อ LAN interface ของคุณ (ดูจาก ip a)

## 4. ติดตั้ง iptables-persistent เพื่อให้ NAT ติดถาวร

sudo apt install iptables-persistent
sudo netfilter-persistent save
~~

## 5. ตั้ง static route บน ZeroTier Central

~~
ไปที่ https://my.zerotier.com → Network → Advanced → Routes:


192.168.0.0/24 via 192.168.192.109
192.168.0.0/24 คือ LAN จริง
192.168.192.109 คือ IP ของเครื่อง bridge ใน ZeroTier

✅ เสร็จแล้ว! ทดสอบ
จากเครื่องใน ZeroTier network:


ping 192.168.0.1          # ping router
ping 192.168.0.184        # ping เครื่องใน LAN จริง
🔁 ทดสอบหลังรีสตาร์ต
รีบูต:


sudo reboot
หลังบูตให้เช็ค:


zerotier-cli listnetworks      # Status = OK
ip a                           # มี IP ztxxxxxxx
iptables -t nat -L -n -v       # มี MASQUERADE


## 💬 หมายเหตุ ใช้งานได้กับ ZeroTier ทุก OS: เครื่องปลายทางไม่ต้องรู้เรื่อง routing เลย แบบนี้ ง่ายและปลอดภัยกว่า การ bridge Layer 2 (ไม่ต้องยุ่งกับ DHCP หรือ bridge จริง) ถ้าจะให้ reverse (LAN → ZeroTier) ก็สามารถทำได้เพิ่ม route/iptables ฝั่ง router หากต้องการให้ผมส่งเป็น Shell Script .sh หรือ PDF เอกสารคู่มือ บอกได้เลยครับ ✨
