
ZeroTier เป็นซอฟต์แวร์ที่ใช้สร้างเครือข่ายเสมือน (Virtual Network) โดยเชื่อมต่ออุปกรณ์ต่างๆ เข้าด้วยกันเสมือนอยู่ในเครือข่ายเดียวกันไม่ว่าจะอยู่ที่ไหนก็ตาม ส่วน Exit Node คือเครื่องที่ทำหน้าที่เป็นทางออกสู่อินเทอร์เน็ตสาธารณะสำหรับอุปกรณ์อื่นๆ ในเครือข่าย ZeroTier ที่ตั้งค่าให้ใช้ Exit Node นั้น

### การตั้งค่า Exit Node บน Proxmox

#### 1. บน Proxmox (เครื่อง Exit Node)

**การติดตั้ง ZeroTier และเข้าร่วมเครือข่าย**

1.  เปิด Terminal บน Proxmox
    
2.  ติดตั้ง ZeroTier โดยใช้คำสั่ง:

    
    ```
    curl -s https://install.zerotier.com | sudo bash
    
    ```
    
3.  หลังจากติดตั้งเสร็จ ให้ใช้คำสั่งต่อไปนี้เพื่อเข้าร่วมเครือข่าย ZeroTier ของคุณ โดยแทนที่ `<nwid>` ด้วย Network ID ที่ได้จาก ZeroTier Central
    

    
    ```
    sudo zerotier-cli join <nwid>
    
    ```
    
4.  ตรวจสอบสถานะการเข้าร่วมเครือข่าย โดยใช้คำสั่ง:
    

    
    ```
    zerotier-cli info
    
    ```
    
    หรือ
    

    
    ```
    zerotier-cli listnetworks
    
    ```
    
    คุณควรจะเห็น Network ID ที่คุณเข้าร่วมปรากฏขึ้นมาพร้อมกับสถานะที่บอกว่า **OK**
    

**การเปิดใช้งาน IPv4 forwarding**

1.  แก้ไขไฟล์การตั้งค่าระบบโดยใช้โปรแกรมแก้ไขข้อความ เช่น `nano`:
    

    ```
    sudo nano /etc/sysctl.conf
    
    ```
    
2.  ค้นหาบรรทัดที่มีข้อความ `#net.ipv4.ip_forward=1` และลบเครื่องหมาย `#` ออก หรือเพิ่มบรรทัด `net.ipv4.ip_forward=1` ลงไปในไฟล์
    
3.  บันทึกและปิดไฟล์
    
4.  โหลดการตั้งค่าใหม่เพื่อให้มีผลทันทีโดยใช้คำสั่ง:
    

    
    ```
    sudo sysctl -p
    
    ```
    
5.  ตรวจสอบว่า IPv4 forwarding ทำงานแล้ว โดยใช้คำสั่ง:
    

    
    ```
    sudo sysctl net.ipv4.ip_forward
    
    ```
    
    ผลลัพธ์ที่ได้ควรเป็น `net.ipv4.ip_forward = 1`
    

**การตั้งค่า NAT และการส่งต่อข้อมูล (Forwarding)**

1.  กำหนดชื่อ Interface ของ ZeroTier และ Interface ที่ใช้เชื่อมต่ออินเทอร์เน็ต (WAN)
    

    
    ```
    export ZT_IF=ztXXXXXX   # เปลี่ยน XXXXXX เป็นชื่อ Interface ของ ZeroTier ของคุณ
    export WAN_IF=eth0         # เปลี่ยน eth0 เป็นชื่อ Interface ที่ออกสู่อินเทอร์เน็ต
    
    ```
    
    คุณสามารถหาชื่อ ZeroTier Interface ได้จากคำสั่ง `ip a` หรือ `ifconfig`
    
2.  ใช้คำสั่ง `iptables` เพื่อกำหนดกฎการส่งต่อข้อมูล:
    

    ```
    sudo iptables -t nat -A POSTROUTING -o $WAN_IF -j MASQUERADE
    sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    sudo iptables -A FORWARD -i $ZT_IF -o $WAN_IF -j ACCEPT
    
    ```
    
3.  ติดตั้ง `iptables-persistent` เพื่อบันทึกกฎ `iptables` ให้คงอยู่หลังการรีบูตเครื่อง:
    

    
    ```
    sudo apt-get install iptables-persistent
    
    ```
    
4.  บันทึกกฎที่ตั้งค่าไว้:
    

    
    ```
    sudo netfilter-persistent save
    
    ```
    

----------

#### 2. บน ZeroTier Central

1.  เข้าสู่ ZeroTier Central ที่ my.zerotier.com
    
2.  ไปที่หน้าเครือข่ายของคุณ
    
3.  เลื่อนไปที่ส่วน **Members** แล้วหา Proxmox Node ของคุณ
    
4.  คลิกที่กล่องเพื่อเปิดใช้งาน **Allow Default Route / Exit Node**
    
5.  ไปที่ส่วน **Managed Routes** และเพิ่มเส้นทางใหม่:
    
    -   **Destination:** `0.0.0.0/0`
        
    -   **Via:** `[IP address ของ Proxmox บนเครือข่าย ZeroTier]`
        
    -   ตัวอย่าง: `0.0.0.0/0` via `192.168.192.123`
        

----------

#### 3. บนมือถือหรือ iPad

1.  เปิดแอป ZeroTier
    
2.  แตะที่เครือข่ายของคุณ
    
3.  เปิดใช้งาน **Use Exit Node** และเลือก Proxmox Node ที่คุณตั้งค่าไว้
    
4.  รอสักครู่เพื่อให้การตั้งค่าซิงค์
    

----------

#### 4. การทดสอบ

1.  เปิดเว็บเบราว์เซอร์และเข้าเว็บไซต์ตรวจสอบ IP เช่น [ifconfig.me](https://ifconfig.me/)
    
2.  ตรวจสอบว่า IP ที่แสดงเป็น IP สาธารณะของ Proxmox (IP WAN) ของคุณ
    
3.  ลองใช้คำสั่ง `ping 8.8.8.8` เพื่อตรวจสอบว่าสามารถเชื่อมต่ออินเทอร์เน็ตได้
