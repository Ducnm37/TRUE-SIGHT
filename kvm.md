# Giới thiệu về ảo hóa và giải pháp KVM

## Ảo hóa (virtualization)

**Ảo hóa** là công nghệ tạo ra môi trường phần cứng ảo bằng cách sao chép chức năng vật lý của phần cứng thật và chúng đưa vào một hệ điều hành. Hệ thống vật lý để chạy các phần mềm ảo hóa gọi là `host`, các máy ảo cài trên đó gọi là `guest`

- **Các công nghệ ảo hóa phổ biến dành cho linux**: KVM, Xen, QEMU, VirtualBox, Openvz

- **Các loại ảo hóa**:
<ul>
  <ul>
    <li>Host based:Hypervisor giao tiếp với phần cứng thông qua 1 hệ điều hành và các phương thức quản lý, cấp phát tài nguyên đều phải thông qua hệ điều hành. Loại ảo hóa này bao gồm các giải pháp như Vmware WorkStation, Oracle VirtualBox, Microsoft Virtual PC, …</li>
    <img src="https://i.imgur.com/oZieKBN.jpg">
    <li>Hypervisor based:Hypervisor tương tác trực tiếp với phần cứng của máy chủ để quản lý, phân phối và cấp phát tài nguyên. Loại ảo hóa này bao gồm các giải pháp như Vmware ESXi, Microsoft Hyper-V, Xen Server, KVM.</li>
    <img src="https://i.imgur.com/BlAk6eQ.jpg">
     </ul>
     </ul>
       
-**Thách thức với x86 platform**: Kiến trúc x86 định nghĩa ra 4 mức hoạt động, gọi là ring, từ ring 0, 1, 2 đến 3. Mỗi ring hiểu là một privileged level. OS luôn hoạt động ở ring 0 còn user level application hoạt động ở ring 3 nhưng trong kiến trúc ảo hóa, virtualization layer lại hoạt đông ở ring 0. Điều đáng ngại là một số chỉ thị của guest OS ( gọi là non-virtualizable instruction ) nếu không được thực hiện trong ring 0 thì sẽ không hoạt động được. Thách thức ở đây là bắt được các chỉ thị này và translate các chỉ thị này xuống hardware ngay trong khi hệ thống hoạt động. Để vượt qua thách thức này, vmware có đề xuất ba giải pháp ảo hóa.

<img src="https://i.imgur.com/nPLhwiH.png">

<ul>
<ul>
<li>Full virtualization: Các non virtualizable instruction từ guest OS được translate bằng binary translation ở virtualization layer xuyên qua kernel ở ring 0 tới phần cứng. Còn user level application thì thực hiện làm việc trực tiếp (direct execution) xuyên qua virtualization layer. Bằng cách này, trở ngại các chỉ thị guest OS không hoạt động ở ring khác 0 bị vượt qua còn các user level application vẫn họat động ở native speed (tốc độ đáp ứng yêu cầu giống như khi không có ảo hóa). Guest OS hoàn toàn không nhận ra nó đang nằm trên một lớp ảo hóa vì các low-level request không có gì thay đổi. Do đó guestOS hoàn toàn không phải chỉnh sửa gì.</li>
 <img src="https://i.imgur.com/NPSQwgZ.png">
  
<li>Paravirtualization:Hypervisor sẽ cung cấp hypercall interface, guest os sẽ được Chỉnh sửa  kernel code để trở thành kernel máy chủ do đó các ứng dụng có thể làm việc trực tiếp với kernel máy chủ</li>
 <img src="https://i.imgur.com/dFijNWX.png">
 <img src="https://i.imgur.com/qQRG7Wh.png">
 
<li>Hardware Assisted Virtualization:Các giải pháp hỗ trợ ảo hóa của hardware vendor được công bố vào năm 2006 như VT-x của Intel hay AMD-v của AMD. Cả hai giải pháp này đều hướng đến việc xây dựng một CPU mode mới dành riêng cho virtualization layer gọi là root mode ( CPU mode -1). Bằng cách này, các OS request từ guest OS sẽ được tự động đi xuyên qua virtualization layer và cũng không cần kỹ thuật binary translation nữa do guest OS đã nằm ở ring 0. Trạng thái của guest OS sẽ được lưu trong Virtual machine control structure (VT-x) hoặc Virtual machine control block (AMD-v)</li>
 <img src="https://i.imgur.com/b39rrli.png">
         </ul>
       </ul>
       
  
  <img src="https://i.imgur.com/5KndopB.png">
  
  
---------
## KVM (Kernel-based Virtual Machine) 
### 1. Khái niệm
**KVM** Là giải pháp ảo hóa cho hệ thống linux trên nền tảng phần cứng x86 có các module mở rộng hỗ trợ ảo hóa (Intel VT-x hoặc AMD-V). Về bản chất, KVM không thực sự là một hypervisor có chức năng giải lập phần cứng để chạy các máy ảo. Chính xác KVM chỉ là một module của kernel linux hỗ trợ cơ chế mapping các chỉ dẫn trên CPU ảo (của guest VM) sang chỉ dẫn trên CPU vật lý (của máy chủ chứa VM). Hoặc có thể hình dung KVM giống như một driver cho hypervisor để sử dụng được tính năng ảo hóa của các vi xử lý như Intel VT-x hay AMD-V, mục tiêu là tăng hiệu suất cho guest VM.

|                       virtualization                      |                       Đặc điểm                      
|-----------------------------------------------------------|--------------------------------------------------------
|OpenVZ                                                     |OpenVZ chỉ chạy trên hệ thống Linux và sử dụng kernel duy nhất và các máy ảo OpenVZ sử dụng kernel chung nhau với máy chủ. Và  gây ra việc cấp phát bộ nhớ hay tài nguyên không được tách biệt rất dễ overload (quá tải) do các máy ảo khác dùng chung tài nguyên với nhau các máy ảo khác yêu cầu.
|XEN                                                        |Công nghệ ảo hóa XEN có thể cài được cả Linux hay Windows Operating system cho phép mỗi máy ảo chạy nhân riêng của nó, do đó mỗi máy ảo có hệ thống File System riêng và hoạt động như 1 máy chủ độc lập. Tài nguyên cung cấp cho máy ảo XEN cũng độc lập, nghĩa là mỗi máy ảo XEN được cấp 1 lượng RAM, CPU và Disk riêng
|KVM                                                        |Hoạt động tương tự như XEN. Máy chủ chỉ chạy trên linux nhưng máy ảo có thể chạy cả linux và window, hiệu năng cao, chi phí thấp, dễ quản lý

---

### 2. **Kiến trúc KVM**

<img src="https://i.imgur.com/NqIunCP.png">

**Kiến trúc KVM gồm 3 thành phần chính:**

- KVM kernel module: Là một phần trong dòng chính của Linux kernel
<ul>
  <ul>
    <li>Cung cấp giao diện chung cho Intel VMX và AMD SVM (thành phần hỗ trợ ảo hóa phần cứng)</li>
<li>Chứa những mô phỏng cho các instructions và CPU modes không được support bởi Intel VMX và AMD SVM</li>
  </ul>
  </ul>
  
- quemu - kvm: là chương trình dòng lệnh để tạo các máy ảo, thường được vận chuyển dưới dạng các package "kvm" hoặc "qemu-kvm". Có 3 chức năng chính:
<ul>
  <ul>
    <li>Thiết lập VM và các thiết bị ra vào (input/output)</li>
    <li>Thực thi mã khách thông qua KVM kernel module</li>
    <li>Mô phỏng các thiết bị ra vào (I/O) và di chuyển các guest từ host này sang host khác</li>
  </ul>
  </ul>
  
- libvirt management stack:
<ul>
  <ul>
    <li>cung cấp API để các tool như virsh có thể giao tiếp và quản lí các VM</li>
    <li>cung cấp chế độ quản lí từ xa an toàn</li>
  </ul>
</ul>

<img src="https://i.imgur.com/Dhdf5cv.png">

---

### 3. **Mô hình triển khai KVM**

- Hình dưới đâu mô tả mô hình thực hiện của KVM. Đây là một vòng lặp của các hành đông diễn ra để vận hành các máy ảo. Những hành động này được phân cách bằng 3 phương thức chúng ta đã đề cập trước đó: user-mode, kernel-mode, guest-mode.

<img src="https://i.imgur.com/t70nByV.png">

Như ta thấy:
- User-mode: các module KVM gọi đến sử dụng ioclt() để thực thi mã khác cho đến khi hoạt động I/O khởi xướng bởi guest hoặc một sự kiện nào đó bên ngoài xảy ra. Sự kiện này có thể là sự xuất hiện của một gói tin mạng, cũng có thể là trả lời của một gói tin mạng được gửi bởi các máy chủ trước đó. Những sự kiện như vậy được biểu diễn như là tín hiệu dẫn đến sự gián đoạn của thực thi mã khách.
- Kernel-mode: Kernel làm phần cứng thực thi các mã khách tự nhiên. Nếu bộ xử lý thoát khỏi guest do cấp phát bộ nhớ hay I/O hoạt động, kernel thực hiện các nhiệm vụ cần thiết và tiếp tục luồng thực hiện. Nếu các sự kiện bên ngoài như tín hiệu hoặc I/O hoặt động khởi xướng bởi các guest tồn tại, nó thoát tới user-mode.
- Guest-mode: Đây là cấp độ phần cứng, nơi mà các lệnh mở rộng thiết lập của một CPU có khả năng ảo hóa được sử dụng để thực thi mã nguồn gốc, cho đến khi một lệnh được gọi như vậy cần sự hỗ trợ của KVM, một lỗi hoặc một gián đoạn từ bên ngoài.
- Khi một máy ảo chạy, có rất nhiều chuyển đổi giữa các chế độ. Từ kernel-mode tới guest-mode và ngược lại rất nhanh, bởi vì chỉ có mã nguồn gốc được thực hiện trên phần cứng cơ bản. Khi I/O hoạt động diễn ra các luồng thực thi tới user-mode, rất nhiều thiết bị ảo I/O được tạo ra, do rất nhiều I/O thoát và chuyển sang chế độ user-mode chờ. Hãy tưởng tượng mô phỏng một đĩa cứng và 1 guest đang đọc các block từ nó. Sau đó QEMU mô phỏng các hoạt động bằng cách giả lập các hoạt động bằng các mô phỏng hành vi của các ổ cứng và bộ điều khiển nó được kết nối. Để thực hiện các hoạt động đọc, nó đọc các khối tương ứng từ một tập tin lớp và trả về  dữ liệu cho guest. Vì vậy user-mode giả lập I/O có xu hướng xuất hiện một nút cổ chai làm chậm việc thực hiện máy ảo.

---

### 4. Tính năng
- **Security**
Trong kiến trúc KVM, máy ảo được xem như các tiến trình Linux thông thường, nhờ đó nó tận dụng được mô hình bảo mật của hệ thống Linux như SElinux, cung cấp khả năng cô lập và kiếm soát tài nguyên. Bên cạnh đó còn có SVirt project - dự án cung cấp giải pháp bảo mật MAC (Mandatory Access Control - kiểm soát truy cập bắt buộc) tích hợp với hệ thống ảo hóa sử dụng SElinux để cung cấp một cơ sở hạ tầng cho phép người quản trị định nghĩa nên các chính sách để cô lập các máy ảo. Nghĩa là SVirt sẽ đảm bảo rằng các tài nguyên của máy ảo không thể bị truy cập bởi bất kìa các tiến trình nào khác, việc này cũng có thể thay đổi bởi người quản trị hệ thống để đặt ra quyền hạn đặc biệt, nhóm các máy ảo với nhau chia sẻ chung tài nguyên.
- **Memory Management**
KVM thừa kế tính năng quản lý bộ nhớ mạnh mẽ của Linux. Vùng nhớ của máy ảo được lưu trữ trên cùng một vùng nhớ dành cho các tiến trình Linux khác và có thể là swap. KVM hỗ trợ NUMA (Non-Uniform Memory Access- bộ nhớ thiết kế cho hệ thống đa xử lý) cho phép tận dụng hiệu quả vùng nhớ kích thước lớn. KVM hỗ trợ các tính năng ảo của mới nhất từ các nhà cung cấp CPU như EPT (Extended Page Table) của Microsoft, Rapid Virtualzation Indexing (RVI) của AMD để giảm thiểu mức độ sử dụng CPU và cho thông lượng cao hơn. KVM cũng hỗ trợ tính năng Memory page sharing bằng cách sử dụng tính năng của kernel là Kernel Sam-page Mergin (KSM)
- **Storage**
KVM có khả năng sử dụng bất kì giải pháp lưu trữ nào hỗ trợ bởi Linux để lưu trữ các Images của các máy ảo, bao gồm các ổ cục bộ như IDE, SCSI và SATA, Network attached Storage (NAS) bao gồm NFS và SAMBA/CIFS hoặc SAN thông qua giao thức iSCSI và Fibre Channel. KVM tận dụng được các hệ thống lưu trữ tin cậy từ các nhà cung cấp hàng đầu trong lĩnh vừa storage. KVM cũng hỗ trợ các images của các máy ảo trên hệ thống tệp tin chia sẻ như GFS2 cho phép các images có thể được chia sẻ giữa nhiều host hoặc chia sẻ chung giữa các ổ logic.
- **Live migration**
KVM hỗ trợ live migration cung cấp khả năng di chuyển các máy ảo đang chạy giữa các host vật lý mà không làm gián đoạn dịch vụ. Khả năng live migration là trong suốt với người dùng, các máy ảo vẫn duy trì trạng thái bật, kết nối mạng vẫn đảm bảo và các ứng dụng của người dùng vẫn tiếp tục duy trì trong khi máy ảo được đưa sang một host vật lý mới. KVM cũng cho phép lưu lại trạng thái hiện tại của máy ảo để cho phép lưu trữ và khôi phục trạng thái đó vào lần sử dụng tiếp theo.
- **Performance and scalability**
KVM kế thừa hiệu năng và khả năng mở rộng của Linux, hỗ trợ máy ảo với 16 CPUs ảo, 256GB RAM và hệ thống máy host lên tới 256 cores vaf treen 1TB RÁM


---

### 5. **Các công nghệ ảo hóa network trong KVM**

Trong KVM sử dụng 2 công nghệ ảo hóa network là linux bridge và OpenVswitch , mục đích dùng để kết nối các máy ảo thông qua 1 con switch ảo được tạo ra bởi 2 công nghệ trên, nó cho phép kết nối các máy ảo với nhau, các máy ảo sẽ foward data giữa các máy ảo và giữa máy ảo với máy chủ vật lý.

- **Open vSwitch**: Hỗ trợ những tính năng sau
<ul>
  <ul>
<li>Hỗ trợ tính năng VLAN chuẩn 802.1Q với các cổng trunk và access như một switch layer thông thường.</li>
<li>Hỗ trợ giao diện NIC bonding có hoặc không có LACP trên cổng uplink switch.</li>
<li>Hỗ trợ NetFlow, sFlow(R), và mirroring để tăng khả năng hiển thị.</li>
<li>Hỗ trợ cấu hình QoS (Quality of Service) và các chính sách thêm vào khác.</li>
<li>Hỗ trợ tạo tunnel GRE, VXLAN, STT và LISP.Hố trợ tính năng quản lý các kết nối 802.1aq</li>
<li>Hỗ trợ OpenFlow các phiên bản từ 1.0 trở lên.</li>
<li>Cấu hình cơ sở dữ liệu với C và Python.</li>
    <li>Hoạt động forwarding với hiệu suất cao sử dụng module trong nhân Linux.</li>
  </ul>
  </ul>
<img src="https://i.imgur.com/5bmHAEh.jpg">

`Flow table`: có chức năng lưu chữ thông tin packet và đẩy các packet dữ liệu đến đúng cổng kết nối

`Port`: Là các cổng kết nối đến các máy ảo

`Tap` : Là các giao diện kết nối giữa máy ảo và openvswitch

`Interface` : Là nơi kết nối giữa máy thật và openvswitch

- **Linux bridge**: Hỗ trợ các tính năng
<ul>
  <ul>
    <li>STP: Spanning Tree Protocol - giao thức chống loop</li>
<li>VLAN: chia switch (do linux bridge tạo ra) thành các mạng LAN ảo, cô lập traffic giữa các VM trên các VLAN khác nhau của cùng một switch.</li>
<li>FDB (forwarding database): chuyển tiếp các gói tin theo database để nâng cao hiệu năng switch. Database lưu các địa chỉ MAC mà nó học được. Khi gói tin Ethernet đến, bridge sẽ tìm kiếm trong database có chứa MAC address không. Nếu không, nó sẽ gửi gói tin đến tất cả các cổng.</li>
  </ul>
  </ul>
<img src="https://i.imgur.com/nXMArq5.png">

`Bridge`: Tương đương với switch layer 2

`Port`: tương đương với port của switch thật

`Tap (tap interface)`: có thể hiểu là giao diện mạng để các VM kết nối với bridge cho linux bridge tạo ra

`Fd (forward data)`: chuyển tiếp dữ liệu từ máy ảo tới bridge

 ---
 
 ### 6. Các chế độ card mạng trong KVM
 
 **Trong kvm có 3 chế độ card mạng:**
 - **NAT (default)**: sử dụng card virbr để nat địa chỉ ip cho máy ảo trên kvm nằm trong thư mục /etc/libvirt/qemu/networks/default.xml (/var/lib/libvirt/network) và có thể đi internet thông qua card br. *Trong trường hợp disable card NAT trên KVM ta có thể copy file từ thư mục /etc/libvirt/qemu/networks/default.xml vào /var/lib/libvirt/network và sử dụng lệnh sau để enable lại `virsh net-start default`.Còn trong trường hợp xóa card NAT sẽ xóa mất các file khởi động cấu hình của card này ở thư mục /var/lib/libvirt/dnsmasq và sẽ không thể bật trở lại trừ khi copy lại các file cấu hình( tương tự đối với card bridge private)*.
NAT: nat dựa vào iptables nat ra đúng địa chỉ ip của host(máy ảo cài trên vmware chính là địa chỉ card ens33) xem lệnh “ iptables -L -t nat ”.
 `Modem -> Eth0  -> virbr0 (cung cấp NAT) -> virnet1 (vNIC) -> eht0(máy ảo kvm)`
 
 - **Bridge public**: Là card br kết nối gới card mạng thật để đi internet và cấp ip bằng dhcp cho máy ảo trên KVM bởi modem.
 Bridge: Đường đi internet: `modem -->card eth0(server vật lý) --> card br0 (linux bridge) --> tap interface (linux bridge switch layer 2)-->card vNIC (máy ảo)`. Địa chỉ của máy ảo sẽ được cấp bởi DHCP của modem thông qua card `eth0 --> br0 --> vNIC`. Card vNIC kết nối với tap interface
 
 - **Bridge private**: Card chỉ cho phép các máy ảo trên kvm kết nối với nhau và không thể đi internet card bridge private nắm trong thư mục /etc/libvirt/qemu/networks/default.xml (hoặc /var/lib/libvirt/network).
`virbr2 -> vnet2-> eth1 (máy ảo kvm)`, các máy ảo có thể kết nối với nhau bằng cách add chung card mạng , 1 máy ảo có thể có nhiều card mạng.

**vnet0 là một giao diện ảo được tạo bởi libvirt (VMM). Giao diện ảo này còn được gọi là giao diện tap trên linux bridge**

---

### 7.Các định dạng khi tạo máy ảo trên KVM
- Khi tạo máy ảo sẽ sinh ra 2 loại file sau:  file chứa thông số CPU, thông số ram,... và file chứa hệ điều hành,disk ảo...

- **`Khi tạo máy ảo từ file định dạng .img hệ thống sẽ sinh ra 2 file, 1 file nằm trên chính file .img chứa hệ điều hành, và 1 file .xml nằm ở thư mục /etc/libvirt/qemu/ chứa thông số ram, cpu, kiểu định dạng...`**
- **`Khi tạo máy ảo từ file định dạng .iso hệ thống sẽ sinh ra 2 file 1 file nằm ở thư mục /var/lib/libvirt/images/ chứa hệ điều hành, và file .xml ở thư mục /etc/libvirt/qemu/ chứa thông số ram, cpu, kiểu định dạng...`**

- Các định dạng file sử dụng phổ biến trong việc tạo máy ảo trên KVM: iso, image, qcow2
- KVM sử dụng file image để tạo máy ảo: Trong đó file image là file đóng gói chứa tất cả các nội dung cài đặt, ứng dụng ở trong đó. Trong KVM sử dụng file image sẽ có 3 định dạng chính là:
<ul>
  <ul>
    <li>ISO: Là một file chứa dữ liệu,các gói để xây dựng lên 1 chương trình của đĩa CD, DVD</li>
    <li>raw: Sử dụng cơ chế phân vùng lưu trữ Thick provisioning.</li>  
    <li>qcow2(QEMU Copy On Write): Sử dụng cơ chế Thin provosioning.</li>
      <li>Thin provision: Sử dụng cơ chế copy on write image tạo 1 file chứa thông tin dữ liệu mới và liên kết với file image ban đầu để đọc dữ liệu cũ, tuy sẽ tạo ra file có kích thước với dung lượng thực tế có nhưng bắt  buộc phải truy cập vào image cũ mới có thể đọc được dữ liệu.
        <ul>
          <ul>
            <li>Ưu điểm: Nó cho phép sử dụng không gian lưu trữ hiệu quả, giảm chi phí lưu trữ, cung cấp dung lượng lưu trữ nhanh</li>
            <li>Nhược điểm: Hiệu suất thấp, Việc lưu trữ quá mức có thể cho phép gián đoạn ứng dụng hoặc bị treo, không hỗ trợ tính năng cluster</li>
  </ul>
  </ul>
  
  <li>Thick Provisioned : Tạo đĩa ảo (VM) theo định dạng này mặc định nó sẽ dự trữ không gian đĩa khi VM được tạo, nhưng các khối chứa dữ liệu cũ trên thiết bị lưu trữ chỉ bị xóa khi máy ảo ghi dữ liệu mới vào đĩa lần đầu tiên. Với dự phòng dày, dung lượng lưu trữ lớn được cung cấp trước nhu cầu lưu trữ trong tương lai. Tuy nhiên, không gian trống không sử dụng gây lãng phí dung lượng lưu trữ.
  <ul>
    <ul>
      <li>Ưu điểm: Hiệu suất sử dụng cao </li>
      <li>Nhược điểm: Lãng phí dung lượng lưu trữ, việc cung cấp dung lượng lưu trữ kém hơn Thin Provisioned,  không hỗ trợ tính năng  snapshot, cluster</li>
    </ul>
  </ul>
  
<img src="https://i.imgur.com/sqSx3Yo.png">
  
 Về hiệu suất đọc ghi file raw sẽ cao hơn file qcow2.
 <img src="https://i.imgur.com/xM3TKQQ.png">
 
 Dung lượng file tạo với định dạng qcow2 sẽ chiếm đúng bằng dung lượng thực tế data có còn với file định dạng raw nó sẽ chiếm cả 1 khoảng dung lượng mà được tạo tuy dung lượng thực tế không được như vậy
 
 <img src="https://i.imgur.com/1tjsqn8.png">
 
  </ul>
  </ul>
 
----

### 8. Kỹ thuật backup trong KVM
#### 1.Clone
- Để clone được máy ảo ta cần shutdown máy ảo trước, sau đó click chuột phải vào máy ảo chọn clone
    
<img src="https://i.imgur.com/rzI6m4W.png">

- Tiếp theo chọn clone để hoàn thành

<img src="https://i.imgur.com/5XpxHhX.png">

#### 2.snapshot
**1. Snapshot trên virt-manager(internal)**
- chọn mục snapshot 
<img src="https://i.imgur.com/YYto7hN.png">

- Tiếp theo click vào biểu tượng create 
<img src="https://i.imgur.com/KwHdNRl.png">

- Sau đó click finish để hoàn thành 
<img src="https://i.imgur.com/a2Nr2Q7.png">

- Khi cần backup lại dữ liệu từ snapshot ta chọn biểu tượng run để đưa máy ảo về trạng thái khi snapshot
<img src="https://i.imgur.com/1kKYWap.png">

**2.Snapshot bằng command line(external)**
- `qemu-img info /var/lib/libvirt/images/file1` kiểm tra định dạng file chứa hệ điều hành của máy ảo
- Tiến hành kiểm tra ổ đĩa mà máy ảo muốn tạo snapshot đang sử dụng bằng câu lệnh `virsh domblklist generic --details`
<img src="https://i.imgur.com/SHmKTbH.png">

- Tiến hành tạo snapshot và kiểm tra snapshot đã tạo `virsh snapshot-create-as generic snapshot "My First Snapshot" --disk-only --atomic` , kiểm tra file snapshot `virsh snapshot-list generic` (generic là tên máy ảo, snapshot là tên của file snapshot sẽ được đặt)

<img src="https://i.imgur.com/K3BxGGK.png">

- Snapshot đã được tạo tuy nhiên nó chỉ lưu trữ duy nhất trạng thái ổ đĩa, kiểm tra bằng lệnh `virsh snapshot-info generic snapshot`
- Kiểm tra nơi chứa file snapshot `virsh domblklist generic`
- Kiểm tra trạng thái của ổ đĩa ban đầu qemu-img info linux-microcore-6.4.img.snapshot2
- Lúc này ổ đĩa cũ đã biến thành trạng thái read-only, VM dùng ổ đĩa mới để lưu dữ liệu và backingfile sẽ là ổ đĩa ban đầu. 
<img src="https://i.imgur.com/7QT6uAM.png">




